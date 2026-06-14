# Agent Platform MOC

## Overview

AI Agent 平台的架构设计、隔离方案、协作模式与机制分析的知识索引。

## 架构与设计

* [[Multi Agent Isolation]] — NanoClaw 多 Agent 架构与数据隔离（容器级）
* [[System Layers]] — NanoClaw 系统分层视图（执行/存储/编排/通道/用户）
* [[Hermes Distributed Architecture]] — Hermes 分布式架构（Manager + Gateway + Daemon）
* [[Agent Collaboration Patterns]] — HiClaw 层级式 vs Clawith 对等网协作模式
* [[Enterprise Agent Service Model]] — 企业 Agent 角色/实例/共享机制概念模型

## 隔离与安全

* [[Group Isolation Design]] — Hermes 群组隔离设计（ContextVar 方案 vs per-group 进程方案）
* [[Sandbox Security Design]] — Hermes 沙箱安全机制（execute_code / Docker / 命令审批）

## 机制分析

* [[Group Chat Memory Mechanism]] — OpenClaw 群聊记忆隔离（Session Key / QMD Scope）
* [[Manager Multi User Memory]] — HiClaw Manager 多用户 Memory 分析
* [[Clawith Memory Analysis]] — Clawith 单 Agent 多用户记忆更新机制
* [[Pairing Mechanism]] — OpenClaw 配对机制与聊天类型
* [[Plugin Mechanism]] — OpenClaw 插件系统（Channel / Provider / Memory Plugin）

## 平台对比

* [[Multi Platform Comparison]] — Multica vs HiClaw vs Clawith 横向对比

## 企业平台设计

* [[Enterprise Platform Overview]] — 企业 Agent 平台全局架构
* [[Message Gateway Design]] — Message Gateway 子系统设计
* [[AI Gateway Design]] — AI Gateway 子系统设计（LLM/MCP 路由、鉴权）
* [[Memory Service Design]] — Memory Service 子系统设计
* [[Management Platform Design]] — 管理平台子系统设计
* [[Storage Architecture]] — 存储架构与数据流

## AI 理论

* [[Edge AI and World Model]] — 端侧 AI、Perception、JEPA 与 World Model 系统性分析

## Tags

#moc #agent-platform
