---
title: CUDA-GDB 使用指南：GPU kernel Function 调试从入门到实践
date: 2026-09-16 10:00:00
categories:
  - CUDA
tags:
  - CUDA
  - GDB
  - GPU调试
  - 开发工具
---

写 CUDA 代码最痛苦的莫过于——CPU 端跑得好好的，一到 GPU 上就段错误、结果不对或者干脆卡死，`printf` 大法既慢又难定位。`cuda-gdb` 是 NVIDIA 官方提供的 GPU 源码级调试器，基于 GDB 扩展，能让你像调 CPU 程序一样在 GPU 内核里打断点、单步、查看变量。这篇整理一下实际开发中常用的流程和命令（命令语法以 NVIDIA 官方 CUDA-GDB 手册为准）。

## 一、环境准备

### 1.1 前置条件

- 安装 CUDA Toolkit（自带 `cuda-gdb`，不需要单独装）
- GPU 驱动版本 ≥ CUDA 版本对应要求
- Linux 环境（cuda-gdb 对 Linux/QNX 支持最完整，Windows 下功能有限）
- 编译时加 `-G` 选项生成调试信息（关键！）

### 1.2 验证安装

```bash
cuda-gdb --version
# CUDA GDB release x.y.z
```

```bash
nvidia-smi
# 确认 GPU 可用、驱动版本匹配
```

### 1.3 编译带调试信息的 CUDA 代码

```bash
nvcc -G -g -O0 -o myapp myapp.cu
```

| 选项 | 作用 |
|---|---|
| `-G` | 生成 GPU 端调试信息（device code debugging），必须加 |
| `-g` | 生成 CPU 端调试信息（host code debugging） |
| `-O0` | 关闭优化，避免变量被优化掉导致打印不出值 |

> **坑点**：`-G` 和 `-lineinfo` 不是一回事。`-G` 生成完整调试信息（可断点、可查值），`-lineinfo` 只保留行号映射（用于栈回溯，不能查变量值）。日常开发建议用 `-g -lineinfo`（对性能影响小），需要交互式调试时再用 `-g -G` 单独编译。

## 二、基础调试流程

### 2.1 启动

```bash
cuda-gdb ./myapp
# 如果程序需要命令行参数：
cuda-gdb --args ./myapp arg1 arg2
```

进入后和普通 GDB 一样打断点：

```bash
(cuda-gdb) break main          # CPU main 函数断点
(cuda-gdb) break my_kernel     # GPU 内核函数断点（直接用内核名即可，每次启动该内核都会命中）
(cuda-gdb) run
```

### 2.2 进入内核

命中断点时，cuda-gdb 会打印一条焦点切换信息，告诉你当前停在哪个 block、哪个线程：

```text
[Switching focus to CUDA kernel 0, grid 1, block (0,0,0), thread (0,0,0), device 0, sm 0, warp 0, lane 0]
```

此时你处于 GPU 调试上下文，可以直接查看内置变量：

```bash
(cuda-gdb) print threadIdx
$1 = {x = 0, y = 0, z = 0}
(cuda-gdb) print blockIdx
$2 = {x = 0, y = 0, z = 0}
(cuda-gdb) print blockDim
$3 = {x = 32, y = 1, z = 1}
```

### 2.3 单步执行

```bash
(cuda-gdb) next          # 单步步过（不进入函数调用）
(cuda-gdb) step          # 单步步入
(cuda-gdb) finish        # 跑到当前函数返回
(cuda-gdb) continue      # 继续执行到下一个断点
```

## 三、GPU 特有命令：焦点（focus）

GPU 上同时跑着成千上万个线程，但任何时刻 `print`、`next` 这些命令只作用于**一个线程**——也就是"当前焦点"。cuda-gdb 的核心扩展就是一组 `cuda` 前缀命令，用来查看和切换焦点。

焦点有两套坐标：

- **软件坐标**：kernel（内核）→ block（块）→ thread（线程）
- **硬件坐标**：device（设备）→ sm（流多处理器）→ warp（线程束）→ lane（通道）

两套坐标可以混用，cuda-gdb 会自动保持一致。

### 3.1 查看当前焦点

不带参数执行就是"查询"：

