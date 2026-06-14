# GPU Memory Hierarchy

## Intuition

理解推理性能优化的前提是搞清 GPU 的存储层次：从最快但最小的寄存器，到最慢但最大的 HBM。推理引擎的每个设计决策都在围绕这个层次做取舍。

## Key Ideas

- GPU 显存带宽是 Decode 阶段的瓶颈，不是算力
- SRAM（片上）极快但极小，HBM（片外）大但慢
- [[KV Cache]] 占用大量 HBM，是显存管理的主要对象
- [[Paged Attention]] 的目标就是减少 HBM 的浪费

## GPU 存储层次

```
┌──────────────────────────────────────────────┐
│ Registers          ~1 cycle   ~KB    (per SM) │  ← 最快最小
├──────────────────────────────────────────────┤
│ Shared Memory / L1 ~5 cycles  ~100KB (per SM) │
│ (SRAM)              可编程使用                  │
├──────────────────────────────────────────────┤
│ L2 Cache           ~50 cycles ~MB    (全局)    │
├──────────────────────────────────────────────┤
│ HBM (Global Memory) ~400 cycles ~80GB (全局)   │  ← 最慢最大
│ (A100: 2TB/s, H100: 3.35TB/s)                │
└──────────────────────────────────────────────┘
```

## 为什么推理是 Memory-bound

```
Decode 阶段每步:
  - 读取全部模型权重: ~140 GB (70B FP16)
  - 读取全部 KV Cache: ~10 GB
  - 计算: 1 个 token 的前向传播（极少量 FLOPS）
  → 瓶颈是加载权重和 KV Cache 的带宽，不是计算能力
```

## 关键比例：Arithmetic Intensity

```
Arithmetic Intensity = FLOPS / Bytes_transferred

Compute-bound: AI 高 → 算力是瓶颈（如 Prefill 阶段）
Memory-bound:  AI 低 → 带宽是瓶颈（如 Decode 阶段）
```

## 优化策略

| 策略 | 目标 | 对应技术 |
|------|------|---------|
| 减少权重加载 | 省 HBM 带宽 | 量化（INT8/FP8）、权重压缩 |
| 减少 KV 加载 | 省 HBM 带宽 | [[Paged Attention]]、KV Cache 量化 |
| 增大 batch | 摊薄权重加载开销 | [[Continuous Batching]] |
| 算子融合 | 减少访存次数 | Flash Attention、自定义 Kernel |

## 学习资料

- [CUDA编程入门极简教程](https://zhuanlan.zhihu.com/p/345661495) — 从硬件到代码
- [GPU Performance Background User Guide](https://docs.nvidia.com/deeplearning/performance/dl-performance-gpu-background/)

## Related

* [[LLM Inference Basics]]
* [[KV Cache]]
* [[CUDA Programming]]
* [[Paged Attention]]

## Tags

#ai-infra #gpu #memory #hbm #bandwidth
