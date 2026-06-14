# AI Infra Learning Plan

## Intuition

按"自底向上"的路线学习 AI Infra，从前置知识到推理部署，每阶段有明确的产出和检验标准。总周期约 8-12 周，可根据已有基础跳过部分内容。

## 学习路线图

```
Week 1-2          Week 3-5             Week 6-8           Week 9-12
┌──────────┐   ┌───────────────┐   ┌──────────────┐   ┌──────────────┐
│ 第零层    │ → │ 第一层         │ → │ 第二层        │ → │ 第三层        │
│ 前置知识  │   │ CUDA + 算子    │   │ 分布式训练    │   │ 推理与部署    │
│          │   │               │   │              │   │              │
│ Python   │   │ GPU 架构      │   │ 数据并行     │   │ KV Cache     │
│ C/C++    │   │ CUDA 编程     │   │ 张量并行     │   │ PagedAttn    │
│ 线性代数  │   │ Reduce/GEMM   │   │ 流水线并行   │   │ Batching     │
│ 概率论    │   │ FlashAttn     │   │ ZeRO         │   │ 量化         │
│ Transf.  │   │ Triton        │   │ 混合精度     │   │ SpecDecoding │
│ PyTorch  │   │ Profiling     │   │ Megatron     │   │ vLLM/SGLang  │
│ 通信拓扑  │   │               │   │              │   │              │
└──────────┘   └───────────────┘   └──────────────┘   └──────────────┘
   1-2 周          3 周               3 周               4 周
```

## 阶段 0：前置知识（1-2 周）

**目标**：建立最小够用的心智模型。

| 主题 | 产出 | 参考 |
|------|------|------|
| Transformer 架构 | 白板默写 Decoder Block + 维度标注 | [[LLM Inference Basics]] |
| Python + C/C++ | 能读懂 CUDA host 端代码 | [[CUDA Programming]] |
| PyTorch | 独立写出完整训练循环 | Karpathy: Let's build GPT from scratch |
| 线性代数 | 看到 (B,S,H)×(H,V) 知道结果是 (B,S,V) | 3Blue1Brown 线代本质 |
| GPU 存储层次 | 理解为什么显存带宽是 Decode 瓶颈 | [[GPU Memory Hierarchy]] |
| 通信拓扑 | 看懂 nvidia-smi topo -m 输出 | NVIDIA NCCL 文档 |

**跳过条件**：能白板默写 Transformer Block + 手算 7B 模型参数量 → 直接进入阶段 1。

## 阶段 1：CUDA 编程与算子优化（3 周）

**目标**：能写出基础 CUDA Kernel，理解 FlashAttention 原理。

| 周 | 主题 | 练习 |
|---|------|------|
| W1 | GPU 架构 + CUDA 基础 | 写 Vector Add / Matrix Transpose Kernel |
| W2 | Reduce + GEMM 优化 | Reduce 三连（原子→共享内存→Warp Shuffle）；GEMM Tiling |
| W3 | FlashAttention + Triton | 白板推导 FlashAttention tiling；用 Triton 写 fused Softmax |

**关键检验**：
- GEMM Kernel 达到 cuBLAS 50% 以上性能
- FlashAttention 白板推导完整过程
- Nsight Systems 抓 trace 判断 kernel 是 memory-bound 还是 compute-bound

**详细资料** → [[CUDA Programming]] | [[Flash Attention]] | [[GPU Memory Hierarchy]]

## 阶段 2：分布式训练（3 周）

**目标**：理解 3D 并行策略，能设计并行拓扑。

| 周 | 主题 | 练习 |
|---|------|------|
| W4 | 数据并行（DP/DDP/FSDP） | 30 分钟把单卡脚本改成 DDP |
| W5 | 模型并行（TP/PP/SP）+ ZeRO | 手算 ZeRO-1/2/3 各自的显存节省量 |
| W6 | 混合精度 + 训练框架实战 | 用 DeepSpeed/Megatron-LM 跑通分布式训练 |

**关键检验**：
- 能手算 7B FP16 训练的完整显存需求（参数 + 优化器 + 激活）
- 给 64 卡集群设计 TP=8/PP=4/DP=2 的 3D 并行拓扑
- 能解释 BF16 vs FP16 为什么大模型训练选 BF16

**详细资料** → [[Distributed Training]] | [[Tensor Parallelism]]

## 阶段 3：推理与部署（4 周）

**目标**：能端到端部署推理服务，理解核心优化技术。

| 周 | 主题 | 练习 |
|---|------|------|
| W7 | 推理基础 + KV Cache + PagedAttention | 手算 KV Cache 显存占用；阅读 nano-vLLM 源码 |
| W8 | 推理引擎（vLLM/SGLang）+ Continuous Batching | 用 vLLM 和 SGLang 分别部署，跑 benchmark 对比 |
| W9 | 量化 | 同一模型跑 FP16 vs AWQ-INT4 对比测试 |
| W10 | Speculative Decoding + 综合实战 | 完整推理服务部署 + 性能调优 |

**关键检验**：
- 能手算 LLaMA-2-7B 在 batch=16、seq=4096 下的 KV Cache 总量
- 端到端部署 vLLM，输出 TTFT/TPOT/throughput 对比表
- 能给出结构化的推理框架选型建议

**详细资料** → [[KV Cache]] | [[Paged Attention]] | [[Continuous Batching]] | [[LLM Serving Architecture]] | [[Quantization]] | [[Speculative Decoding]]

## 推荐学习源

- [AIInfraGuide 学习路线](https://caomaolufei.github.io/AIInfraGuide/guides/ai-infra%E5%AD%A6%E4%B9%A0%E8%B7%AF%E7%BA%BF/) — 本计划的主要参考
- [nano-vLLM](https://github.com/GeeeekExplorer/nano-vllm) — 1.2k 行的教学级推理引擎
- [Infrasys-AI/AIInfra](https://github.com/Infrasys-AI/AIInfra) — AI Infra 代码库与 Notebook 练习

## Related

* [[AI Infra Learning MOC]]
* [[Edge AI and World Model]]

## Tags

#ai-infra #learning-plan #roadmap