```bash
(cuda-gdb) cuda kernel block thread     # 一次看全部软件坐标
kernel 0, block (0,0,0), thread (0,0,0)

(cuda-gdb) cuda device sm warp lane     # 一次看全部硬件坐标
device 0, sm 0, warp 0, lane 0

# 也可以单独查某一级
(cuda-gdb) cuda thread
thread (0,0,0)
(cuda-gdb) cuda block
block (0,0,0)
```

### 3.2 切换焦点

带上参数就是"切换"：

```bash
# 用线程在线程块内的线性编号切换（等价于 thread (170,0,0)）
(cuda-gdb) cuda thread 170
[Switching focus to CUDA kernel 0, grid 1, block (0,0,0), thread (170,0,0), device 0, sm 0, warp 5, lane 10]

# 用 (x,y,z) 三维坐标切换
(cuda-gdb) cuda thread (16,0,0)
(cuda-gdb) cuda block (0,0,2)

# 也可以按硬件坐标切换，还能一次写多级
(cuda-gdb) cuda device 0 sm 1 warp 2 lane 0

# 多个内核在跑时，先切内核
(cuda-gdb) cuda kernel 0
```

> 这和 GDB 管 CPU 线程的方式完全对称：CPU 侧是 `info threads` + `thread N`，GPU 侧就是 `info cuda threads` + `cuda thread N`。

### 3.3 查看活动的内核、块和线程

用 `info cuda ...` 系列命令枚举：

```bash
(cuda-gdb) info cuda kernels     # 所有正在运行的内核（含 grid/block 维度）
(cuda-gdb) info cuda blocks      # 当前内核的所有活跃 block
(cuda-gdb) info cuda threads     # 当前 block 的所有线程（含各自停在哪一行）
(cuda-gdb) info cuda warps       # 活跃 warp
(cuda-gdb) info cuda lanes       # warp 内各 lane 的活跃/发散掩码
(cuda-gdb) info cuda sms         # 各 SM 上的 warp 分布
(cuda-gdb) info cuda devices     # GPU 设备信息
```

`info cuda threads` 输出里 `Filename Line` 一列能直接看到每个线程当前停在哪一行，排查"为什么大家走的路不一样"时特别有用。

### 3.4 批量查看一个 warp 内各线程的变量

`print` 只能打印当前焦点线程的值。想看整个 warp（32 个 lane）的同一个变量，可以用 GDB 的 `while` 脚本循环切换焦点：

```bash
(cuda-gdb) set $i = 0
(cuda-gdb) while $i < 32
 >cuda thread $i
 >printf "tid=%d val=%d\n", $i, data[$i]
 >set $i = $i + 1
 >end
```

如果只看一两个线程，手动切就行：

```bash
(cuda-gdb) cuda thread 7
(cuda-gdb) print data[threadIdx.x]
```

> 查看连续内存还有一个 GDB 原生技巧：`print array[0]@12` 表示从 `array[0]` 开始打印 12 个元素，不用逐个线程切。

## 四、断点进阶

### 4.1 条件断点（定位特定线程的首选）

条件表达式里可以直接用 `threadIdx`、`blockIdx` 等内置变量和程序里的普通变量：

```bash
# 只在 threadIdx.x == 5 时中断
(cuda-gdb) break my_kernel if threadIdx.x == 5

# 多条件组合
(cuda-gdb) break foo.cu:23 if threadIdx.x == 1 && i < 5

# 给已经存在的 3 号断点追加/修改条件
(cuda-gdb) cond 3 blockIdx.x == 2 && threadIdx.x == 31
```

> 注意：条件表达式里不允许调用函数。

### 4.2 源文件行号断点

```bash
# 在 myapp.cu 第 42 行断（断点对所有执行到该行的 GPU 线程生效）
(cuda-gdb) break myapp.cu:42
```

命中后焦点默认在第一个触断的线程上，再配合第三节的 `cuda thread` 切换到你想看的那个。

### 4.3 内核入口断点

直接对内核函数名打断点，就是内核入口断点：

```bash
(cuda-gdb) break my_kernel
```

每次该内核被启动、任意线程一进入就会中断。模板函数需要写完整签名。

## 五、实战场景

### 5.1 调一个结果不对的内核

典型流程：

1. 先在内核入口打断点，确认参数传对了：

