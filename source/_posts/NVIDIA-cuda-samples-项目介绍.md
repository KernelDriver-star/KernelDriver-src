---
title: NVIDIA/cuda-samples 项目介绍：CUDA 官方示例仓库学习指南
date: 2026-09-22 15:00:00
categories:
  - CUDA
tags:
  - CUDA
  - cuda-samples
  - GPU编程
  - CUDA-Toolkit
---

学 CUDA 最好的教材不是某本书，而是 NVIDIA 官方维护的示例仓库 [NVIDIA/cuda-samples](https://github.com/NVIDIA/cuda-samples)。它曾经随 CUDA Toolkit 一起安装（老版本里的 `~/NVIDIA_CUDA-*/Samples`），后来独立到 GitHub 持续更新，目前已对齐 **CUDA Toolkit 13.4**。从装机后验证环境的 `deviceQuery`，到面试必考的 `reduction` 七种优化，再到 CUDA 13 全新的 Tile kernel 编程模型，全都在这里。本文系统介绍这个仓库的结构、内容和正确用法。

## 一、项目是什么

- **仓库地址**：[github.com/NVIDIA/cuda-samples](https://github.com/NVIDIA/cuda-samples)
- **定位**：演示 CUDA Toolkit 各项特性的**官方示例代码集**，不是教学文档，也不是测试套件——每个样例都是可独立编译运行的最小工程
- **技术栈**：C++/CUDA 为主（CMake 构建），CUDA 12.x 起新增 Python 示例（基于 [cuda.core](https://nvidia.github.io/cuda-python/cuda-core/latest/)）
- **平台覆盖**：Linux x86_64、Windows、aarch64（Tegra/Jetson）、DriveOS、QNX / QNX Safety，交叉编译工具链齐全
- **规模**：约 180 个可执行样例，按主题分 10 个 C++ 分类 + 4 个 Python 分类

官方特别强调一句容易被误解的话：**samples 不是 CUDA 的验证套件（validation suite）**——不覆盖边界情况、不追求 API 全覆盖、不适合做性能基准。它的价值是"演示一个特性最小、最规范的用法长什么样"。

## 二、仓库结构总览

克隆后目录非常规整，C++ 示例在 `cpp/`，Python 示例在 `python/`，用数字前缀编号，**编号本身就是学习顺序**：

```text
cuda-samples/
├── Common/              # 样例共用的辅助代码（图像加载、计时等）
├── cmake/               # 工具链文件（Tegra/QNX 交叉编译）
├── run_tests.py         # 批量跑全部样例的冒烟测试脚本
├── test_args.json       # 各样例运行参数配置
├── cpp/
│   ├── 0_Introduction/          # 入门：第一个 CUDA 程序
│   ├── 1_Utilities/             # 工具：查设备、测带宽
│   ├── 2_Concepts_and_Techniques/  # 核心算法范式
│   ├── 3_CUDA_Features/         # CUDA 平台特性
│   ├── 4_CUDA_Libraries/        # cuBLAS/cuFFT/NPP 等库
│   ├── 5_Domain_Specific/       # 图形、金融、图像等领域
│   ├── 6_Performance/           # 性能优化
│   ├── 7_libNVVM/               # NVVM IR / JIT 编译
│   ├── 8_Platform_Specific/     # Tegra/cuDLA/NvSci 平台专属
│   └── 9_CUDA_Tile/             # CUDA 13 全新 Tile 编程模型
└── python/
    ├── 1_GettingStarted/
    ├── 2_CoreConcepts/
    ├── 3_FrameworkInterop/      # 与 PyTorch/TensorFlow 互操作
    └── 4_DistributedComputing/  # 多卡、P2P、IPC
```

## 三、C++ 十大分类详解

### 0_Introduction：入门第一课

演示 CUDA 最基础的概念和 Runtime API，每个都只有几十行：

| 样例 | 学到什么 |
|---|---|
| `vectorAdd` | CUDA 的 "Hello World"：`__global__` kernel、`<<<grid, block>>>` 启动、`cudaMalloc/cudaMemcpy` |
| `vectorAddDrv` | 同一个向量加法的 **Driver API** 版本（`cuInit/cuModuleLoadData/cuLaunchKernel`），对照 Runtime API 看本质 |
| `vectorAddMMAP` | 用 `cuMemAddressReserve + cuMemMap` 的虚拟内存 API 做零拷贝映射 |
| `vectorAddUnifiedMemory` | Unified Memory（`cudaMallocManaged`）版本，感受页迁移 |
| `asyncAPI` | stream + event 的异步执行与计时 |
| `simpleAssert` | GPU 端 `assert()` 怎么报错、怎么定位出错线程 |

**建议入门顺序**：先跑通 `vectorAdd`，再对照读 `vectorAddDrv` 和 `vectorAddUnifiedMemory` 三个变体，一节课就能理解 CUDA 两种 API 与三种内存模型的区别。

### 1_Utilities：装机必跑

| 样例 | 用途 |
|---|---|
| `deviceQuery` | 枚举所有 GPU 的完整属性：SM 版本、显存、L2 大小、最大线程数、是否支持 ECC……**装完驱动第一件事就是跑它** |
| `bandwidthTest` | 测 host↔device、device↔device 的实际带宽，含 pinned memory 对比 |
| `p2pBandwidthLatencyTest` | 多卡 P2P（NVLink/PCIe）互连带宽与延迟矩阵 |
| `topologyQuery` | GPU 与 NUMA、PCIe switch 的拓扑关系 |

做驱动/性能工作时，这几个小工具是排查环境问题的第一手手段（输出还可以直接贴进 bug report）。

### 2_Concepts_and_Techniques：含金量最高的一类

CUDA 并行算法的经典范式基本都在这里，也是面试与调优的核心：

- **`reduction`**：数组归约，一个文件里递进演示 7 种优化（交错配对 → 共享内存 bank conflict 消除 → warp shuffle 无同步），堪称"单文件 GPU 优化教程"；
- **`scan`**：前缀和（并行扫描算法）；
- **`histogram`**：直方图，处理共享内存冲突；
- **`transpose`**：矩阵转置，专门演示 global memory 合并访问与 shared memory bank conflict 的取舍；
- **`matrixMul`**：分块矩阵乘法，tiling 思想的标准范本；
- **`sort` / `convolutionSeparable` / `particles` / `marchingCubes`**：排序、可分离卷积、粒子系统、等值面提取。

**这一类值得逐行精读**——工作中写新 kernel 遇到的 90% 的模式（tiling、归约、扫描、私有化+聚合）都有现成的规范实现。

### 3_CUDA_Features：平台特性演示

演示 CUDA 编程模型的具体机制：Cooperative Groups（线程协作原语）、CUDA Dynamic Parallelism（kernel 启动 kernel）、**CUDA Graphs**（把一串启动录制成图重复执行）、stream-ordered memory allocation（流有序内存分配）、可压缩内存、多进程 IPC、虚拟内存映射、TMA（Tensor Memory Accelerator）等。想确认"某个特性的 API 到底怎么用"，直接来这里抄骨架。

### 4_CUDA_Libraries：站在库的肩膀上

演示 CUDA 生态库的用法：cuBLAS（线性代数）、cuFFT（快速傅里叶）、cuSPARSE（稀疏矩阵）、cuSOLVER（求解器）、cuRAND（随机数）、NPP（图像/信号处理原语）、nvJPEG（JPEG 硬解码）、nvGRAPH（图分析）。生产代码原则是**能用库就别自己写 kernel**——比如矩阵乘永远优先 `cublasGemmEx` 而不是手写。

### 5_Domain_Specific：领域应用

图形互操作是重头戏：`simpleGL`、`simpleVulkan`、`simpleVulkanMMAP` 演示 OpenGL/Vulkan 与 CUDA 共享显存零拷贝；`nbody`、`fluidsGL`、`Mandelbrot` 是经典可视化 demo；金融类有蒙特卡洛期权定价；图像处理有高斯滤波、Sobel 边缘检测等。

### 6_Performance：榨干硬件

面向优化的进阶样例：**`cudaTensorCoreGemm`** 用 WMMA API 调用 Tensor Core 做混合精度矩阵乘；多个样例对比 naive 与优化版的耗时（用 CUDA event 计时），并配合 NVProf/Nsight 工具分析。学完第 2 类再来看这一类，能建立"为什么这样写更快"的完整认知。

### 7_libNVVM：编译器视角

`ptxgen` 等样例演示如何基于 libNVVM 把 LLVM IR（NVVM IR）在运行时 JIT 编译成 PTX。做 TVM、XLA、自研编译器或需要运行时生成 kernel 时才会用到，普通开发者了解即可。

### 8_Platform_Specific：嵌入式与车规

Tegra/Jetson 专属：cuDLA（深度学习加速器）、NvMedia、NvSciBuf/NvSciSync（安全缓存区与同步原语）、OpenGL ES/EGLOutput，以及面向 DRIVE OS 的样例。需要 `-DBUILD_TEGRA=True` 才编译，并支持 aarch64 交叉编译和 QNX/QNX Safety 工具链（CUDA 13.0+）。

### 9_CUDA_Tile：CUDA 13 的新编程模型（重点关注）

这是仓库里最新的分类，对应 CUDA 13 引入的 **CUDA Tile C++**——一套有别于传统 SIMT 的 tile 级 kernel 编程模型（`cuda::tiles::partition_view` 划分数据、`cuda::tiles::mma` 发射矩阵乘指令、tile kernel 与 SIMT kernel 可通过 global memory 协作）。样例直接覆盖当前主流 AI 算子：

| 样例 | 内容 |
|---|---|
| `helloTile` | tile kernel 启动、SIMT 与 Tile 间传数据 |
| `tileVectorAdd` / `tileTranspose` | partition_view 切分、masked load 处理边界 |
| `tileMatmul` | FP16 输入/FP32 累加的高性能矩阵乘，naive vs 优化对比 |
| `tileMatmulAutotuner` | 用 nvrtc/nvcc 自动搜索 tile 尺寸与优化提示 |
| `tileBmm` | persistent kernel + grid-stride loop 批量矩阵乘 |
| `tileLayerNorm` | 持久化 kernel 实现 LayerNorm |
| `tileRope` | RoPE 旋转位置编码（GPT-NeoX 风格） |
| `tileSpMV` | 稀疏矩阵向量乘 |

从 LayerNorm、RoPE、BMM 这些名字就能看出官方意图：**CUDA Tile 是面向 AI 算子时代的新内核写法**。做 LLM 推理/训练底层开发的话，这一类代表未来方向，值得尽早跟踪。

## 四、Python 示例：cuda.core 生态

`python/` 目录是后来新增的，定位与 C++ 版一一对应，但**不走 CMake**，每个样例独立 `pip install -r requirements.txt` 后直接运行（要求 Python 3.10+、CUDA 13.x）：

- `1_GettingStarted`：`vectorAdd`、`deviceQuery`、`systemInfo`、统一内存图像模糊、NumPy vs CuPy；
- `2_CoreConcepts`：reduction、histogram、FFT、流重叠、cudaGraphs、memoryResources、JIT/LTO、TMA tensor map；
- `3_FrameworkInterop`：与 PyTorch、TensorFlow 的互操作（数组/流共享）；
- `4_DistributedComputing`：多卡、P2P、IPC 内存池。

注意它用的是较新的 `cuda.core`（CUDA Python 下一代统一 API），而不是老的 `cuda-python` 风格，写法上更接近 C++ Runtime API 的一一映射。

## 五、构建方法

### 5.1 前置条件

安装 [CUDA Toolkit](https://developer.nvidia.com/cuda-downloads) 和 CMake 3.20+。Linux 下 `sudo apt install cmake` 即可。

### 5.2 Linux：全量构建

```bash
git clone https://github.com/NVIDIA/cuda-samples.git
cd cuda-samples
mkdir build && cd build
cmake ..
make -j$(nproc)
```

默认会为本版本支持的**所有 GPU 架构**编译，非常慢。自用的话强烈建议只指定自己卡的 SM 架构，构建时间能缩短一个数量级：

```bash
# 90 = sm_90（Hopper H100）；80 = A100；89 = RTX 40 系；120 = Blackwell
cmake -DCMAKE_CUDA_ARCHITECTURES=90 ..
```

### 5.3 只构建单个样例（最常用）

不需要全量编译，进入样例目录单独配置即可，但**必须显式指定架构**（独立构建没有顶层默认值）：

```bash
cd cpp/2_Concepts_and_Techniques/reduction
mkdir -p build && cd build
cmake -DCMAKE_CUDA_ARCHITECTURES=90 ..
make
./reduction
```

### 5.4 Windows

用 VS 2019 16.5+，在 "x64 Native Tools Command Prompt for VS" 里：

```bat
mkdir build && cd build
cmake .. -G "Visual Studio 16 2019" -A x64
```

然后打开生成的 `CUDA_Samples.sln`，F7 构建；也可以直接用 VS 打开任意子目录（CMake 语言服务原生支持）。

### 5.5 开启 GPU 端调试

默认编译关闭影响性能的调试信息。需要配合 `cuda-gdb` 断点调试时：

```bash
cmake -DENABLE_CUDA_DEBUG=True ..
```

它等价于给 nvcc 加 `-G`（注意 `-G` 和 `-lineinfo` 的区别：前者完整调试但大幅掉速，后者只保留行号信息，profiling 时用 `-lineinfo` 更合适）。

### 5.6 交叉编译与前向兼容

- **Tegra/Jetson**：`cmake .. -DCMAKE_TOOLCHAIN_FILE=../cmake/toolchains/toolchain-aarch64-linux.cmake -DTARGET_FS=<目标根文件系统>`
- **QNX / QNX Safety**：CUDA 13.0+ 支持，用对应 qnx 工具链文件，Safety 版走 CUDA Safe Toolkit（`-safety-compat`，目前只支持 `matrixMul` 和 `cudaNvSci`）
- **新 Toolkit + 旧 KMD 前向兼容**：UMD 580+ 配 KMD 550 及更早时，指定 stub 库路径：
  ```bash
  cmake -DCMAKE_PREFIX_PATH=/usr/local/cuda/lib64/stubs/ ..
  ```

## 六、批量冒烟测试：run_tests.py

换了驱动或环境后，可以跑一遍全部样例做 sanity check：

```bash
python3 run_tests.py \
  --dir ./build/cpp \
  --config test_args.json \
  --output ./test \
  --parallel 8
```

脚本递归找所有可执行文件，参数由 `test_args.json` 按程序名配置，支持三种模式：`skip`（跳过需要图形界面的，如 `fluidsGL`）、单次运行带参数、多组参数多次运行。全部通过输出：

```text
Test Summary:
Ran 199 test runs for 180 executables.
All test runs passed!
```

失败会列出程序名和返回码，日志在输出目录的 `APM_<name>.txt`。**它只验证"能不能跑起来"，不验证正确性覆盖**，再次提醒不要把它当合规测试。

## 七、推荐学习路径

根据不同目标，建议这样用这个仓库：

**路径 A：CUDA 入门（1~2 周）**
```text
0_Introduction/vectorAdd → vectorAddDrv → vectorAddUnifiedMemory → asyncAPI
→ 1_Utilities/deviceQuery → bandwidthTest
→ 2_Concepts_and_Techniques/matrixMul → reduction → transpose
```
配合 CUDA C++ Programming Guide 对应章节，边读边改参数量观察行为。

**路径 B：性能优化（有基础后）**
```text
2_Concepts/reduction（逐版本对比）→ transpose（合并访问/bank conflict）
→ 3_CUDA_Features/cudaGraphs、streamOrderedAllocation
→ 6_Performance/cudaTensorCoreGemm
```
每步用 Nsight Systems/Compute 看 profile，对照理解优化为什么生效。

**路径 C：AI 算子 / LLM 底层（前沿方向）**
```text
4_CUDA_Libraries 里 cuBLAS 用法
→ 9_CUDA_Tile/tileMatmul → tileMatmulAutotuner → tileLayerNorm → tileRope
→ python/3_FrameworkInterop（与 PyTorch 对接）
```

**路径 D：驱动/系统排障（日常工具）**
`deviceQuery`、`bandwidthTest`、`p2pBandwidthLatencyTest`、`topologyQuery` 四个直接编译好放 PATH 里，环境问题先跑它们。

## 八、几个实用提醒

1. **依赖是可选的**：图形样例依赖 FreeImage、GLFW、Vulkan SDK 等，缺库时样例会在构建阶段自动跳过（waive）而不是报错，全量构建看到 skipped 不用慌；
2. **样例代码允许直接抄**：官方仓库的代码就是规范模板——错误检查宏（`CUDA_CHECK`）、计时方式、grid/block 维度选择都可以直接搬到项目里；
3. **版本要对齐**：仓库 master 分支跟随最新 Toolkit（13.4），老环境请按 [CHANGELOG](https://github.com/NVIDIA/cuda-samples/blob/master/CHANGELOG.md) / tag 切对应版本，否则可能遇到 API 不存在；
4. **看 README 再跑**：每个样例目录下都有 README，写明它演示什么、支持的最低 SM 架构、命令行参数和预期输出，样例之间的代码风格高度统一，读起来成本很低；
5. **配合官方文档食用**：样例负责"怎么写"，[CUDA C++ Programming Guide](https://docs.nvidia.com/cuda/cuda-c-programming-guide/) 负责"为什么"，两者对照效率最高。

## 参考

- [NVIDIA/cuda-samples（GitHub）](https://github.com/NVIDIA/cuda-samples)
- [仓库 README（完整构建/交叉编译/测试说明）](https://github.com/NVIDIA/cuda-samples/blob/master/README.md)
- [CUDA Toolkit 下载](https://developer.nvidia.com/cuda-downloads)
- [CUDA C++ Programming Guide](https://docs.nvidia.com/cuda/cuda-c-programming-guide/)
- [CUDA Python / cuda.core 文档](https://nvidia.github.io/cuda-python/)
