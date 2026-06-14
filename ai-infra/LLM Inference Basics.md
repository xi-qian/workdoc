# LLM Inference Basics

## Intuition

LLM 推理本质是一个自回归循环：给一段 tokens，模型输出下一个 token 的概率分布，采样一个 token 追加到输入，重复直到结束。

## Key Ideas

- 推理分两个阶段：[[Prefill]] 和 [[Decode]]
- Prefill 阶段并行处理所有输入 token，是 compute-bound
- Decode 阶段逐个生成 token，是 memory-bound
- 推理性能瓶颈在 Decode 阶段的显存带宽

## Prefill vs Decode

| 维度 | Prefill | Decode |
|------|---------|--------|
| 输入 | 全部 prompt tokens | 1 个新 token |
| 计算 | 大矩阵乘法（并行） | 小向量运算（串行） |
| 瓶颈 | Compute-bound | Memory-bound |
| 产出 | 所有 token 的 KV | 1 个 token 的 KV + logits |

## 朴素推理循环

```python
def generate(model, input_ids, max_tokens):
    for _ in range(max_tokens):
        logits = model(input_ids)
        next_token = sample(logits[:, -1])
        input_ids = torch.cat([input_ids, next_token], dim=1)
        if next_token == eos_token:
            break
    return input_ids
```

问题：每步重复计算所有已生成 token 的 KV，浪费大量算力。

## 推理引擎的核心任务

1. **避免重复计算** → [[KV Cache]]
2. **高效管理显存** → [[Paged Attention]]
3. **提高 GPU 利用率** → [[Continuous Batching]]
4. **突破单卡限制** → [[Tensor Parallelism]]

## 学习资料

- [LLM模型推理入门](https://zhuanlan.zhihu.com/p/676168779) — 从 Prefill 到 Decode 的完整推理流程
- [transformer到LLM极速入门](https://zhuanlan.zhihu.com/p/691880270) — 图多文少，快速建立整体印象
- [大模型LLM知识整理](https://zhuanlan.zhihu.com/p/626539405) — 全景式知识整理

## Related

* [[KV Cache]]
* [[Paged Attention]]
* [[Continuous Batching]]
* [[GPU Memory Hierarchy]]
* [[AI Infra Learning MOC]]

## Tags

#ai-infra #llm-inference #prefill #decode