```bash
(cuda-gdb) break my_kernel
(cuda-gdb) run
# 命中后
(cuda-gdb) print d_in       # 指针地址对不对
(cuda-gdb) print *d_in@10   # 看前 10 个元素
(cuda-gdb) print N
```

2. 在出问题的那行断点 + 条件：

```bash
(cuda-gdb) break myapp.cu:55 if threadIdx.x == 7
(cuda-gdb) continue
# 命中后逐行看变量
(cuda-gdb) next
(cuda-gdb) print partial_sum
```

3. 对比 warp 内其他线程的值，用 3.4 节的循环脚本批量打印。

### 5.2 调一个段错误 / 非法内存访问

非法访问通常在 `cudaDeviceSynchronize` 后才报错，错误定位不到具体行。两种办法：

**方法一：用 compute-sanitizer 先定位（推荐第一步）**

```bash
compute-sanitizer ./myapp
# 旧版 CUDA 里叫 cuda-memcheck
```

它会精确打印出错的内核名、block/thread 坐标、访问地址和源码行，拿到这些信息再去 cuda-gdb 里精确打断点。

**方法二：进 cuda-gdb，用条件断点逐线程排查**

```bash
(cuda-gdb) break myapp.cu:55 if blockIdx.x == 2 && threadIdx.x == 31
```

异常发生时 cuda-gdb 也会直接收到信号（如 `CUDA_EXCEPTION_14, Warp Illegal Address`），并自动把焦点切到肇事线程：

```text
CUDA Exception: Warp Illegal Address
Thread 1 "myapp" received signal CUDA_EXCEPTION_14, Warp Illegal Address.
[Current focus set to CUDA kernel 1, grid 1, block (3,0,0), thread (32,0,0), device 0, sm 6, warp 1, lane 0]
```

> 也可以在调试器内部开启内存检查：`(cuda-gdb) set cuda memcheck on`。

### 5.3 调一个卡死的内核

内核死循环或者死锁、程序卡住不动时，直接按 **Ctrl+C** 中断，cuda-gdb 会把焦点设置到中断点，然后：

```bash
(cuda-gdb) bt                  # 看当前焦点线程的调用栈，停在哪一行
(cuda-gdb) info cuda threads   # 列出所有线程各自停在哪一行
(cuda-gdb) info cuda warps     # 看 warp 分布
# 怀疑某个线程，切过去看它的栈
(cuda-gdb) cuda thread (31,0,0)
(cuda-gdb) bt
```

死锁的典型场景：warp 内部分支发散后依赖彼此同步（比如不同线程走到不同的 `__syncthreads()`）。看 `info cuda threads` 里各线程的行号是否散落各处，以及 `info cuda lanes` 的活跃/发散掩码，就能判断控制流有没有对齐。

## 六、常用技巧

### 6.1 几个实用的 `set cuda` 开关

```bash
# 每次应用启动 CUDA 内核时都断下来（application 指你自己的内核，system 指系统内核）
(cuda-gdb) set cuda break_on_launch application

# CUDA API 调用一旦返回错误立即中断（比如 kernel launch 配置非法时）
(cuda-gdb) set cuda api_failures stop

# 开启设备端越界内存检查
(cuda-gdb) set cuda memcheck on
```

### 6.2 让内核启动变同步

如果你在**调试器外面**排查"错误不知道是哪次 launch 引起的"，用环境变量把内核启动变成同步的：

```bash
CUDA_LAUNCH_BLOCKING=1 ./myapp
```

这样错误会在发起启动的那一刻立刻暴露，而不是攒到下一次同步。注意这是 **shell 环境变量，不是 cuda-gdb 的命令**。

### 6.3 autostep：自动单步可疑代码区

全程序单步太慢，正常跑又抓不到精确出错行。`autostep` 可以只对你圈定的代码区间自动单步，区间外全速运行：

```bash
# 怀疑第 16 行附近，对该行起 3 行的窗口自动单步
(cuda-gdb) autostep 16 for 3 lines
(cuda-gdb) run
```

一旦异常发生在窗口内，会精确报出具体源码行。

### 6.4 TUI 模式

```bash
cuda-gdb -tui ./myapp
```

进入后按 `Ctrl+X, 2` 可以同时显示源码和汇编，对定位哪条指令出错非常有用。

