---
title: vLLM 详解：从 PagedAttention 到 V1 引擎的高吞吐 LLM 推理
date: 2026-09-22 10:00:00
categories:
  - AI推理
tags:
  - vLLM
  - LLM
  - 推理优化
  - PagedAttention
  - GPU
---

大模型自托管部署，几乎绕不开 vLLM。它把 HuggingFace 上的开源模型变成一条 OpenAI 兼容的高吞吐推理服务，凭借 PagedAttention 一项技术把同等硬件下的吞吐拉高了数倍，如今已成为事实上的 LLM 推理引擎标准。本文讲清楚 vLLM 解决什么问题、核心技术原理、2026 年 V1 引擎的架构现状，以及如何快速上手（内容以 [vLLM 官方文档](https://docs.vllm.ai/) 与 [Inside vLLM 技术博客](https://vllm.ai/blog/2025-09-05-anatomy-of-vllm) 为准）。

## 一、vLLM 是什么

vLLM 是一个开源的 **LLM 推理与服务引擎**，2023 年诞生于 UC Berkeley RISELab，现在是 [vllm-project](https://github.com/vllm-project/vllm) 下的社区项目，几乎所有主流厂商（NVIDIA、AMD、Intel、Google TPU）都为它提供后端支持。

它做的事情可以一句话概括：**加载一个 HuggingFace 模型权重，对外提供高吞吐、低延迟的文本生成 API**。

```bash
vllm serve Qwen/Qwen3-8B
# 一条命令得到一个 OpenAI 兼容服务：http://localhost:8000/v1/chat/completions
```

和直接用 Transformers 跑模型相比，vLLM 的定位差异是：

| | Transformers 脚本 | vLLM |
|---|---|---|
| 设计目标 | 模型研究/功能验证 | 生产级高并发服务 |
| 请求处理 | 一次一个（或自己写 batch） | 连续批处理，成百上千请求并发 |
| 显存管理 | 预分配、易 OOM | PagedAttention 分页管理 KV Cache |
| 服务接口 | 自己封装 | 内置 OpenAI 兼容 HTTP API |
| 多卡/多机 | 手动 | 张量并行/流水线并行/PD 分离内置 |
| 典型吞吐比 | 1× | 10×~24×（官方早期 benchmark） |

## 二、要解决的核心矛盾：KV Cache 吃掉显存

理解 vLLM 的价值，先要理解自回归推理的显存账。

LLM 生成文本分两阶段：

- **Prefill（预填充）**：把用户输入的 prompt 一次性并行算完，计算密集；
- **Decode（解码）**：逐 token 自回归生成，每生成一个新 token，都要用到之前**所有 token 的 Key/Value 向量**，这就是 **KV Cache**。

KV Cache 有多大？以 70B 模型（80 层、8192 维、128 头）为例，单条 4K 长度对话的 KV Cache 约占 **1GB 显存**。而模型权重本身在 FP16 下约 140GB。也就是说：

- 不做优化时，一张 80GB H100 跑 70B 连权重都放不下；
- 即使放得下，剩余显存有一大半要留给 KV Cache，并发数直接受限于"能缓存多少条对话"。

传统推理框架（FasterTransformer、TGI 早期版本）的做法是给**每个请求预分配一块连续的最大长度显存**（比如预留 4096 token 的空间），这带来两个致命浪费：

1. **内部碎片**：大多数对话远到不了 4096 token，预留空间空着；
2. **外部碎片**：请求结束释放后，显存被切成大小不一的空洞，新请求塞不进去，哪怕总空闲量足够。

实测中这种方式的显存利用率常常只有 20%~40%。**vLLM 的全部优化，本质上都是围绕"怎么把 KV Cache 管理好"展开的。**

## 三、PagedAttention：像操作系统管内存一样管 KV Cache

PagedAttention 是 vLLM 的立身之本，论文发表于 SOSP'23。核心灵感直接借自操作系统的**虚拟内存分页**：

- 不再给请求分配连续大块显存，而是把 KV Cache 切成固定大小的 **Block**（默认每块存 16 个 token 的 KV）；
- 每个请求维护一张逻辑块表（Block Table），逻辑块通过它映射到物理块；
- 物理块**按需分配、不必连续**，请求生成到哪、新块分到哪。

```text
传统方式（连续分配）                PagedAttention（分页）
请求A: [████████░░░░░░░░]         逻辑块:  A0 A1 A2
      预分配整块，尾部浪费           物理块: [A2][C1][A0][B1][A1][C0]
请求B: [██████░░░░░░░░░░]                 任意拼接，零内部碎片
      释放后留下难利用的空洞           Block Table: A0→#3, A1→#5, A2→#0
```

这一个设计改动带来三个直接收益：

1. **内部碎片趋近于零**：最坏情况只浪费最后一个块（< 16 token 的空间）；
2. **外部碎片消失**：所有物理块等大，释放即可被任何请求复用；
3. **共享变得天然**：多个请求的相同前缀（system prompt、few-shot 示例）可以让逻辑块映射到**同一个物理块**，写时复制——这就是 Prefix Caching 的底层基础。

官方论文数据：PagedAttention 把 KV Cache 显存利用率提升到 90% 以上，同等硬件下吞吐量达到当时 SOTA 方案的 2~4 倍。

## 四、Continuous Batching：请求级别的动态拼批

光有分页还不够，GPU 利用率还取决于"每个 step 能同时算多少请求"。

**静态批处理（static batching）**：凑齐一批请求一起进，必须等最长的那个生成完，整批才能释放。短请求早早算完也得干等，GPU 算力空转。

**连续批处理（continuous batching / in-flight batching）**：vLLM 的调度器以**每个解码 step（一次前向）为粒度**重新组批——

- 每完成一步，生成结束的请求立刻退出批次；
- 等待中的新请求立刻插入下一步的批次；
- prefill 和 decode 的请求还能通过 chunked prefill 混在同一批次里。

```text
静态批：  [A B C D 一起开始] -----> 等 D 结束才整体释放
连续批：  step1: A B C
          step2: A B C D(新插入)
          step3: A(结束) B C D
          step4: B C D E(新插入)
```

因为 PagedAttention 已经让每个请求的 KV Cache 变成了可独立寻址的块表，调度器才得以在每一步自由地增删请求而不需要连续显存——两项技术是咬合在一起的。

## 五、其他几项关键优化

### 5.1 Prefix Caching（前缀缓存）

多条请求共享相同前缀时（固定 system prompt、RAG 文档、多轮对话历史），自动复用已算好的 KV 物理块，命中部分跳过 prefill，直接省掉计算。V1 中默认开启，对 Agent 场景（每次都带长 system prompt + 工具定义）收益极大。

### 5.2 Chunked Prefill（分块预填充）

超长 prompt 的 prefill 会独占 GPU 很多个 step，导致正在 decode 的请求 token 间隔（ITL）飙高。Chunked prefill 把长 prefill 切成固定 token 预算的小块，与 decode 请求穿插执行。**V1 中默认开启**，由统一调度器按每请求的 token 预算字典动态分配。

### 5.3 Speculative Decoding（投机解码）

用小模型（draft model）一次猜多个 token，大模型（主模型）一次前向并行验证，猜对的直接接受。典型能获得 2~3 倍 decode 加速，vLLM 支持多种投机算法（draft model、ngram、DSpark block-diffusion 等）。

### 5.4 量化与内核

内置 AWQ、GPTQ、FP8、MXFP4/NVFP4（4-bit）等量化方案，并集成 FlashAttention、FlashInfer 等高性能注意力内核。显存不够时用量化版权重，往往能在几乎不掉点的情况下把并发再提一档。

### 5.5 并行策略

- **张量并行（TP）**：单层权重切到多卡，NVLink 机内首选；
- **流水线并行（PP）**：按层切分到多机，跨节点首选；
- **数据并行（DP）**：多副本各自服务，配合负载均衡扩吞吐；
- **PD 分离**：prefill 和 decode 拆到不同实例（见下节）。

## 六、2026 年的现状：V1 引擎全面取代 V0

vLLM 在 2025 年 1 月发布了 V1 核心引擎（[V1 Alpha 博客](https://blog.vllm.ai/2025/01/27/v1-alpha-release.html)），到 2026 年 **V0 已完全废弃**（见 [RFC #18571](https://github.com/vllm-project/vllm/issues/18571)）。重写的动机和收益：

**为什么重写**：V0 各功能（调度器、KV 管理器、worker、采样器、API server）是独立长出来的，技术债堆积，新优化难以叠加。

**V1 的设计目标**：

- 简单、模块化、易 hack 的代码库；
- 近零 CPU 开销的高性能调度；
- 把 chunked prefill、prefix caching、speculative decoding 等优化统一进一套架构；
- 零配置——优化默认全开。

**架构上最关键的变化是统一调度器**：V0 严格区分 prefill 和 decode 两类请求；V1 把两者**一视同仁**，用一个简单字典 `{request_id: num_tokens}` 给每个请求动态分配固定 token 预算。这一个抽象让 chunked prefill、prefix caching、speculative decoding 不再需要各自特殊的调度路径。同时 V1 不再需要 GPU↔CPU 的 KV Cache 交换来处理抢占（简化了核心架构）。

硬件覆盖在 V1 也已齐全：NVIDIA、AMD、Intel GPU、TPU、CPU 全部 Functional，昇腾（Ascend）、Gaudi、OpenVINO 等通过插件支持。

### PD 分离（Disaggregated Prefilling）

这是当前生产部署最热的特性（实验性）：把 prefill 实例和 decode 实例拆开。

- **为什么**：prefill 是计算密集型，decode 是显存/带宽密集型，两者混部时 prefill 会打断 decode 造成尾延迟（tail ITL）。拆开后可以分别调优 TTFT（首 token 延迟）和 ITL（token 间隔），还能给两类实例配不同的并行策略。
- **怎么传 KV**：prefill 算完的 KV Cache 通过 KV Connector 搬到 decode 实例，目前支持 9 种 connector——NIXL（基于 UCX/GDS，RDMA 直传）、Mooncake（月之暗面开源）、LMCache、CPU Offload、MultiConnector 组合等。
- **注意**：PD 分离优化的是**延迟**，不提高吞吐；吞吐要靠 continuous batching。

## 七、快速上手

### 7.1 安装

需要 Linux + NVIDIA GPU（Windows 需走 WSL2）：

```bash
# 推荐：新建干净环境
conda create -n vllm python=3.11 -y && conda activate vllm

# 直接 pip 安装（带预编译 CUDA 内核）
pip install vllm
```

### 7.2 启动 OpenAI 兼容服务

```bash
vllm serve Qwen/Qwen3-8B \
  --max-model-len 32768 \
  --gpu-memory-utilization 0.9 \
  --enable-prefix-caching
```

起服务后直接用 OpenAI SDK 调用：

```python
from openai import OpenAI

client = OpenAI(base_url="http://localhost:8000/v1", api_key="EMPTY")

resp = client.chat.completions.create(
    model="Qwen/Qwen3-8B",
    messages=[{"role": "user", "content": "用三句话解释什么是 KV Cache"}],
    temperature=0.7,
)
print(resp.choices[0].message.content)
```

### 7.3 离线批量推理（Python API）

做数据处理/评测时不需要 HTTP 服务，直接用 LLM 类：

```python
from vllm import LLM, SamplingParams

llm = LLM(model="Qwen/Qwen3-8B", gpu_memory_utilization=0.9)

prompts = ["写一首关于驱动开发的七言绝句", "解释 RCU 锁的原理", "什么是 MMU？"]
sampling = SamplingParams(temperature=0.7, max_tokens=512)

outputs = llm.generate(prompts, sampling)
for out in outputs:
    print(out.outputs[0].text)
```

一次传入整个 prompt 列表，vLLM 内部自动做 continuous batching——这是最简单感受它吞吐威力的方式。

### 7.4 多卡张量并行

```bash
# 单机 4 卡跑 70B
vllm serve Qwen/Qwen3-235B-A22B --tensor-parallel-size 4
```

### 7.5 常用参数速查

| 参数 | 作用 |
|---|---|
| `--gpu-memory-utilization 0.9` | 允许使用的显存比例（默认 0.9） |
| `--max-model-len 32768` | 最大上下文长度，影响 KV 预留 |
| `--enable-prefix-caching` | 前缀缓存（V1 默认开） |
| `--tensor-parallel-size N` | 机内张量并行卡数 |
| `--pipeline-parallel-size N` | 流水线并行（跨机） |
| `--quantization awq/fp8` | 权重量化格式 |
| `--max-num-seqs N` | 单步最大并发序列数 |
| `--served-model-name xxx` | API 中暴露的模型名 |

## 八、适用场景与局限

**适合**：

- 自托管开源对话模型（Qwen、Llama、DeepSeek、Kimi、GLM 等）；
- 高并发 API 服务、RAG、Agent 后端；
- 离线批量生成（数据合成、评测、标注）；
- 多卡/多机大模型生产部署。

**不适合 / 需要注意**：

- **Windows 原生不支持**，必须 WSL2 或 Linux；
- 对只有核显/小显存（<8GB）的机器意义不大，这类场景考虑 llama.cpp；
- PD 分离等高级特性仍标 experimental，升级版本时要看 [V1 迁移指南](https://docs.vllm.ai/en/latest/usage/v1_guide/)；
- 它是**推理**引擎，不做训练/微调。

## 九、小结

vLLM 的成功可以归纳为一条清晰的技术主线：

> 用操作系统的虚拟内存思想（PagedAttention 分页）解决 KV Cache 碎片 → 有了分页才有请求级动态拼批（continuous batching）→ 统一调度器（V1）把 prefix caching、chunked prefill、speculative decoding 全部收进同一套 token 预算抽象 → 再通过 KV Connector 把架构延伸到 PD 分离的分布式部署。

记住三个数字概念就抓住了本质：**KV Cache 是显存大头、Block 是管理单位、step 是调度粒度**。想深入实现细节，推荐直接读官方长文 [Inside vLLM: Anatomy of a High-Throughput LLM Inference System](https://vllm.ai/blog/2025-09-05-anatomy-of-vllm)，以及源码中的 `vllm/v1/core/sched/`（调度器）与 `vllm/v1/core/kv_cache_manager.py`（块表管理）。

## 参考文档

- [vLLM 官方文档](https://docs.vllm.ai/) — 安装、参数、特性总览
- [vLLM V1 用户指南](https://docs.vllm.ai/en/latest/usage/v1_guide/) — V1 与 V0 差异、特性支持矩阵
- [Inside vLLM: Anatomy of a High-Throughput LLM Inference System](https://vllm.ai/blog/2025-09-05-anatomy-of-vllm) — 官方架构深度长文
- [Disaggregated Prefilling](https://docs.vllm.ai/en/latest/features/disagg_prefill/) — PD 分离与 KV Connector
- [PagedAttention 论文（SOSP'23）](https://arxiv.org/abs/2309.06180) — 原始论文
- [vllm-project/vllm（GitHub）](https://github.com/vllm-project/vllm)
