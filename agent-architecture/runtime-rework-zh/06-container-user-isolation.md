# 版本 1.6：容器内分组用户隔离

## 目标

在每个智能体服务容器内，让每个组以独立的 Linux 用户运行。通过目录所有权和权限控制，确保同一智能体服务处理的各组无法读写彼此的运行时状态。

## 非目标

- 这不等同于每组一个容器。
- 这不提供每组独立的网络命名空间。
- 这不能防御所有内核/容器逃逸。
- 这不将任务与其父组身份隔离。
- 这不将多个智能体服务合并到同一容器中。

## 用户模型

智能体服务容器内的用户名：

```text
ncg_<group>
```

规范化：

- 小写
- 仅允许 `[a-z0-9_]`
- 最大长度选取以适配 Linux 用户名限制
- 如有需要，添加防冲突后缀

示例：

```text
group feishu-main -> ncg_feishu_main
```

监督进程组：

```text
nc-supervisor
```

每个组用户加入一个私有组，在需要时通过共享 ACL 授予监督进程访问权限。

## 目录权限

一个智能体容器的运行时根目录：

```text
/runtime/
```

组运行时：

```text
/runtime/groups/feishu-main
  owner: ncg_feishu_main
  group: nc-supervisor
  mode: 0770
```

实时运行：

```text
/runtime/groups/feishu-main/live
  owner: ncg_feishu_main
  group: nc-supervisor
  mode: 0770
```

任务运行：

```text
/runtime/groups/feishu-main/runs/task-123
  owner: ncg_feishu_main
  group: nc-supervisor
  mode: 0770
```

租户共享技能是挂载到智能体容器中的配置资产：

```text
/workspace/tenants/acme/skills
  owner: root
  group: nc-agent-readers
  mode: 0750
```

组本地可写文件：

```text
/workspace/tenants/acme/agents/finance/groups/feishu-main
  owner: ncg_feishu_main
  group: nc-supervisor
  mode: 0770
```

## 进程启动

监督进程准备目录并启动：

```bash
setpriv \
  --reuid ncg_feishu_main \
  --regid ncg_feishu_main \
  --init-groups \
  --no-new-privs \
  node /app/agent-runner/dist/index.js ...
```

如果 `setpriv` 不可用，使用 `gosu` 或 `su-exec`。

## 用户创建

监督进程方法：

```text
users.ensure
```

必须：

- 根据已加载的配置和路由状态校验智能体与组 ID
- 组不存在时创建
- 用户不存在时创建
- 在 `/home/ncg_feishu_main` 下创建主目录
- 如不需要交互式 shell，将 shell 设为 `/usr/sbin/nologin`
- 确保目录所有权和权限位

## 密钥

不要将真实的共享密钥放入组用户可读的文件中。

推荐方案：

- 主机/监督进程持有密钥
- 组进程通过凭据代理调用 Provider
- 组进程只能看到 Provider 特定的占位符或有范围的运行时令牌

如果为了兼容性需要直接注入环境变量，仅注入到特定组进程，绝不注入到监督进程的全局环境或共享文件。

在将此运行时设为默认之前，须审计将 Provider 凭据传入容器的遗留 Docker 路径。最终的 `agent-container-users` 运行时应优先使用主机/监督进程凭据代理或有范围的按运行令牌。直接注入 Provider 环境变量仅作为兼容性兜底方案，且必须在配置或日志中明确标记。

## 附加挂载

NanoClaw 1.0 对每个组校验 `containerConfig.additionalMounts`，因为每个组拥有独立的 Docker 容器。在本运行时中，一个智能体服务容器可承载多个组，因此挂载范围必须显式声明：

- 智能体范围的挂载对该智能体服务处理的所有组可见
- 组特定挂载必须挂载到组所有路径下，或以阻止其他组用户读取的权限进行准备
- 外部挂载白名单的非主只读策略继续保持执行
- 被阻止的密钥模式继续保持阻止
- 主机 `.env` 和凭据文件继续保持遮蔽或未挂载

如果运行时无法在共享智能体容器内安全地应用组特定挂载，应拒绝该运行并返回可操作的错误，而非将挂载范围扩大到整个智能体服务。

## 隔离任务

隔离任务使用相同的组用户：

```text
live chat:      ncg_feishu_main
isolated task:  ncg_feishu_main
```

它们的区别在于：

- 运行时目录
- DB 文件
- 续接状态
- 生命周期

它们是上下文隔离的，而非权限隔离的。

## 资源限制

添加可选的按组或按运行限制：

```json
{
  "memoryMb": 1024,
  "pids": 256,
  "cpuShares": 512,
  "concurrentTasks": 1
}
```

实现选项：

- 如果容器权限足够，从监督进程使用 cgroup v2
- 进程级监控和终止作为兜底方案
- Docker 容器级限制作为智能体服务总量上限

## 测试

在智能体运行时容器内添加权限测试：

- `feishu-main` 用户无法读取其他组的 `state.db`。
- `feishu-main` 用户无法写入租户共享技能。
- `feishu-main` 用户可以写入自己的 `outbound.db`。
- 隔离任务可以写入自己的运行 DB。
- 监督进程可以读取所有运行状态。
- 运行时目录中不存在全局可写的 `0777/0666` 路径。
- 组特定的附加挂载对其他组用户不可读。
- Provider 凭据不存在于组可读的文件、DB 行和日志中。

## 验收标准

- 组以非 root 的按组独立用户在其智能体容器内运行。
- 跨组运行时读取因权限被拒绝而失败。
- 现有文件 IPC 兼容路径不是全局可写的。
- 主机/监督进程仍可投递消息和停止运行。
- 该安全模型被记录为：弱于每组一容器方案，但强于同用户组进程方案。
