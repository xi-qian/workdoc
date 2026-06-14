# 版本 1.9：技能运行时加载

## 目标

定义租户管理的技能和智能体管理的技能如何进入智能体服务容器，并可供分组运行使用，同时不使租户技能文件对分组用户可写。

运行时层级：

```text
tenant -> 部署/配置/技能管理层
agent  -> Docker 服务/容器运行时单元
group  -> 智能体容器内的 Linux 用户隔离单元
```

## 非目标

- 不使租户技能对分组用户可写。
- 不将租户作为运行时隔离单元。
- 不允许生成的技能覆盖租户仓库文件。
- 不要求每个 Provider 以相同方式内部加载技能。

## 技能来源

技能来源在部署/配置加载时解析：

```text
平台内置技能
租户管理的技能
智能体服务技能
分组生成的技能
```

推荐的来源布局：

```text
nanoclaw/
  skills/builtin/
    welcome/
      SKILL.md

nanoclaw-tenants/
  tenants/acme/
    skills/
      acme-approval/
        SKILL.md
        manifest.json
    agents/
      finance/
        agent.json
        instructions.md
        skills/
          finance-local/
            SKILL.md
```

运行时可写的生成技能：

```text
/runtime/groups/<group>/skills/generated/
```

生成的技能默认为分组本地。提升为租户管理的技能需要在分组运行时之外执行显式的审核/发布操作。

## 旧版可变运行器和技能状态

NanoClaw 1.0 有两个可变定制路径，必须显式处理：

- `container/agent-runner/src` 被复制到 `data/sessions/<group>/agent-runner-src` 并以可写方式挂载为 `/app/src`。
- 内置技能被复制到 `data/sessions/<group>/.claude/skills`，reporter/本地 API 可编辑该处的分组技能文件。

最终的智能体服务运行时不应以可写方式挂载平台运行器源码。将运行器/监督进程/Provider/MCP 代码视为镜像所有且只读。分组定制应使用：

- 分组运行时或配置的分组工作区下的分组记忆文件
- `/runtime/groups/<group>/skills/generated` 下的分组生成技能
- 提升后以只读方式挂载的已审核租户或智能体技能

如果暂时保留可写运行器源码，必须放在显式的兼容性标志之后，并存储在其他分组用户无法读写的分组拥有的运行时路径中。

## 智能体配置

`agent.json` 声明智能体服务启用的技能：

```json
{
  "id": "finance",
  "tenant": "acme",
  "skills": [
    "builtin:welcome",
    "tenant:acme-approval",
    "tenant:acme-erp",
    "agent:finance-local"
  ]
}
```

配置加载器将这些解析为智能体服务的具体技能清单。

## 解析后的技能清单

部署/配置加载器写入或传递标准化清单：

```json
{
  "tenant": "acme",
  "agent": "finance",
  "revision": "sha256:...",
  "skills": [
    {
      "id": "welcome",
      "scope": "builtin",
      "source": "/opt/nanoclaw/skills/builtin/welcome",
      "containerPath": "/workspace/skills/builtin/welcome",
      "entry": "SKILL.md",
      "readonly": true
    },
    {
      "id": "acme-approval",
      "scope": "tenant",
      "source": "/srv/nanoclaw-tenants/tenants/acme/skills/acme-approval",
      "containerPath": "/workspace/skills/tenant/acme/acme-approval",
      "entry": "SKILL.md",
      "readonly": true
    }
  ],
  "generatedSkillRoot": "/runtime/groups/${group}/skills/generated"
}
```

清单的作用域为智能体服务。特定于分组的附加内容可在运行开始时追加，但租户和智能体技能保持只读。

## 容器挂载

启动 `nanoclaw-agent-finance` 时，宿主机以只读方式挂载已解析的技能根目录：

```bash
docker run -d \
  --name nanoclaw-agent-finance \
  -v /srv/nanoclaw/platform/skills/builtin:/workspace/skills/builtin:ro \
  -v /srv/nanoclaw-tenants/tenants/acme/skills:/workspace/skills/tenant/acme:ro \
  -v /srv/nanoclaw-tenants/tenants/acme/agents/finance:/workspace/agent:ro \
  -v /var/lib/nanoclaw/runtime/agents/finance:/runtime \
  nanoclaw-agent-runtime
```

