# Flash Attention

## Intuition

把一张大桌子上的拼图分成小块，每次只搬一小块到手边（SRAM）拼好再搬下一块，而不是把所有碎片一股脑倒出来占满桌面（HBM）。通过 tiling 减少 HBM 读写次数。

## Key Ideas

- 标准 Attention 的 HBM 读写是 O(N²)，Flash Attention 降到 O(N)
- 核心：分块计算 + Online Softmax（避免两遍遍历）
- 数学结果完全等价，不是近似
- 是 [[CUDA Programming]] 高级优化的典型案例

## 标准 Attention 的问题

```
S = QK^T / sqrt(d)       ← 需要 N×N 矩阵，HBM 读写 O(N²)
P = softmax(S)            ← 需要完整 S 矩阵
O = PV                    ← 需要 N×N 矩阵

问题：N×N 矩阵远超 SRAM 容量，必须反复读写 HBM
```

## Flash Attention 的解法

```
外层循环：遍历 KV 的 Block
  内层循环：遍历 Q 的 Block
    每个 Tile 在 SRAM 中完成:
      1. Q_tile × K_tile → S_tile
      2. Online Softmax 更新（维护 running max 和 running sum）
      3. S_tile × V_tile → 累加到 O_tile

关键：不需要在 HBM 中存储完整的 N×N S 和 P 矩阵
```

## Online Softmax

```
传统 Softmax: 需要先遍历一遍找 max，再遍历一遍算 exp/max/sum
Online Softmax: 维护 running max(m) 和 running sum(l)

新 tile 到来时:
  m_new = max(m_old, max(S_tile))
  l_new = l_old × exp(m_old - m_new) + sum(exp(S_tile - m_new))
  O_new = O_old × (l_old × exp(m_old - m_new) / l_new) + ...
```

## 版本演进

| 版本 | 改进 |
|------|------|
| FlashAttention V1 | 分块计算 + Online Softmax，O(N) HBM 读写 |
| FlashAttention V2 | 更好的并行策略（沿 seq_len 维度并行） |
| FlashAttention V3 | Hopper 架构优化（异步 + Tensor Core） |
| Flash-Decoding | Decode 阶段的 Attention 加速 |
| FlashInfer | 可组合的 Attention 引擎，面向 Serving |

## 学习资料

- [FlashAttention V1 Paper](https://arxiv.org/abs/2205.14135) — Memory-aware Attention 里程碑
- [FlashAttention V2 Paper](https://arxiv.org/abs/2307.08667)
- [猛猿：图解FlashAttention V1/V2](https://zhuanlan.zhihu.com/) — 适合新手入门
- [方佳瑞：深入浅出理解PagedAttention CUDA实现](https://zhuanlan.zhihu.com/)

## Related

* [[CUDA Programming]]
* [[GPU Memory Hierarchy]]
* [[Paged Attention]]
* [[KV Cache]]

## Tags

#ai-infra #flash-attention #cuda #attention #optimization
