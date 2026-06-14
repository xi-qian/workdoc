# CUDA Programming

## Intuition

CUDA 是 NVIDIA GPU 的编程模型。理解 CUDA 是阅读推理引擎自定义 Kernel（如 Flash Attention、Paged Attention Kernel）的前提。

## Key Ideas

- GPU 由多个 SM（Streaming Multiprocessor）组成，每个 SM 运行多个线程
- 编程模型：Grid → Block → Thread 三级层次
- 关键概念：Kernel 启动、共享内存、线程同步、Warp
- 推理引擎用 CUDA 写自定义 Kernel 来优化 [[Paged Attention]] 和 Flash Attention

## CUDA 编程模型

```
Grid（整个 GPU）
├── Block 0（在 1 个 SM 上运行）
│   ├── Thread 0
│   ├── Thread 1
│   └── ... (最多 1024 threads/block)
├── Block 1
│   └── ...
└── Block N
```

## 关键概念

| 概念 | 说明 |
|------|------|
| Kernel | 在 GPU 上并行执行的函数 |
| SM | 流多处理器，GPU 的计算单元 |
| Warp | 32 个线程的执行单元，同一 Warp 内线程同步执行 |
| Shared Memory | 片上高速存储，Block 内线程共享 |
| Thread Synchronization | `__syncthreads()` 同步 Block 内线程 |

## 执行模型

```
Host (CPU):
  1. 分配 GPU 显存 (cudaMalloc)
  2. 拷贝数据到 GPU (cudaMemcpy H→D)
  3. 启动 Kernel (kernel<<<grid, block>>>)
  4. 拷贝结果回 CPU (cudaMemcpy D→H)
  5. 释放显存 (cudaFree)

Device (GPU):
  - Kernel 在所有 Block/Thread 上并行执行
```

## 在推理引擎中的应用

- **Flash Attention** — 分块计算注意力，减少 HBM 读写
- **Paged Attention Kernel** — 按 Block 索引读取 KV Cache
- **采样 Kernel** — 高效并行采样
- **量化 Kernel** — FP8/INT8 反量化

## 学习资料

- [CUDA编程入门极简教程](https://zhuanlan.zhihu.com/p/345661495) — 从硬件到代码
- [CUDA C++ Programming Guide](https://docs.nvidia.com/cuda/cuda-c-programming-guide/)
- [torch.compile 使用指南](https://zhuanlan.zhihu.com/p/670674773) — JIT 编译加速

## Related

* [[GPU Memory Hierarchy]]
* [[Paged Attention]]
* [[Tensor Parallelism]]

## Tags

#ai-infra #cuda #gpu #programming