智能体容器内部：

```text
/workspace/skills/builtin/...
/workspace/skills/tenant/acme/...
/workspace/skills/agent/finance/...
/workspace/agent/skills.manifest.json
/runtime/groups/<group>/skills/generated/...
```

## 权限

只读技能路径：

```text
/workspace/skills/builtin
  owner: root
  mode: 0555

/workspace/skills/tenant/acme
  owner: root
  group: nc-agent-readers
  mode: 0750

/workspace/skills/agent/finance
  owner: root
  group: nc-agent-readers
  mode: 0750
```

分组生成技能：

```text
/runtime/groups/feishu-main/skills/generated
  owner: ncg_feishu_main
  group: nc-supervisor
  mode: 0770
```

分组用户可以读取已解析的租户和智能体技能，但不能修改它们。

## 运行时加载流程

```text
1. 配置加载器读取租户仓库
2. 配置加载器解析智能体服务技能引用
3. 运行时驱动启动/更新智能体服务容器，附带只读技能挂载
4. 监督进程启动分组运行
5. agent-runner 读取 skills.manifest.json
6. agent-runner 添加分组生成技能根目录
7. Provider 特定的适配器加载技能
```

## Provider 特定的适配器

Provider 差异必须封装在技能加载适配器之后：

```ts
interface SkillLoaderAdapter {
  prepare(input: SkillLoadInput): Promise<SkillLoadResult>;
}

interface SkillLoadInput {
  provider: string;
  agentId: string;
  groupId: string;
  manifestPath: string;
  generatedSkillRoot: string;
  runtimeDir: string;
}
```

Claude 适配器可以将已解析的技能暂存到每个分组的 `.claude/skills` 目录中，同时保持租户源目录只读。OpenCode 适配器可以将同一清单转换为 OpenCode 的技能/资源加载器格式。

不要让 Provider 特定的机制泄漏到调度器/路由器代码中。

## 重新加载策略

租户或智能体技能变更需要以下操作之一：

- 重启智能体服务容器
- 监督进程 `skills.reload` 方法
- 使用新的清单版本启动新分组运行，同时旧运行完成

推荐的初始行为：

- 在技能/配置变更时重启智能体服务容器
- 将运行时动态重加载作为后续优化

如果挂载的清单版本与宿主机配置不匹配，监督进程可以拒绝启动新运行。

## 生成技能提升

提升流程：

```text
生成技能 -> 审核 -> 租户仓库 PR/提交 -> 配置重加载 -> 只读租户技能挂载
```

不要从运行时容器内部自动将生成的技能文件复制到租户仓库中。

## Reporter 和本地 API

读取或编辑分组技能和记忆的 reporter/本地 API 方法必须重新映射：

- 技能读取包含已解析的只读技能和分组生成技能，只读状态对调用者可见
- 技能编辑仅写入分组生成技能根目录
- 记忆编辑仅写入分组工作区/运行时记忆路径
- 租户和智能体技能仓库永远不会被运行时 API 调用修改

Provider 钩子创建的对话归档应保持分组本地，例如位于分组工作区或运行时目录下。

## 测试

- 配置加载器解析 `builtin:`、`tenant:` 和 `agent:` 技能引用。
- 缺失的技能引用产生诊断信息并在未显式允许时阻止部署。
- 智能体服务容器以只读方式挂载租户技能根目录。
- 分组用户可以读取租户技能 `SKILL.md`。
- 分组用户不能修改租户技能 `SKILL.md`。
- 分组用户可以在其运行时目录下创建生成技能。
- 其他分组用户不能读取生成的技能，除非显式共享。
- Claude 技能适配器准备预期的技能路径。
- OpenCode 技能适配器准备预期的技能路径。
- 检测到技能清单版本不匹配。
- 平台运行器源码在最终运行时中不可写。
- reporter 技能编辑创建或更新分组生成技能，而非租户技能。
- 现有分组 `.claude/skills` 编辑可以迁移，或通过回滚驱动显式保留。

## 验收标准

- 租户管理的技能作为只读智能体服务输入进入运行时。
- 分组运行可以使用已配置的技能，而无需对租户仓库的写入权限。
- 生成的技能与租户管理的技能分离。
- Provider 特定的技能加载通过适配器隔离。
- 重新加载语义明确且安全。
