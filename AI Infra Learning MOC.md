# AI Infra Learning MOC

## Overview

AI Infra（人工智能基础设施）是大模型时代壁垒最高、最核心的技术高地。本质是"用系统工程释放硬件算力"。

所有优化都在"计算、通信、显存"不可能三角中取舍：
- ZeRO → 用通信换显存
- 重计算（Activation Checkpointing）→ 用计算换显存
- 量化 → 用精度换显存和带宽

**参考路线**：[AIInfraGuide 学习路线](https://caomaolufei.github.io/AIInfraGuide/guides/ai-infra%E5%AD%A6%E4%B9%A0%E8%B7%AF%E7%BA%BF/) | [nano-vLLM 学习资料汇编](https://zhuanlan.zhihu.com/p/2032406199111521673)

## 第零层：前置知识

编程语言、数学基础、Transformer 架构、PyTorch、通信拓扑。

- [[LLM Inference Basics]] — Prefill/Decode 两阶段推理流程
- [[GPU Memory Hierarchy]] — GPU 存储层次与带宽瓶颈
- [[CUDA Programming]] — CUDA 编程模型与在推理引擎中的应用

## 第一层：CUDA 编程与算子优化

GPU 架构、存储层次、Kernel 编写、FlashAttention、AI 编译器。

- [[CUDA Programming]] — Grid/Block/Thread、内存模型、关键概念
- [[Flash Attention]] — 分块计算 + Online Softmax，O(N²) → O(N) HBM 读写
- [[GPU Memory Hierarchy]] — 寄存器 > SRAM > L2 > HBM，写 Kernel 的核心是最大化数据复用

## 第二层：分布式训练

数据并行、3D 并行、ZeRO、混合精度。

- [[Distributed Training]] — DP/DDP/FSDP、TP/PP/SP、ZeRO-1/2/3
- [[Tensor Parallelism]] — Column/Row Parallel，Attention 层的 TP

## 第三层：推理与部署

KV Cache、PagedAttention、量化、Speculative Decoding。

- [[KV Cache]] — 缓存 Key/Value 避免重复计算，显存随序列线性增长
- [[Paged Attention]] — 虚拟内存分页管理 KV Cache，按需分配 Block
- [[Continuous Batching]] — 动态组批，请求随到随处理
- [[LLM Serving Architecture]] — Engine → Scheduler → BlockManager → ModelRunner
- [[Quantization]] — SmoothQuant/GPTQ/AWQ/FP8，用精度换显存和带宽
- [[Speculative Decoding]] — Draft-then-Verify，用小模型猜测大模型验证

## 检验标准速查

| 层级 | 核心检验 |
|------|---------|
| 第零层 | 白板默写 Transformer Block + 维度推导 |
| 第一层 | Reduce 三连 + GEMM 分块 + FlashAttention 白板推导 |
| 第二层 | 显存账本 + 3D 并行拓扑设计 + DDP 改造 |
| 第三层 | 端到端部署 + KV Cache 算账 + 量化方案选型 |

## Tags

#moc #ai-infra #learning
