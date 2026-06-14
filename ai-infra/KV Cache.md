# KV Cache

## Intuition

自回归推理中，每生成一个新 token 都需要对所有历史 token 做注意力计算。KV Cache 把已计算的 Key/Value 缓存下来，避免重复计算。

## Key Ideas

- 缓存每层的 Key 和 Value 矩阵，避免每步重新计算
- KV Cache 占用显存随序列长度线性增长
- 管理不当会耗尽显存，是推理引擎要解决的核心问题
- [[Paged Attention]] 是 KV Cache 管理的关键创新

## 显存占用估算

```
KV Cache 显存 = 2 × num_layers × seq_len × hidden_dim × dtype_size × batch_size
```

示例：Llama-2-70B, seq_len=4096, batch_size=1, FP16:
- 2 × 80 × 4096 × 8192 × 2 = ~10 GB

## KV Cache 的生命周期

```
Prefill 阶段: 计算全部 prompt tokens 的 KV → 写入 Cache
Decode 阶段: 只计算新 token 的 KV → 追加到 Cache → 用全量 Cache 做注意力
```

## 管理挑战

| 挑战 | 说明 |
|------|------|
| 动态长度 | 生成长度不可预知，无法静态分配 |
| 碎片化 | 频繁分配/释放导致显存碎片 |
| 跨请求共享 | prefix caching 需要共享 KV 块 |

## nano-vLLM 中的实现

涉及四个核心组件：
1. **LLMEngine** — 顶层协调器
2. **Scheduler** — 管理序列调度
3. **BlockManager** — 负责内存块分配
4. **ModelRunner** — 执行模型推理

## 学习资料

- [小白想学LLM(2)：KV Cache的具体实现流程](https://zhuanlan.zhihu.com/p/1933282638145230745)
- [小白想学LLM(3)：KV Cache存储与调度机制](https://zhuanlan.zhihu.com/p/1933915115662612173)
- [KV Cache、PagedAttention和nano-vllm的实现](https://juejin.cn/post/7611851387790606374)
- [LLM推理入门指南②：深入解析KV缓存](https://docs.feishu.cn/article/wiki/K8WxwbKRDi22Y3kwjt7cK0fonVd)

## Related

* [[LLM Inference Basics]]
* [[Paged Attention]]
* [[Continuous Batching]]
* [[GPU Memory Hierarchy]]

## Tags

#ai-infra #kv-cache #memory-management
