# 版本 1.1：租户、智能体与技能边界

## 目标

在不改变运行时行为的前提下，将平台代码与租户特定业务配置及技能分离。此版本是一次仓库与配置边界清理，而非隔离或 IPC 重写。

## 非目标

- 不改变 `docker run` 行为。
- 不改变智能体服务或分组进程生命周期。
- 不迁移消息 IPC。
- 不引入 Linux 用户。
- 尚不移除已有 `groups/` 支持。

## 问题

当前扩展版 NanoClaw 部署混淆了以下关注点：

- 通用平台代码
- 租户特定业务技能
- 分组/智能体指令
- 运行时/会话数据
- 部署文件
- 密钥与服务特定配置

这使得部署以及后续的分组隔离难以理清。在智能体容器内创建按分组用户之前，我们需要对以下问题给出明确答案：

- 哪些文件是平台所有的？
- 哪些文件是租户管理的？
- 哪些文件属于某个智能体服务？
- 哪些文件在运行时对分组可写？
- 哪些技能是共享的、租户特定的或智能体特定的？
- 哪些密钥是引用而非具象化值？

## 目标布局

平台仓库：

```text
nanoclaw/
  src/
  container/
  setup/
  skills/
    builtin/
  docs/
  schemas/
    tenant.schema.json
    agent.schema.json
    skill-manifest.schema.json
```

租户仓库：

```text
nanoclaw-tenants/
  tenants/
    acme/
      tenant.json
      skills/
        acme-erp/
          SKILL.md
          manifest.json
        acme-approval/
          SKILL.md
          manifest.json
      agents/
        finance/
          agent.json
          instructions.md
          skills/
        ops/
          agent.json
          instructions.md
          skills/
```

运行时状态保持在 Git 之外：

```text
data/
  runtime/
  sessions/
  ipc/
  logs/
```

## 配置模型

`tenant.json`：

```json
{
  "id": "acme",
  "name": "Acme Corp",
  "enabled": true,
  "metadata": {}
}
```

`agent.json`：

```json
{
  "id": "finance",
  "name": "Finance Bot",
  "tenant": "acme",
  "provider": "claude",
  "model": "sonnet",
  "instructions": "./instructions.md",
  "skills": [
    "builtin:welcome",
    "tenant:acme-erp",
    "agent:local-finance-skill"
  ],
  "envRefs": ["ANTHROPIC_API_KEY"],
  "mounts": [],
  "limits": {
    "concurrentTasks": 1
  }
}
```

技能清单：

```json
{
  "id": "acme-erp",
  "scope": "tenant",
  "version": "1.0.0",
  "entry": "SKILL.md",
  "allowedAgents": ["finance", "ops"]
}
```

## 加载器设计

添加 `TenantConfigLoader` 模块，负责：

- 读取 `NANOCLAW_TENANTS_DIR`
- 使用 schema 校验租户和智能体 JSON
- 解析技能引用
- 输出规范化的 `RegisteredTenant` 和 `RegisteredAgent` 对象
- 拒绝重复的租户 ID 以及同一租户内重复的智能体 ID
- 记录诊断信息，但不启动运行时进程

规范化结构：

```ts
interface RegisteredTenant {
  id: string;
  name: string;
  rootDir: string;
  enabled: boolean;
}

interface RegisteredAgent {
  id: string;
  tenantId: string;
  name: string;
  folder: string;
  provider: string;
  model?: string;
  instructionsPath: string;
  resolvedSkills: ResolvedSkill[];
  envRefs: string[];
  mounts: AgentMount[];
  limits: AgentLimits;
}
```

## 向后兼容

现有 `groups/` 和现有 DB 分组记录继续支持。引入遗留适配器：

```text
groups/<folder>/CLAUDE.md -> tenant: legacy, agent: <folder>
```

遗留模式不得阻塞新的租户加载器。在此阶段路由器可以同时解析两个来源。

## 密钥

租户仓库可以引用密钥，但不得包含密钥值。

允许：

```json
{ "envRefs": ["ANTHROPIC_API_KEY", "FEISHU_APP_SECRET"] }
```

不允许：

```json
{ "ANTHROPIC_API_KEY": "sk-..." }
```

密钥解析仍在宿主侧完成。

## 代码变更

- 添加 `src/tenants/types.ts`。
- 添加 `src/tenants/loader.ts`。
- 添加 `src/tenants/schema.ts` 或 `schemas/` 下的 JSON schema。
- 添加 `setup` 对 `NANOCLAW_TENANTS_DIR` 的支持。
- 添加已加载租户/智能体的诊断命令或启动日志输出。
- 保留现有分组注册路径。

## 迁移脚本

添加脚本：

```bash
npm run migrate:tenants -- --tenant acme --from-groups groups --to ../nanoclaw-tenants
```

该脚本应当：

- 创建 `tenant.json`
- 为每个现有分组创建一个 `agents/<group>/agent.json`
- 将 `CLAUDE.md` 复制为 `instructions.md`
- 将技能归类为 `agent`，除非显式映射
- 在修改文件之前输出 dry-run 报告

## 测试

- 加载器能接受合法的租户仓库。
- 加载器拒绝重复 ID。
- 加载器拒绝配置中疑似密钥的值。
- 遗留分组仍能加载并保持为分组，而非智能体。
- 智能体技能解析是确定性的。
- 缺失的技能引用产生可操作的诊断信息。

## 验收标准

- 一次部署可以同时加载至少一个租户仓库和一个遗留分组。
- 无需任何运行时行为变更即可通过现有测试。
- 新的租户配置文件足以描述 provider、模型、指令和技能。
- 平台仓库不再需要为新租户包含公司特定的技能内容。
