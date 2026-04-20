# 企业 Agent 服务模式

## Intuition

把 Agent 当作企业员工：角色是岗位，实例是员工入职后的独立工作空间，共享机制是内部知识流转。

## 核心概念

本文梳理四个基础概念：

* Agent 角色
* Agent 实例
* 多 Agent 工作模式
* 实例之间的隐式共享与显式共享

## Agent 角色

Agent 角色 = 岗位职责 + 必备内部知识的默认设定。

* 职责：定义 Agent 该做什么（如 PM Agent 需要知道项目管理工具是什么、如何获取项目信息）
* 内部知识：不内置整个知识库，只告诉 Agent 关键信息，让它自行查找
* 属于 [[Harness Engineering]] 的设计范畴

**共享规则**：同一角色的所有实例共享角色设定，单个实例不能随意修改公共设定。

## Agent 实例

Agent 实例 = 一个 Agent 角色服务特定用户时的实际运行进程。

* 可能是云端服务器进程，也可能是本地电脑进程
* 实例之间**严格隔离**：[[ChatSession]]、chat history、memory 独立读写，默认互相不可见
* 跨实例可见必须通过**显式共享机制**

### 当前claw体系的现状

* 除 [[Multi Agent Isolation|NanoClaw]] 外，所有 claw 实际都是**单实例模式**（一个角色只产生一个实例）
* 所有 claw 针对多个 group 维护多个 session，但只维护**一份 memory**，不按 user 区分
* 本质上所有 claw 都没有 user 层面的概念
* NanoClaw 方案：一个 group 一个实例，适合 IDO 以 group 为项目单位的场景

## 多 Agent 工作模式

多 Agent 协作与并行程序设计模式本质相同，分两类：

### 显式编排

* 一个主 Agent 生成任务图（DAG）
* 按依赖关系创建子 Agent 实例，并行或串行执行
* 执行顺序确定，可预测

典型实现：[[agent-collaboration-patterns|HiClaw]] 的 Manager → Team Leader → Worker 层级

### 事件驱动

* 所有 Agent 订阅事件总线，设定自己的触发事件
* 随事件输入被触发，消费事件并输出新事件
* 执行顺序不确定，无固定任务图

典型实现：[[agent-collaboration-patterns|Clawith]] 的 [[Trigger Daemon|Trigger Daemon]]

### 通讯基础设施

Agent 间通讯方案见 [[agent-collaboration-patterns]]。

Matrix 是 Agent + Agent + 人协作的良好方案，且支持跨组织通讯。

一种可行的完整方案：

**Matrix + 可视化配置管理 + 共享空间**

* Matrix：通讯层
* 可视化配置管理：角色设定与权限配置
* 共享空间：跨实例知识流转

> 类比：Moxt ≈ Slack + 自建共享文件系统；Truman 系统 ≈ Matrix + 可视化配置管理

### 实例与 Group 的关系

当一组 Agent 进入一个 group 工作时：

* 每个 Agent 角色为当前 group 创建一个实例
* 该实例与其他 group 的实例完全隔离
* **除非有显式共享**

## Agent 实例间的共享机制

### 隐式共享

同一角色的所有实例自动共享：

* 角色设定
* 工作技能（[[Plugin 机制|Plugin]] / MCP Server）

无需额外操作，零成本。

### 显式共享

实例主动将公共经验抽象后发布到共享空间：

* 需要明确的**发布**和**读取**动作
* 不一定需要用户操作（如每日定时反思，从对话中抽取公共经验自动发布）
* 关键在于：设计层面必须有发布/读取的语义

### 隐式 vs 显式的边界

* 所有功能都可以通过显式共享实现
* 隐式共享的目的是**降低使用成本，减少系统摩擦**
* 边界划分没有 100% 准确的方式，本质是**产品策略的权衡**

| 维度 | 隐式共享 | 显式共享 |
|------|---------|---------|
| 触发方式 | 自动 | 主动发布 |
| 典型内容 | 角色设定、技能 | 经验总结、知识沉淀 |
| 使用成本 | 零 | 需要读取动作 |
| 适用场景 | 变化频率低、所有实例都需要的 | 变化频率高、按需获取的 |

## Related

* [[Agent Collaboration Patterns]]
* [[Multi Agent Isolation]]
* [[Group Chat Memory Mechanism]]
* [[Group Isolation and Sandbox Design]]
* [[Plugin Mechanism]]

## Tags

#agent-architecture #enterprise-patterns #sharing-mechanism