### 6.5 保存 / 加载调试脚本

把常用断点和打印写成脚本：

```
# debug.gdb
break my_kernel
break myapp.cu:55 if threadIdx.x == 7
commands 2
  print idx
  print data[idx]
  continue
end
```

启动时加载：

```bash
cuda-gdb -x debug.gdb ./myapp
```

### 6.6 printf 调试与 cuda-gdb 配合

内核里写 `printf("tid=%d val=%f\n", tid, val);` 配合：

```cpp
if (threadIdx.x == 7) printf(...);   // 只在特定线程打印
```

注意内核 `printf` 的输出在 `cudaDeviceSynchronize` 后才会 flush，有时候看起来"没输出"其实只是还没同步。

## 七、常见坑

| 现象 | 原因 | 解决 |
|---|---|---|
| `No CUDA devices found` | 驱动版本太低 / GPU 被占用 | 升级驱动，确保没有其他进程独占 GPU |
| 变量值显示 `<optimized out>` | 用了 `-O2` 优化或没加 `-G` | 重新用 `-G -g -O0` 编译 |
| `print` 出来的值"像别的线程的" | 当前焦点不在你以为的线程上 | 先 `cuda thread` 确认，再 `cuda thread (x,y,z)` 切换 |
| 输入了 `cuda focus` 报 `Undefined cuda command` | 没有这个命令 | 查看焦点用 `cuda kernel block thread`，切换用 `cuda thread/block/...` |
| `printf` 在内核里没输出 | 未 `cudaDeviceSynchronize` | 在启动后加 `cudaDeviceSynchronize()` |
| 调试时性能极慢 | `-G` 会强制 -O0 并禁用大量硬件优化 | 调试用小数据量（grid/block 都调小） |
| 错误在 `cudaDeviceSynchronize` 才报，找不到行 | 内核启动是异步的 | 用 compute-sanitizer，或 `CUDA_LAUNCH_BLOCKING=1` |

## 八、速查表

| 命令 | 作用 |
|---|---|
| `cuda kernel block thread` | 查看当前软件坐标（内核/块/线程） |
| `cuda device sm warp lane` | 查看当前硬件坐标 |
| `cuda thread N` / `cuda thread (x,y,z)` | 切换焦点线程（N 为块内线性编号） |
| `cuda block (x,y,z)` / `cuda kernel N` | 切换焦点块 / 内核 |
| `cuda device 0 sm 1 warp 2 lane 0` | 按硬件坐标切换 |
| `info cuda kernels` | 列出活跃内核 |
| `info cuda blocks` / `info cuda threads` | 列出活跃块 / 线程（含所在行） |
| `info cuda warps` / `info cuda lanes` / `info cuda sms` | warp / lane / SM 状态 |
| `info cuda devices` | GPU 设备信息 |
| `print <var>` / `print arr[0]@N` | 打印焦点线程的变量 / 连续 N 个元素 |
| `break <kernel> if threadIdx.x == N` | 条件断点定位单线程 |
| `set cuda break_on_launch application` | 内核启动即断 |
| `set cuda api_failures stop` | CUDA API 报错即断 |
| `set cuda memcheck on` | 调试器内开启越界检查 |
| `autostep 16 for 3 lines` | 只对可疑区间自动单步 |
| `CUDA_LAUNCH_BLOCKING=1 ./app` | 调试器外：强制同步启动（环境变量） |

## 小结

cuda-gdb 的核心思路就一句话——**先定位到出问题的具体线程，再像调 CPU 一样单步看变量**。难点在于 GPU 的并行性让你需要时刻清楚"当前焦点是哪个线程"：不带参数的 `cuda thread/block/...` 用来查，带参数用来切，`info cuda threads` 用来总览。

日常开发建议这么用：

1. 用 `compute-sanitizer` 快速定位出错内核、线程坐标和源码行；
2. 进 cuda-gdb，用条件断点锁定到那个线程；
3. 单步 + `print` 查变量；
4. 需要横向对比时，用 `while` 循环批量切换 `cuda thread` 打印整个 warp 的值。

写内核调试这种事情，熟能生巧，多按这套流程走几遍，你会发现 GPU 里"有一个线程算错了"这种 bug 比想象中好定位得多。
