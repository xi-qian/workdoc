# LLM Serving Architecture

## Intuition

LLM 推理引擎的架构本质是一个事件循环：接收请求、调度执行、返回结果。核心设计决策在于如何组织调度器和显存管理器来最大化吞吐。

## Key Ideas

- 推理引擎 = 请求调度 + 显存管理 + 模型执行 + 采样
- nano-vLLM 是最小实现，vLLM / SGLang 是工业级实现
- 关键组件：Engine → Scheduler → BlockManager → ModelRunner
- 设计取舍：延迟 vs 吞吐、显存效率 vs 实现复杂度

## 架构全景

```
API Server (HTTP/gRPC)
    ↓ 请求入队
LLMEngine
    ├── Scheduler        ← 决定哪些请求参与本次迭代
    │   ├── waiting queue   ← 新请求等待区
    │   └── running batch   ← 正在生成的请求
    ├── BlockManager    ← 管理 KV Cache 的物理 Block
    │   ├── block table     ← 逻辑到物理的映射
    │   └── free list       ← 空闲 Block 池
    └── ModelRunner     ← 执行模型前向传播
        ├── model weights   ← 从 HBM 加载
        ├── attention       ← 用 Paged Attention Kernel
        └── sampler         ← 从 logits 采样 token
```

## 一次推理的完整流程

```
1. 用户发送请求 (prompt + params)
2. Tokenizer 将 prompt 转为 token ids
3. Scheduler 将请求加入 waiting queue
4. 调度决策：移出已完成请求，加入新请求
5. BlockManager 为新请求分配 KV Block
6. ModelRunner 执行 forward:
   a. Prefill: 处理全部 prompt tokens
   b. Decode: 逐个生成 token
7. Sampler 从 logits 采样 next token
8. 检查终止条件 (EOS / max_tokens)
9. 返回生成的 tokens 给用户
```

## vLLM vs nano-vLLM vs SGLang

| | nano-vLLM | vLLM | SGLang |
|--|-----------|------|--------|
| 定位 | 教学 | 生产 | 生产 |
| 代码量 | ~1.2k 行 | ~100k 行 | ~50k 行 |
| Chunked Prefill | 不支持 | 支持 | 支持 |
| Prefix Caching | 不支持 | 支持 | 支持 |
| Speculative Decoding | 不支持 | 支持 | 支持 |
| 学习难度 | 低 | 高 | 中 |

## 学习资料

- [Nano-vLLM架构介绍](https://zhuanlan.zhihu.com/p/2010638958783131701)
- [推理框架极简入门：用Nano-vLLM搭建知识体系](https://zhuanlan.zhihu.com/p/2008285806222132143)
- [nano-vLLM学习后记：如何设计一个轻量推理引擎](https://zhuanlan.zhihu.com/p/1977336847567983629)
- [小白想学LLM：nano-vllm项目的高层架构关系梳理](https://zhuanlan.zhihu.com/p/1930721878219133883)
- [LLM推理框架(vLLM/SGLang)入门Notebook练习](https://zhuanlan.zhihu.com/p/1999518738303693534)

## Related

* [[LLM Inference Basics]]
* [[KV Cache]]
* [[Paged Attention]]
* [[Continuous Batching]]
* [[Tensor Parallelism]]

## Tags

#ai-infra #serving #architecture #vllm
