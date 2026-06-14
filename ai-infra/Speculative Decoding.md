# Speculative Decoding

## Intuition

让实习生（小模型）先快速起草一段文字，资深主编（大模型）一次性审阅：猜对的直接用，猜错的当场改。保证输出分布不变，但生成速度显著提升。

## Key Ideas

- 用 Draft 模型批量猜测多个 token，Target 模型一次验证
- 数学上保证输出分布与原始自回归完全一致（无偏）
- 适合 memory-bound 的 Decode 阶段，利用闲置算力
- 关键指标：acceptance rate（Draft 命中率）

## 工作流程

```
Draft 模型: 生成 K 个候选 token [t1, t2, ..., tK]
    ↓
Target 模型: 一次 forward 验证所有候选
    ↓
接受/拒绝:
  - t1 正确 → 接受
  - t2 正确 → 接受
  - t3 错误 → 拒绝，用 Target 在 t3 位置的正确分布重新采样
    ↓
输出: [t1, t2, correct_t3]（至少 1 个正确 token）
```

## 主流方案

| 方案 | Draft 来源 | 特点 |
|------|-----------|------|
| Speculative Sampling | 外部小模型 | 经典框架，需额外维护 Draft 模型 |
| Medusa | 多个 Decoding Heads | 不需要外部模型，在同模型上加 Head |
| EAGLE-2 | 动态 Draft Tree | 利用校准置信度更激进地生成候选 |
| Block Speculative Decoding | 块级猜测 | 批量生成+验证，减少 kernel launch |

## 性能分析

```
加速比 = acceptance_rate × K / (1 + T_draft / T_target)

关键变量:
- acceptance_rate: Draft 与 Target 分布越接近，越高
- K: 候选 token 数
- T_draft / T_target: Draft 生成成本 vs Target 验证成本
```

## 学习资料

- [Speculative Decoding Paper](https://arxiv.org/abs/2211.17192)
- [Medusa Paper](https://arxiv.org/abs/2401.10774)
- [EAGLE-2 Paper](https://arxiv.org/abs/2406.16858)

## Related

* [[LLM Inference Basics]]
* [[LLM Serving Architecture]]
* [[Quantization]]

## Tags

#ai-infra #speculative-decoding #inference #latency
