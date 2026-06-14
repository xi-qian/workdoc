# Tensor Parallelism

## Intuition

把模型的一个层的权重矩阵切分到多个 GPU 上，每张卡算一部分，然后通信合并结果。同一层由多卡协同完成计算。

## Key Ideas

- 按列或按行切分权重矩阵（Column Parallel / Row Parallel）
- 通信发生在层内，每层需要一次 AllReduce
- 与 Pipeline Parallelism 互补（TP 切层内，PP 切层间）
- nano-vLLM 支持 TP 并行运行 Qwen3 模型

## Column Parallel vs Row Parallel

| | Column Parallel | Row Parallel |
|--|----------------|--------------|
| 切分方式 | 按列切权重矩阵 | 按行切权重矩阵 |
| 输入 | 每张卡拿到完整输入 | 每张卡拿到切分后的输入 |
| 输出 | 每张卡产出一部分列 | 需要累加所有卡的结果 |
| 通信 | 输出端 AllReduce | 输入端 AllReduce |

## MLP 层的 TP 示意

```
单卡: X → [W1] → gate → [W2] → output
       (hidden → intermediate)  (intermediate → hidden)

TP=2:  GPU 0: X → [W1_col0] → gate_0 → [W2_row0] ─┐
       GPU 1: X → [W1_col1] → gate_1 → [W2_row1] ─┤→ AllReduce → output
```

## Attention 层的 TP

- Q/K/V 矩阵按 Head 数切分（每张卡负责部分 Head）
- Output 投影按行切分
- GQA 模型中，KV Head 可能少于 Q Head，需要特殊处理

## 通信开销

```
每层 2 次 AllReduce（Attention + MLP 各 1 次）
每次 AllReduce 数据量 = batch_size × seq_len × hidden_dim × dtype_size

优化方向：
- 通信与计算重叠（NCCL pipelining）
- 减少 AllReduce 次数（融合 kernel）
```

## 学习资料

- [nano-vLLM 学习后记](https://zhuanlan.zhihu.com/p/1977336847567983629) — 第6节：张量并行
- [Megatron-LM: Efficient Large-Scale Language Model Training](https://arxiv.org/abs/2104.04473)

## Related

* [[LLM Inference Basics]]
* [[CUDA Programming]]
* [[GPU Memory Hierarchy]]
* [[LLM Serving Architecture]]

## Tags

#ai-infra #tensor-parallelism #distributed #multi-gpu
