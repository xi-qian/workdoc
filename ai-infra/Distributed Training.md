# Distributed Training

## Intuition

训练千亿参数模型就像抄写一本数万页的百科全书——一个人抄不完。数据并行是复印多份每人抄不同章节再汇总；张量并行是把每页拆成几列每人抄自己那几列；流水线并行是第一个人抄完第一章就传给下一个人，自己继续抄下一批。

## Key Ideas

- 所有优化都在"计算、通信、显存"不可能三角中取舍
- 数据并行（DP/DDP/FSDP）复制模型、切分数据
- 模型并行（TP/PP/SP）切分模型本身
- ZeRO 系列用通信换显存

## 3D 并行策略

| 策略 | 切分维度 | 通信频率 | 适用范围 |
|------|---------|---------|---------|
| 数据并行 (DP) | 数据 batch | 每步 AllReduce 梯度 | 任意规模 |
| 张量并行 (TP) | 层内权重矩阵 | 每层 2 次 AllReduce | 单机内（NVLink） |
| 流水线并行 (PP) | 层间 | 每微批次 1 次 P2P | 可跨机（IB） |
| 序列并行 (SP) | 序列长度 | 配合 TP 使用 | 单机内 |

## ZeRO 系列

| 级别 | 切分内容 | 显存节省 | 通信开销 |
|------|---------|---------|---------|
| ZeRO-1 | 优化器状态 | ~4x | 与 DDP 相同 |
| ZeRO-2 | + 梯度 | ~8x | 与 DDP 相同 |
| ZeRO-3 | + 参数 | ~N 倍 | ~2x（需 AllGather 参数） |

## 混合精度训练

| 格式 | 指数位 | 尾数位 | 特点 |
|------|-------|-------|------|
| FP32 | 8 | 23 | 基准精度 |
| FP16 | 5 | 10 | 需 Loss Scaling，容易溢出 |
| BF16 | 8 | 7 | 动态范围接近 FP32，无需 Loss Scaling |
| FP8 | 5/3 | 2/4 | Hopper 原生支持，用于训练和推理 |

## 通信拓扑

- **NVLink**：机内 GPU 互连，~900 GB/s（TP 的基础）
- **InfiniBand**：跨机互连，~400 Gb/s（PP/DP 跨机的基础）
- **PCIe Gen5**：~64 GB/s（不够快，TP 不应跨 PCIe）

## 检验标准

- 能手算 7B 模型 FP16 训练的显存需求：参数 ~14GB + 优化器 ~56GB
- 能设计 64 卡集群的 3D 并行拓扑（TP=8/PP=4/DP=2）
- 30 分钟内把单卡训练脚本改造成 DDP

## 学习资料

- [Megatron-LM Paper](https://arxiv.org/abs/2104.04473) — TP 与 PP 原理的里程碑
- [ZeRO Paper](https://arxiv.org/abs/1910.02054) — 显存优化核心方法
- [DeepSpeed 官方文档](https://www.deepspeed.ai/)
- [PyTorch DDP/FSDP 教程](https://pytorch.org/tutorials/)

## Related

* [[Tensor Parallelism]]
* [[GPU Memory Hierarchy]]
* [[CUDA Programming]]
* [[LLM Serving Architecture]]

## Tags

#ai-infra #distributed-training #zero #parallelism
