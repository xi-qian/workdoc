# Paged Attention

## Intuition

借鉴操作系统虚拟内存分页机制管理 [[KV Cache]]。把 KV Cache 切成固定大小的块（Block），按需分配，避免预分配连续显存造成的浪费和碎片。

## Key Ideas

- KV Cache 按固定大小的 Block 组织（如 16 tokens/block）
- Block 是最小分配单位，不需要物理连续
- 通过 Block Table 映射逻辑序列到物理 Block
- 引用计数管理 Block 生命周期，支持共享（prefix caching）

## 传统方案 vs Paged Attention

| | 传统方案（预分配） | Paged Attention |
|--|------------------|-----------------|
| 分配方式 | 按 max_seq_len 预分配连续显存 | 按 Block 按需分配 |
| 浪费 | 严重（实际长度远小于 max） | 极小（最多浪费 1 个 Block） |
| 碎片 | 外部碎片严重 | 无外部碎片 |
| 共享 | 不支持 | 支持（引用计数） |

## Block 管理流程

```
1. 序列创建 → BlockManager 分配初始 Block
2. Prefill → 按 Block 填充 KV Cache
3. Block 满了 → 分配新 Block
4. 序列结束 → 引用计数减 1，归零则回收 Block
5. 共享前缀 → 多个序列指向同一 Block（引用计数 > 1）
```

## Block Table

```
逻辑序列: [tok0, tok1, ..., tok15 | tok16, ..., tok31 | tok32, ..., tok47]
             Block 0                  Block 1              Block 2
                ↓                        ↓                    ↓
物理 Block:  [GPU mem addr 0x...]  [GPU mem addr 0x...]  [GPU mem addr 0x...]
```

Block Table 是一个映射表：`sequence_id → [physical_block_ids]`

## 注意力计算适配

标准注意力假设 KV 是连续张量。Paged Attention 需要：
1. 根据 Block Table 收集物理 Block 地址
2. 在 Kernel 中按 Block 索引读取 KV
3. 使用自定义 CUDA Kernel 实现

## 学习资料

- [Nano-vLLM架构介绍](https://zhuanlan.zhihu.com/p/2010638958783131701)
- [KV Cache、PagedAttention和nano-vllm的实现](https://juejin.cn/post/7611851387790606374)
- [vLLM: PagedAttention 原理详解](https://vllm.hyper.ai/docs/performance/optimization/)

## Related

* [[KV Cache]]
* [[Continuous Batching]]
* [[CUDA Programming]]
* [[GPU Memory Hierarchy]]

## Tags

#ai-infra #paged-attention #memory-management #vllm
