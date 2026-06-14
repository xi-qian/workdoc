## 一、问题背景

本文围绕以下核心问题展开：

- 端侧 AI 是否会被云端吞噬？
    
- Perception（感知）在 AI 系统中的真实地位是什么？
    
- LLM 是否已经隐式具备 world model？
    
- JEPA 提出的 world model 路线解决了什么问题？
    
- 为什么 planning 强迫系统必须具备 state representation？
    
- 为什么 low-entropy state 是必要条件？
    

---

## 二、云端 vs 端侧：能力与分工

### 2.1 能力结论

- 云端模型在 **推理能力、规模、数据** 上长期占优
    
- 端侧模型难以在：
    
    - 长推理
        
    - agent planning
        
    - 多工具调用  
        上竞争
        

### 2.2 端侧不可替代的优势

端侧的核心优势不在“智能”，而在“系统位置”：

- 低延迟（real-time）
    
- 隐私数据访问
    
- 设备控制能力（OS / sensors）
    
- 离线运行
    

---

### 2.3 最可能的架构

```text
Cloud: reasoning / planning
Edge: perception / execution
```

---

## 三、Perception AI 的真实状态

### 3.1 已解决（closed-world）

- 人脸识别
    
- 语音识别
    
- 目标检测
    
- 姿态估计
    

特点：

```text
类别有限 + 场景稳定
```

---

### 3.2 未解决（open-world）

- 场景理解（scene understanding）
    
- 异常检测（long-tail）
    
- 动作与意图理解
    
- 物理直觉（physics intuition）
    

---

### 3.3 核心难点

```text
组合爆炸 + 数据稀缺 + 动态复杂性
```

---

## 四、LLM vs Perception vs JEPA

### 4.1 LLM 的本质

```text
latent → token（受语言约束）
```

特点：

- latent 存在
    
- 但被 token prediction 强约束
    

---

### 4.2 JEPA 的本质

```text
latent → latent（无 token 约束）
```

目标：

```text
学习 world state + dynamics
```

---

### 4.3 关键差异

|维度|LLM|JEPA|
|---|---|---|
|优化目标|token预测|state预测|
|表示类型|离散语义|连续状态|
|是否预测细节|必须|可忽略|
|适合任务|语言推理|世界建模|

---

## 五、JEPA 机制解析

### 5.1 基本结构

```text
context encoder → z_c
target encoder  → z_t
predictor: z_c → ẑ_t
```

训练目标：

```text
ẑ_t ≈ z_t
```

---

### 5.2 Joint 的含义

```text
joint embedding space（共享latent空间）
```

---

### 5.3 target encoder

- 结构相同
    
- 参数不同（EMA更新）
    
- 不参与梯度
    

作用：

```text
提供稳定训练目标，避免collapse
```

---

### 5.4 state 是否人为设计？

不是：

```text
state = learned latent representation
```

---

## 六、Action-conditioned World Model

### 6.1 核心公式

```text
state_{t+1} = f(state_t , action_t)
```

---

### 6.2 为什么 action 必须存在

否则：

```text
未来 = 多模态 → 平均化 → 无效预测
```

---

### 6.3 与 LLM 的差异

||LLM|World Model|
|---|---|---|
|输入|token|state + action|
|性质|被动预测|因果预测|
|类型|observer|agent|

---

## 七、Planning 为什么必须有 State

### 7.1 Planning 定义

```text
当前状态 → 模拟未来 → 选择路径
```

---

### 7.2 State 的定义

```text
state = 对未来预测充分的历史压缩
```

---

### 7.3 核心原因

Planning 需要：

```text
branching（多路径模拟）
```

必须在：

```text
compact representation 上进行
```

---

### 7.4 如果没有 state

系统只能：

```text
reactive policy（反应式）
```

无法：

```text
长期决策
```

---

## 八、Low-Entropy State 的必要性

### 8.1 定义

- high entropy：细节多、不确定
    
- low entropy：结构化、可预测
    

---

### 8.2 误差传播问题

多步 rollout：

```text
error → 指数增长
```

在 high entropy 空间：

```text
预测崩溃
```

---

### 8.3 low entropy 优势

- dynamics 简单
    
- 不确定性减少
    
- rollout 稳定
    

---

### 8.4 核心结论

```text
planning 成功 ⇔ state 低熵且可预测
```

---

## 九、LLM 的局限性（从 state 角度）

LLM 的 state：

```text
hidden state + token sequence
```

问题：

- 离散（不适合连续空间）
    
- 冗余（低效率）
    
- 不稳定（长推理漂移）
    

---

## 十、Latent 的本质作用

### 10.1 三个功能

```text
1. 信息压缩
2. 去噪
3. 提高可预测性
```

---

### 10.2 为什么现在必须有 latent

- 计算复杂度限制
    
- 优化困难（credit assignment）
    
- 数据效率问题
    

---

### 10.3 未来趋势

```text
latent 不会消失
但会：
从显式 → 隐式
从固定 → 动态
```

---

## 十一、三种路线统一

### LLM

```text
latent（隐式） + token约束
```

---

### JEPA

```text
latent（显式优化） + dynamics
```

---

### End-to-End（Bitter Lesson）

```text
latent（隐式存在） + action目标
```

---

## 十二、核心统一结论

### 12.1 关于 state

```text
planning 本质上就是在构造 state
```

---

### 12.2 关于 latent

```text
latent ≠ 是否存在
latent = 是否被显式约束
```

---

### 12.3 关于 world model

```text
world model = low entropy + predictable + sufficient state
```

---

## 十三、最终总结

1. 云端负责 **推理（reasoning）**，端侧负责 **感知与执行（perception + action）**
    
2. perception 不是简单 encoding，而是：
    
    ```text
    world state construction
    ```
    
3. LLM 已有 latent，但：
    
    ```text
    被 token 目标扭曲
    ```
    
4. JEPA 提供路径：
    
    ```text
    学习可预测的 world state
    ```
    
5. planning 强制要求：
    
    ```text
    state representation
    ```
    
6. state 必须满足：
    
    ```text
    low entropy + predictable
    ```
    

---

## 一句话核心结论

> **智能系统的本质，不是”会预测”，而是”在一个低熵、可预测的状态空间中进行多步未来推演”。**

## Related

* [[Enterprise Agent Service Model]]

## Tags

#ai-theory #edge-ai #jepa #world-model #perception