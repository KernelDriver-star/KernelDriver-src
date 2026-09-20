---
title: GPU coreDump 实现原理：从 CUDA 核心转储到 ELF 与 Xid 的全链路拆解
date: 2026-09-20 10:00:00
categories:
  - GPU
tags:
  - CUDA
  - GPU
  - coreDump
  - 调试
  - ELF
---

GPU 程序跑飞了怎么定位？CPU 上靠 `ulimit -c unlimited` + `gdb` 就能拿到崩溃时的寄存器和栈，GPU 上却没这么简单——GPU 没有独立的 OS、没有 `/proc/<pid>/maps`，整张卡的现场全靠驱动配合 runtime 把显存和寄存器拍照保存下来。这篇把 GPU coreDump 的实现原理讲清楚：触发条件、生成路径、文件格式、加载调试，以及 NVIDIA 与 AMD 两种生态的差异（内容以 [CUDA-GDB 手册](https://docs.nvidia.com/cuda/cuda-gdb/) 和 [CUDA Driver API Coredump Attributes](https://docs.nvidia.com/cuda/cuda-driver-api/group__CUDA__COREDUMP.html) 为准）。

## 一、先回答：GPU coreDump 到底是什么

GPU coreDump（GPU 核心转储）是 GPU 在异常或主动触发时，由 GPU 驱动 + runtime 协作把当前 GPU 上下文快照到一个文件里的产物。它和 CPU 的 `core` 文件目的相同——事后调试（post-mortem debugging），但内容结构完全不同：

| 维度 | CPU core dump | GPU coreDump |
|---|---|---|
| 生成方 | Linux 内核（`coredump` 流程） | GPU 驱动 + runtime（如 CUDA driver / ROCr） |
| 文件内容 | 进程虚拟内存映射 + 通用寄存器 | GPU 设备内存 + warp/线程上下文 + 加载的 cubin/ELF 镜像 |
| 触发信号 | `SIGSEGV`、`SIGABRT` 等 | GPU 异常（Xid 13/31/43 等）或用户主动触发 |
| 加载方式 | `gdb ./a.out core` | `cuda-gdb` / `rocgdb` 配合 `target cudacore` |
| 默认路径 | `core` 或 `core.PID` | `core.cuda.HOSTNAME.PID`（CUDA 默认） |

理解 GPU coreDump 的关键，是理解它是一份**由 GPU 驱动在内核态+用户态联合构造的、基于 ELF 容器扩展的**结构化快照，而不是简单的"内存 memcpy"。

## 二、CUDA coreDump 的整体架构

NVIDIA 是目前把 GPU coreDump 做得最完整的厂商。整个机制在 CUDA 7.0 引入，到 CUDA 12 系列已经形成"环境变量 + Driver API + 回调 + 轻量级模式"四位一体的体系。

### 2.1 触发路径

GPU coreDump 不是凭空生成的，需要先有"触发点"。CUDA 提供四类触发方式：

```text
┌──────────────────────────────────────────────────────────┐
│   1. 异常触发（被动）                                       │
│      CUDA_ENABLE_COREDUMP_ON_EXCEPTION=1                  │
│      → GPU 异常（如非法访问、非法指令）→ 驱动捕获 → 写文件    │
├──────────────────────────────────────────────────────────┤
│   2. 用户主动触发                                           │
│      CU_COREDUMP_ENABLE_USER_TRIGGER + cuCoredumpGenerate │
│      → 应用代码主动调用，类似 CPU 上 gcore                   │
├──────────────────────────────────────────────────────────┤
│   3. Host 端触发                                           │
│      CU_COREDUMP_TRIGGER_HOST                              │
│      → CPU 端崩溃时连带生成 GPU 端转储                       │
├──────────────────────────────────────────────────────────┤
│   4. 命名管道（流式）                                       │
│      CU_COREDUMP_PIPE / CUDA_COREDUMP_PIPE                 │
│      → 不落地为文件，直接写到 FIFO 给消费端处理              │
└──────────────────────────────────────────────────────────┘
```

第 4 种是 CUDA 12.3 新增的能力，让外挂的采集进程可以流式接管 coreDump 数据，常用于大规模集群中的故障采集代理。

### 2.2 配置矩阵：环境变量 vs Driver API

CUDA 给了两种配置入口，本质是同一套属性集合的两种写入方式：

**（1）环境变量方式（最常用）**

| 环境变量 | 作用 | 默认值 |
|---|---|---|
| `CUDA_ENABLE_COREDUMP_ON_EXCEPTION` | 异常时是否生成 GPU coreDump | `0` |
| `CUDA_ENABLE_CPU_COREDUMP_ON_EXCEPTION` | 同时生成 CPU core dump | `0` |
| `CUDA_ENABLE_LIGHTWEIGHT_COREDUMP` | 轻量级（跳过大块内存） | `0` |
| `CUDA_COREDUMP_FILE` | 自定义文件名 | `core.cuda.HOSTNAME.PID` |
| `CUDA_COREDUMP_PIPE` | 命名管道路径（流式） | 未设置 |

**（2）Driver API 方式（CUDA 12.1+ 引入）**

为了让应用程序能动态控制 coreDump 行为（而不是依赖部署时设环境变量），CUDA 12.1 在 Driver API 里加了一组 `cuCoredump*` 函数，对应的 `CUcoredumpSettings` 枚举：

```c
typedef enum CUcoredumpSettings {
    CU_COREDUMP_ENABLE_ON_EXCEPTION = 1,
    CU_COREDUMP_TRIGGER_HOST,
    CU_COREDUMP_LIGHTWEIGHT,
    CU_COREDUMP_ENABLE_USER_TRIGGER,
    CU_COREDUMP_FILE,           // 字符串，最长 1023 字节
    CU_COREDUMP_PIPE,           // 字符串，最长 1023 字节
    CU_COREDUMP_GENERATION_FLAGS,
    CU_COREDUMP_MAX
} CUcoredumpSettings;
```

读取和设置的核心 API：

```c
CUresult cuCoredumpGetAttribute(CUcoredumpSettings attr,
                                 void* value, size_t* size);          // 当前 context
CUresult cuCoredumpGetAttributeGlobal(CUcoredumpSettings attr,
                                       void* value, size_t* size);    // 全应用
CUresult cuCoredumpSetAttribute(CUcoredumpSettings attr,
                                 void* value, size_t* size);          // 当前 context
CUresult cuCoredumpSetAttributeGlobal(CUcoredumpSettings attr,
                                       void* value, size_t* size);    // 全应用
```

注意区分两个作用域：

- **per-context**：仅对当前 CUDA context 生效，多个 GPU 各自独立配置；
- **application-global**：对整个进程所有 context 生效，通常用于初始化阶段一次性配置。

### 2.3 生成内容控制：CUCoredumpGenerationFlags

完整 coreDump 可能包含数 GB 的设备内存，生产环境很难接受。`CU_COREDUMP_GENERATION_FLAGS` 用位掩码控制要跳过哪些段：

```c
typedef enum CUCoredumpGenerationFlags {
    CU_COREDUMP_DEFAULT_FLAGS                  = 0,
    CU_COREDUMP_SKIP_NONRELOCATED_ELF_IMAGES   = (1 << 0),
    CU_COREDUMP_SKIP_GLOBAL_MEMORY             = (1 << 1),
    CU_COREDUMP_SKIP_SHARED_MEMORY              = (1 << 2),
    CU_COREDUMP_SKIP_LOCAL_MEMORY               = (1 << 3),
    CU_COREDUMP_SKIP_ABORT                      = (1 << 4),
    CU_COREDUMP_SKIP_CONSTBANK_MEMORY           = (1 << 5),
    CU_COREDUMP_GZIP_COMPRESS                   = (1 << 6),

    // 轻量级等价组合
    CU_COREDUMP_LIGHTWEIGHT_FLAGS =
          CU_COREDUMP_SKIP_NONRELOCATED_ELF_IMAGES
        | CU_COREDUMP_SKIP_GLOBAL_MEMORY
        | CU_COREDUMP_SKIP_SHARED_MEMORY
        | CU_COREDUMP_SKIP_LOCAL_MEMORY
        | CU_COREDUMP_SKIP_CONSTBANK_MEMORY,
} CUCoredumpGenerationFlags;
```

`CU_COREDUMP_LIGHTWEIGHT` 环境变量等价于 `CU_COREDUMP_LIGHTWEIGHT_FLAGS`——只保留寄存器、warp 调度状态、异常信息等"事故现场"，跳过大块数据。它的体积通常能压到完整转储的 1/100 以下，是线上排查的首选。

`CU_COREDUMP_GZIP_COMPRESS`（位 6）会在生成时直接 gzip 压缩，进一步减小体积，适合跨节点归档。

### 2.4 回调机制：让外挂程序感知

CUDA 12.1+ 还提供了两个回调注册点，让你能在 coreDump 开始/结束时被通知：

```c
typedef void* CUcoredumpCallbackHandle;

typedef void (CUDA_CB *CUcoredumpStatusCallback)(
    void*      userData,
    int        pid,
    CUdevice   dev);

CUresult cuCoredumpRegisterStartCallback(
    CUcoredumpStatusCallback  callback,
    void*                     userData,
    CUcoredumpCallbackHandle* callbackOut);

CUresult cuCoredumpRegisterCompleteCallback(
    CUcoredumpStatusCallback  callback,
    void*                     userData,
    CUcoredumpCallbackHandle* callbackOut);

CUresult cuCoredumpDeregisterStartCallback(CUcoredumpCallbackHandle);
CUresult cuCoredumpDeregisterCompleteCallback(CUcoredumpCallbackHandle);
```

典型用途：

- **Start 回调**：coreDump 即将开始，可以记录"此刻在跑哪个 kernel、用户参数是什么"；
- **Complete 回调**：coreDump 落地完成，可以触发上传到对象存储、通知运维系统、释放相关资源。

回调拿到的是 `pid` 和 `CUdevice`，刚好对应"哪个进程的哪张卡出了事"。

## 三、coreDump 文件格式：为什么是 ELF

### 3.1 ELF 选择的合理性

NVIDIA 把 GPU coreDump 的容器格式选成了 **ELF64**，而不是自定义格式或 Linux 原生 core 格式。原因有三：

1. **cubin 本身就是 ELF**：CUDA 编译器（`nvcc`/`ptxas`）产出的 GPU 可执行码（cubin）就是标准 ELF64 文件，`e_machine = EM_CUDA (0xBE = 190)`、`EI_OSABI = 0x41 (CUDA ABI)`。把 coreDump 也做成 ELF，可以用同一套工具链解析。
2. **ELF 的 section 机制天然适合装"多段异构数据"**：寄存器、warp 状态、设备内存、cubin 镜像、DWARF 调试信息，每类放进一个独立 section，互不干扰。
3. **复用成熟生态**：`readelf`、`objdump`、`binutils`、GDB 自带的 ELF 解析器都能直接读 coreDump 容器，不用造新轮子。

### 3.2 cubin ELF 的关键字段

理解 GPU coreDump 的内部结构，先得理解它要"装"的 cubin 长什么样。一个单 kernel 的 cubin 大致包含这些 section（参见 [Anatomy of a CUDA Binary](https://hiraditya.github.io/posts/anatomy-of-a-cuda-binary/)）：

| Section | 类型 | 作用 |
|---|---|---|
| `.text.<kernel>` | `SHT_PROGBITS` | SASS 机器码（每条指令 128-bit/16 字节） |
| `.nv.constant0.<kernel>` | `SHT_PROGBITS` | 常量 bank 0，kernel 参数通过它传递 |
| `.nv.info` | `SHT_LOPROC` | 模块级 metadata（EIATTR 流） |
| `.nv.info.<kernel>` | `SHT_LOPROC` | 每个 kernel 的 metadata |
| `.note.nv.cuver` | `SHT_NOTE` | CUDA 版本信息 |
| `.note.nv.tkinfo` | `SHT_NOTE` | Toolkit 版本信息 |
| `.nv.compat` | 自定义 | SM 兼容性描述 |
| `.nv.callgraph` | 自定义 | 模块内调用图 |
| `.symtab` / `.strtab` / `.shstrtab` | 标准 ELF | 符号表/字符串表 |

ELF header 里几个关键字段：

```text
e_ident[EI_OSABI]    = 0x41   // CUDA ABI（老版本是 0x33）
e_ident[EI_ABIVERSION] = 8
e_type               = ET_EXEC
e_machine            = 0xBE  // EM_CUDA = 190
e_flags              = 0x06006402  // bits 8-15 = SM 架构（0x64 = 100 = sm_100）
                                   // bit 1 = 64-bit addressing
                                   // bits 24-26 = 格式版本
```

`e_flags` 高位编码 SM 架构，这是驱动加载 cubin 时用来校验"这个 cubin 是给哪一代 GPU 编的"的关键。CUDA 13 的驱动甚至会拒绝旧版 `ELFOSABI_CUDA = 0x33` 的 cubin，`cuModuleGetFunction` 直接返回 `CUDA_ERROR_NOT_SUPPORTED`。

### 3.3 coreDump ELF 额外塞了什么

GPU coreDump 复用 cubin 的 ELF 容器，但额外塞进了"事故现场"的 section：

- **设备内存 dump**：global memory、shared memory、local memory、constant bank memory 的内容（除非被 `SKIP_*` flag 跳过）；
- **warp/线程上下文**：当前活跃 warp 的寄存器值、调度状态、PC；
- **异常信息**：触发的异常类型、触发的 warp、触发的指令地址（参见 CUDA 12.3 修复的 SIGTRAP 显示问题）；
- **加载的 cubin 镜像**：异常时已加载到 GPU 上的 cubin 副本（除非 `SKIP_NONRELOCATED_ELF_IMAGES`）；
- **DWARF 调试信息**：如果 kernel 用 `-G` 编译，调试信息会嵌入，供 `cuda-gdb` 做源码级回溯。

加载到调试器时，`cuda-gdb` 用专门的 target：

```text
(cuda-gdb) target cudacore core.cuda.localhost.1234
```

这条命令告诉 GDB 后端"这不是普通 core 文件，是 CUDA core dump"，然后会按 ELF 解析出 GPU 段、按 DWARF 还原源码位置、按 cubin 符号表把寄存器映射到变量。

## 四、AMD GPU coreDump 的不同走法

AMD 在 ROCm 体系里也实现了 GPU coreDump，但架构和 NVIDIA 有明显差异。

### 4.1 谁来生成

NVIDIA 是 GPU 驱动 + CUDA runtime 一起干，AMD 则主要交给 **ROCr Runtime**（用户态 runtime，对应 `libhsa-runtime`）在 GPU 异常时生成。这带来一个差异：

> **重要约束**：如果 host 端先发生了 fault 并触发了 Linux 内核的 core dump，AMD ROCm runtime 就**不会**再生成 AMD GPU core 文件——整个进程只拿到 host 侧的 core。要让 GPU 端被捕获，必须保证进程没被内核提前终结。

### 4.2 加载方式：ROCgdb

AMD 的 GPU 调试器叫 [ROCgdb](https://rocm.docs.amd.com/projects/ROCgdb/)（GDB 的 AMD GPU 扩展）。加载 core dump 的方式和 GDB 标准用法一致：

```bash
rocgdb ./a.out core
# 或在 GDB 内
(gdb) core-file core
```

AMD 的 cubin 也是 ELF，但 `e_machine` 是 `EM_AMDGPU (0xE0 = 224)`，section 命名规则和 NVIDIA 不同（没有 `.nv.*` 前缀，而是用 LLVM AMDGPU Backend 的约定）。

### 4.3 架构限制

不是所有 AMD GPU 都支持 coreDump。当前 [ROCgdb 文档](https://rocm.docs.amd.com/projects/ROCgdb/) 明确说明：

- `gfx1100` / `gfx1101` / `gfx1102`（RDNA3 消费卡）**不能**生成 AMD GPU core dump；
- 数据中心卡（MI100/MI200/MI300 系列，对应 `gfx908/gfx90a/gfx942` 等）支持最完整；
- 新版本 ROCr 修复了一个问题：即使有调试器 attached，runtime 也能从自己的内部状态捕获触发异常并生成合法 core dump（以前 attached 时不行）。

选择生态时要留意：消费级 AMD 显卡的 GPU coreDump 能力是残缺的。

## 五、驱动层与 Xid：coreDump 的上游

GPU coreDump 不会凭空触发，背后是 GPU 驱动对硬件异常的捕获。NVIDIA 把这些异常用 **Xid** 编码报告给系统。

### 5.1 Xid 是什么

[Xid Errors](https://docs.nvidia.com/deploy/xid-errors/) 是 NVIDIA 驱动向 OS 内核日志报告 GPU 错误的标准机制，格式形如：

```text
NVRM: Xid (0000:03:00): 13, Channel 00000001
       │           │       │
       │           │       └── Xid 编号（决定错误类型）
       │           └── PCI BDF（定位是哪张卡）
       └── 驱动标识
```

Linux 下查看：

```bash
sudo dmesg | grep -i NVRM         # 所有 NVRM 消息
sudo dmesg | grep -i "Xid"        # 仅 Xid
```

每个 Xid 编号对应一类错误，含义跨驱动版本稳定。Xid 的位置在 `/var/log/messages` 或 `/var/log/syslog`（视发行版而定），systemd 系统主要看 `journalctl -k`。

### 5.2 哪些 Xid 会触发 coreDump

不是所有 Xid 都对应"可调试的应用异常"。和 coreDump 关系最密切的几类：

| Xid | 含义 | 是否触发 coreDump |
|---|---|---|
| 13 | Graphics Engine Exception（图形引擎异常，多为应用非法访问/非法指令） | **是**，最典型的 coreDump 触发源 |
| 31 | GPU memory page fault（GPU 内存页错误） | **是**，常对应越界访问 |
| 8 | GPU stopped processing（FIFO idle timeout） | 可能，视配置 |
| 43 | ForceRecovery（驱动强制重置 GPU） | 否，但会终止上下文 |
| 48 | ECC error（双位 ECC 不可纠正） | 否，硬件错误 |
| 79 | GPU has fallen off the bus | 否，硬件掉线 |
| 119 | GSP RPC timeout | 否，驱动自身问题 |
| 154 | GPU Recovery Action | 否，但会清理现场 |

官方推荐排查思路：

> 应用程序问题（Xid 13/31）→ 跑 `cuda-gdb` 或 Compute Sanitizer memcheck；
> 硬件问题（Xid 48/79）→ 联系硬件厂商跑诊断；
> 驱动问题（Xid 119 等）→ 用 `nvidia-bug-report.sh` 收集日志报 NVIDIA。

### 5.3 故障排查链路

把 Xid、coreDump、调试器串起来，一条完整的 GPU 事故排查链路是这样的：

```text
   GPU 硬件异常
        │
        ▼
   驱动捕获异常 → 写入 Xid 到内核日志
        │
        ▼
   用户态 runtime 检测到 context 异常
        │
        ├─ 若 CUDA_ENABLE_COREDUMP_ON_EXCEPTION=1
        │     │
        │     ▼
        │  驱动+runtime 生成 core.cuda.HOSTNAME.PID（ELF 容器）
        │     │
        │     ├─ 调用 Start 回调（如果有）
        │     ├─ 收集设备内存/warp 状态/cubin 镜像
        │     ├─ 可选 gzip 压缩
        │     └─ 调用 Complete 回调
        │
        ▼
   事后调试：
   $ cuda-gdb ./myapp
   (cuda-gdb) target cudacore core.cuda.myhost.12345
   (cuda-gdb) bt            # 栈回溯
   (cuda-gdb) info cuda threads
   (cuda-gdb) print some_var
```

如果跑的是大规模集群，常见做法是：

1. 全局打开 `CUDA_ENABLE_LIGHTWEIGHT_COREDUMP=1` + `CUDA_COREDUMP_FILE=/var/log/gpudump/core.%h.%p`；
2. 监控 `dmesg` 里的 Xid，一旦出现 13/31，立即捞取最近一次 coreDump；
3. Complete 回调里把 coreDump 上传到对象存储，附带 Xid 编号和 GPU GUID。

## 六、实战：从触发到加载

### 6.1 环境变量方式（最简）

```bash
# 开异常触发 + 轻量级 + 自定义路径
export CUDA_ENABLE_COREDUMP_ON_EXCEPTION=1
export CUDA_ENABLE_LIGHTWEIGHT_COREDUMP=1
export CUDA_COREDUMP_FILE=/var/log/gpudump/core.%h.%p

# 跑一个会越界的 kernel
./bad_kernel
# ls /var/log/gpudump/
# core.myhost.12345
```

加载：

```bash
cuda-gdb ./bad_kernel
(cuda-gdb) target cudacore /var/log/gpudump/core.myhost.12345
(cuda-gdb) bt
(cuda-gdb) info cuda kernels
(cuda-gdb) info cuda threads
(cuda-gdb) print *(int*)devPtr
```

### 6.2 Driver API 方式（应用内控）

如果应用需要在特定条件下主动触发 coreDump（而不是等异常），用 Driver API 更可控：

```c
#include <cuda.h>
#include <stdio.h>

void on_coredump_complete(void* userData, int pid, CUdevice dev) {
    fprintf(stderr, "[coredump] pid=%d dev=%d done\n", pid, dev);
    /* 在这里上传到对象存储 / 通知监控 */
}

void enable_coredump(void) {
    int enable = 1;
    cuCoredumpSetAttributeGlobal(CU_COREDUMP_ENABLE_ON_EXCEPTION, &enable, sizeof(enable));
    cuCoredumpSetAttributeGlobal(CU_COREDUMP_ENABLE_USER_TRIGGER,  &enable, sizeof(enable));

    /* 轻量级 + 压缩 */
    unsigned flags = CU_COREDUMP_LIGHTWEIGHT_FLAGS | CU_COREDUMP_GZIP_COMPRESS;
    cuCoredumpSetAttributeGlobal(CU_COREDUMP_GENERATION_FLAGS, &flags, sizeof(flags));

    /* 注册完成回调 */
    CUcoredumpCallbackHandle h;
    cuCoredumpRegisterCompleteCallback(on_coredump_complete, NULL, &h);
}

void dump_now(void) {
    /* 主动触发当前 context 的 coreDump */
    cuCoredumpGenerate();
}
```

> 注意：`cuCoredumpGenerate` 是 user-trigger 模式的入口，必须在已经 enable `CU_COREDUMP_ENABLE_USER_TRIGGER` 之后调用，且当前线程必须有 active CUDA context。

### 6.3 用命名管道做流式采集（CUDA 12.3+）

```bash
# 1. 先建 FIFO
mkfifo /var/run/gpu_coredump.fifo

# 2. 让采集进程持续消费
while true; do
    cat /var/run/gpu_coredump.fifo >> /storage/gpu_dumps/$(date +%s).core.gz
done &

# 3. 应用端指向 FIFO
export CUDA_COREDUMP_PIPE=/var/run/gpu_coredump.fifo
export CUDA_ENABLE_COREDUMP_ON_EXCEPTION=1
./myapp
```

coreDump 数据会以流的方式写到 FIFO，被采集进程接走，不会在本地落盘。这对容器化部署非常友好——容器内没多少磁盘，coreDump 直接流到宿主机的归档进程。

## 七、最佳实践与坑点

**（1）开轻量级而不是完整**

线上别开完整 coreDump。一个 H100 有 80GB HBM，一个 coreDump 可能就 80GB，加上多卡场景直接打爆磁盘。`CUDA_ENABLE_LIGHTWEIGHT_COREDUMP=1` 是默认推荐，遇到疑难问题再单次开完整。

**（2）`-lineinfo` 和 `-G` 的区别要分清**

- `-lineinfo`：只保留行号映射，体积小，性能影响小，足够做栈回溯；
- `-G`：完整调试信息，能查变量值，但性能下降明显，且让 cubin 显著变大。

线上构建用 `-lineinfo`，本地复现用 `-G`。`cuda-gdb` 加载 coreDump 时，如果原始 cubin 没有任何调试信息，栈回溯只会显示地址，不会映射到源码行号。

**（3）AMD 平台注意 host 先挂的情况**

ROCm 的 GPU coreDump 强依赖 runtime 自己捕获异常。如果 host 端先 `SIGSEGV`，Linux 内核直接走 `core` 流程把进程终结，runtime 没机会生成 GPU core。规避办法是把容易挂的 host 代码用 `setjmp`/`longjmp` 或信号处理拦截，给 runtime 留出时间。

**（4）coreDump 文件权限**

CUDA coreDump 默认写到进程当前目录，权限是运行用户的 `umask` 决定。如果用 systemd 跑服务，注意 `WorkingDirectory=` 和 `ReadWritePaths=` 必须包含目标路径，否则会看到 coreDump 生成失败但 GPU 异常已经吃掉了现场。

**（5）MPS 场景的传染性**

在 Volta-MPS 下，一个 client 触发 coreDump，**所有其他 client 都会受影响**——因为它们共享同一个 GPU context。生产环境用 MPS 时，coreDump 策略要在 client 隔离层制定，而不是简单全局打开。

**（6）Xid 13 不一定是应用 bug**

Xid 13 官方定义虽然是 "Graphics Engine Exception"，但文档同时提到"罕见情况下，硬件故障或系统软件 bug 也会表现为 Xid 13"。所以看到 Xid 13 + coreDump 后，如果 `cuda-gdb` 加载发现现场是合法代码，要怀疑硬件/驱动问题，直接用 `nvidia-bug-report.sh` 收集日志报 NVIDIA，而不是继续啃应用代码。

## 八、小结

GPU coreDump 的实现原理可以浓缩成一句话：

> GPU 驱动 + runtime 在异常或主动触发时，把 GPU 的设备内存、warp 上下文、已加载的 cubin 镜像按 ELF 容器格式快照成文件，供 `cuda-gdb` / `rocgdb` 做源码级事后调试。

几个关键设计选择值得记住：

- **ELF 容器**：让 GPU coreDump 和 cubin 共用一套工具链，`readelf`/`objdump` 都能读；
- **分层触发**：异常被动 + 用户主动 + host 联动 + 管道流式，覆盖从开发到生产的不同需求；
- **轻量级 + 压缩**：`CU_COREDUMP_LIGHTWEIGHT_FLAGS | CU_COREDUMP_GZIP_COMPRESS` 是生产环境的默认组合；
- **回调机制**：Start/Complete 回调让 coreDump 能接入企业的可观测体系；
- **Xid 是上游**：coreDump 是事故现场，Xid 是事故编码，两者配套使用才能完整还原事故。

把 GPU coreDump 用好，等于把 GPU 程序的"事后可调试性"提升一个量级。下一篇会讲 `cuda-gdb` 在加载 coreDump 后的具体调试命令组合，把这套机制用起来。

## 参考文档

- [CUDA-GDB User Manual](https://docs.nvidia.com/cuda/cuda-gdb/) — GPU core dump 支持与 `target cudacore` 用法
- [CUDA Driver API: Coredump Attributes Control API](https://docs.nvidia.com/cuda/cuda-driver-api/group__CUDA__COREDUMP.html) — `cuCoredump*` 系列函数与 `CUcoredumpSettings` 枚举
- [Xid Errors](https://docs.nvidia.com/deploy/xid-errors/) — Xid 编号含义与排查指南
- [ROCgdb Documentation](https://rocm.docs.amd.com/projects/ROCgdb/) — AMD GPU core dump 支持与限制
- [Anatomy of a CUDA Binary](https://hiraditya.github.io/posts/anatomy-of-a-cuda-binary/) — cubin ELF section 布局分析
- [Decoding CUDA Binary File Format](https://zenodo.org/record/2339027) — cubin ELF 字段逆向
