# Continuous Batching

## Intuition

传统批处理等所有请求生成完毕才能处理下一批，导致 GPU 空闲。Continuous Batching 在任意迭代步都可以插入新请求、移除已完成请求，保持 GPU 始终满载。

## Key Ideas

- 每个迭代步动态决定哪些请求参与计算
- 已完成的请求立即让出资源，新请求立即填充
- 需要配合 [[KV Cache]] 管理和 [[Paged Attention]]
- Scheduler 是核心组件，决定每次迭代的 batch 组成

## Static Batching vs Continuous Batching

| | Static Batching | Continuous Batching |
|--|----------------|---------------------|
| 批次组成 | 固定，直到所有请求完成 | 每步可变 |
| GPU 利用率 | 低（等待最长请求） | 高（即时填充） |
| 延迟 | 受最慢请求拖累 | 每个请求独立 |
| 吞吐量 | 低 | 高（vLLM 提升 24x） |

## 调度流程

```
等待队列: [req_A, req_B, req_C, req_D]
Running:  [req_1, req_2, req_3]

每次迭代:
  1. 检查 running 中已完成的请求 → 移出，释放 Block
  2. 从等待队列取新请求 → 分配 Block，加入 running
  3. 对 running 中所有请求执行一步 forward
  4. 更新 KV Cache，检查终止条件
```

## Prefill 与 Decode 的混合调度

```
Iteration 1: Prefill(req_A, req_B)          ← 新请求加入
Iteration 2: Decode(req_A) + Prefill(req_C) ← req_B 完成，req_C 加入
Iteration 3: Decode(req_A, req_C)           ← 纯 Decode
Iteration 4: Decode(req_A, req_C) + Prefill(req_D)
...
```

nano-vLLM 的调度器不支持 Chunked Prefill，先处理所有 prefill 请求，再处理 decode 请求。

## 学习资料

- [管中窥豹nano-vllm（一）：从样例到主流程](https://zhuanlan.zhihu.com/p/1986582078280704324)
- [基于nano-vLLM学习大模型推理关键功能](https://www.cnblogs.com/cswuyg/p/19471225)
- [以Nano-vLLM为例深入理解LLM推理引擎](https://xie.infoq.cn/article/6ff774310b8e48a1f77b64561)

## Related

* [[KV Cache]]
* [[Paged Attention]]
* [[LLM Inference Basics]]
* [[LLM Serving Architecture]]

## Tags

#ai-infra #batching #scheduling #inference
