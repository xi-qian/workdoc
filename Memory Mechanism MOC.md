# Memory Mechanism MOC

## Overview

Agent 记忆系统的隔离、存储、检索机制分析索引。核心问题：多用户/多群组场景下，Agent 记忆如何隔离与共享。

## 记忆机制分析

* [[Group Chat Memory Mechanism]] — OpenClaw 的 Session Key 隔离，内置 SQLite 与 QMD 后端对比，群间不隔离的现状
* [[Manager Multi User Memory]] — HiClaw Manager 的文件系统 memory 方案，多用户场景下的 memory 断裂分析
* [[Clawith Memory Analysis]] — Clawith 的 memory.md 全局共享机制，写入链路追踪，用户维度区分方案

## 隔离与共享设计

* [[Group Isolation Design]] — Hermes 的群组隔离设计（双层记忆：共享只读 + 群组可读写）
* [[Enterprise Agent Service Model]] — 隐式共享 vs 显式共享的概念框架
* [[Multi Agent Isolation]] — NanoClaw 的容器级隔离（每个 group 一个容器）

## 企业级记忆服务

* [[Memory Service Design]] — 独立 Memory Service 的设计（实时 + 定时两种模式）
* [[Storage Architecture]] — PostgreSQL 记忆表 schema 与数据流

## 关键结论

| 平台 | 记忆隔离现状 | 解决方案 |
|------|------------|---------|
| OpenClaw | 群间不隔离 | 每 group 独立 agent 或 deny 规则 |
| HiClaw Manager | 多用户混合 | 需加 per-user 目录层 |
| Clawith | memory.md 全局共享 | 新增 users/ 分区（方案三） |
| Hermes | 按 profile 隔离 | 复用 Profile 机制做 group 隔离 |

## Tags

#moc #memory #isolation
