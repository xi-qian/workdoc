# Quantization

## Intuition

量化本质是把"高清照片压缩成缩略图"——用更少的比特位表示权重，省下显存和带宽，代价是精度会有一定损失。

## Key Ideas

- 核心取舍：用精度换显存和带宽
- 权重量化（W8A8、INT4）减少模型大小
- [[KV Cache]] 量化减少推理显存占用
- FP8 是 Hopper 架构的原生支持方案

## 量化方案对比

| 方案 | 目标 | 精度影响 | 实现复杂度 |
|------|------|---------|-----------|
| W8A8 (SmoothQuant) | 通用、工程友好 | 小 | 低 |
| INT4 Weight-only (GPTQ/AWQ) | 极致省显存 | 中 | 中 |
| KV Cache 量化 (KIVI) | 长上下文/大并发 | 中 | 中 |
| FP8 | Hopper 原生 | 小 | 低 |

## 量化选择决策树

```
目标：省显存？省带宽？提吞吐？
├─ 通用、工程友好 → W8A8 (SmoothQuant)
├─ 更省显存/带宽 → INT4 weight-only (AWQ/GPTQ)
└─ 长上下文/大并发 → KV Cache 量化 (KIVI)
```

## 关键概念

- **Calibration**：用代表性数据校准量化参数
- **Per-channel vs Per-tensor**：按通道还是按张量量化
- **Outlier**：激活值中的异常大值，是量化的主要挑战
- **SmoothQuant**：把 activation 的 outlier 转移到 weights，工程友好

## 检验标准

- 能判断 70B 模型在 2×A100 80GB 上的量化策略选择
- 能解释为什么某些场景 INT4 反而比 INT8 慢
- 能排查量化后质量下降的原因（outlier、校准数据、特殊结构）

## 学习资料

- [SmoothQuant Paper](https://arxiv.org/abs/2211.10438) — W8A8 量化经典方案
- [GPTQ Paper](https://arxiv.org/abs/2210.17323) — Weight-only PTQ 3/4-bit
- [AWQ Paper](https://arxiv.org/abs/2306.00978) — 基于激活分布的 4-bit 量化
- [KIVI Paper](https://arxiv.org/abs/2402.02750) — 2-bit KV Cache 量化
- [vLLM 量化支持](https://docs.vllm.ai/en/latest/quantization/)

## Related

* [[KV Cache]]
* [[LLM Inference Basics]]
* [[LLM Serving Architecture]]

## Tags

#ai-infra #quantization #int4 #fp8 #inference
