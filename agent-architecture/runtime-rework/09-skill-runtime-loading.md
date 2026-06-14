# Version 1.9: Skill Runtime Loading

## Goal

Define how tenant-managed and agent-managed skills enter an agent service container and become available to group runs without making tenant skill files writable by group users.

Runtime layers:

```text
tenant -> deployment/config/skill management layer
agent  -> Docker service/container runtime unit
group  -> Linux user isolation unit inside an agent container
```

## Non-goals

- Do not make tenant skills writable by group users.
- Do not treat tenant as the runtime isolation unit.
- Do not let generated skills overwrite tenant repository files.
- Do not require every provider to load skills the same way internally.

## Skill Sources

Skill sources are resolved at deployment/config load time:

```text
platform builtin skills
tenant-managed skills
agent-service skills
group-generated skills
```

Recommended source layout:

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

Runtime writable generated skills:

```text
/runtime/groups/<group>/skills/generated/
```

Generated skills are group-local by default. Promotion to tenant-managed skills requires an explicit review/publish action outside the group runtime.

## Legacy Mutable Runner and Skill State

NanoClaw 1.0 has two mutable customization paths that must be handled
explicitly:

- `container/agent-runner/src` is copied into `data/sessions/<group>/agent-runner-src` and mounted writable as `/app/src`.
- builtin skills are copied into `data/sessions/<group>/.claude/skills`, and reporter/local API can edit group skill files there.

The final agent-service runtime should not mount platform runner source writable.
Treat runner/supervisor/provider/MCP code as image-owned and read-only. Group
customization should use:

- group memory files under the group runtime or configured group workspace
- group-generated skills under `/runtime/groups/<group>/skills/generated`
- reviewed tenant or agent skills mounted read-only after promotion

If writable runner source is kept temporarily, it must be behind an explicit
compatibility flag and stored in a group-owned runtime path that other group
users cannot read or write.

## Agent Configuration

`agent.json` declares enabled skills for the agent service:

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

The config loader resolves these into a concrete skill manifest for the agent service.

## Resolved Skill Manifest

The deploy/config loader writes or passes a normalized manifest:

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

The manifest is agent-service scoped. Group-specific additions can be appended at run start, but tenant and agent skills remain read-only.

## Container Mounts

When starting `nanoclaw-agent-finance`, host mounts resolved skill roots read-only:

```bash
docker run -d \
  --name nanoclaw-agent-finance \
  -v /srv/nanoclaw/platform/skills/builtin:/workspace/skills/builtin:ro \
  -v /srv/nanoclaw-tenants/tenants/acme/skills:/workspace/skills/tenant/acme:ro \
  -v /srv/nanoclaw-tenants/tenants/acme/agents/finance:/workspace/agent:ro \
  -v /var/lib/nanoclaw/runtime/agents/finance:/runtime \
  nanoclaw-agent-runtime
```

Inside the agent container:

```text
/workspace/skills/builtin/...
/workspace/skills/tenant/acme/...
/workspace/skills/agent/finance/...
/workspace/agent/skills.manifest.json
/runtime/groups/<group>/skills/generated/...
```

## Permissions

Read-only skill paths:

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

Generated group skills:

```text
/runtime/groups/feishu-main/skills/generated
  owner: ncg_feishu_main
  group: nc-supervisor
  mode: 0770
```

Group users can read resolved tenant and agent skills, but cannot modify them.

## Runtime Loading Flow

```text
1. config loader reads tenant repo
2. config loader resolves agent service skill references
3. runtime driver starts/updates agent service container with read-only skill mounts
4. supervisor starts a group run
5. agent-runner reads skills.manifest.json
6. agent-runner adds group generated skill root
7. provider-specific adapter loads skills
```

## Provider-specific Adapters

Provider differences must stay behind a skill loading adapter:

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

Claude adapter may stage resolved skills into a per-group `.claude/skills` directory while preserving tenant source directories as read-only. OpenCode adapter may translate the same manifest into OpenCode's skill/resource loader format.

Do not let provider-specific mechanics leak into scheduler/router code.

## Reload Strategy

Tenant or agent skill changes require one of:

- restart the agent service container
- supervisor `skills.reload` method
- start new group runs with a new manifest revision while old runs finish

Recommended initial behavior:

- restart agent service container on skill/config changes
- keep run-time dynamic reload as a later optimization

Supervisor can reject starting new runs if the mounted manifest revision does not match host config.

## Generated Skill Promotion

Promotion flow:

```text
generated skill -> review -> tenant repo PR/commit -> config reload -> read-only tenant skill mount
```

Do not auto-copy generated skill files into the tenant repo from inside the runtime container.

## Reporter and Local API

Reporter/local API methods that read or edit group skills and memory must be
remapped:

- skill reads include resolved readonly skills and group-generated skills, with
  readonly state visible to callers
- skill edits write only to the group-generated skill root
- memory edits write only to the group workspace/runtime memory path
- tenant and agent skill repositories are never mutated by runtime API calls

Conversation archives created by provider hooks should remain group-local, for
example under the group workspace or runtime directory.

## Tests

- config loader resolves `builtin:`, `tenant:`, and `agent:` skill references.
- missing skill references produce diagnostics and block deployment unless explicitly allowed.
- agent service container mounts tenant skill roots read-only.
- group user can read tenant skill `SKILL.md`.
- group user cannot modify tenant skill `SKILL.md`.
- group user can create generated skill under its runtime directory.
- another group user cannot read generated skills unless explicitly shared.
- Claude skill adapter prepares expected skill paths.
- OpenCode skill adapter prepares expected skill paths.
- skill manifest revision mismatch is detected.
- platform runner source is not writable in the final runtime.
- reporter skill edits create or update group-generated skills, not tenant skills.
- existing group `.claude/skills` edits can be migrated or explicitly left with the rollback driver.

## Acceptance Criteria

- Tenant-managed skills enter runtime as read-only agent service inputs.
- Group runs can use configured skills without write access to tenant repositories.
- Generated skills are separated from tenant-managed skills.
- Provider-specific skill loading is isolated behind an adapter.
- Reload semantics are explicit and safe.
