# Version 1.1: Tenant, Agent, and Skill Boundary

## Goal

Separate platform code from tenant-specific business configuration and skills without changing runtime behavior. This version is a repository and configuration boundary cleanup, not an isolation or IPC rewrite.

## Non-goals

- Do not change `docker run` behavior.
- Do not change agent process lifecycle.
- Do not migrate message IPC.
- Do not introduce Linux users.
- Do not remove existing `groups/` support yet.

## Problem

The current extended NanoClaw deployment mixes these concerns:

- generic platform code
- tenant-specific business skills
- group/agent instructions
- runtime/session data
- deployment files
- secrets and service-specific configuration

That makes user isolation hard to reason about. Before creating per-agent users, we need a stable answer to:

- Which files are platform-owned?
- Which files are tenant-owned?
- Which files are agent-writable?
- Which skills are shared, tenant-specific, or agent-specific?
- Which secrets are references versus materialized values?

## Target Layout

Platform repository:

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

Tenant repository:

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

Runtime state remains outside Git:

```text
data/
  runtime/
  sessions/
  ipc/
  logs/
```

## Configuration Model

`tenant.json`:

```json
{
  "id": "acme",
  "name": "Acme Corp",
  "enabled": true,
  "metadata": {}
}
```

`agent.json`:

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

Skill manifest:

```json
{
  "id": "acme-erp",
  "scope": "tenant",
  "version": "1.0.0",
  "entry": "SKILL.md",
  "allowedAgents": ["finance", "ops"]
}
```

## Loader Design

Add a `TenantConfigLoader` module that:

- reads `NANOCLAW_TENANTS_DIR`
- validates tenant and agent JSON with schemas
- resolves skill references
- emits normalized `RegisteredTenant` and `RegisteredAgent` objects
- rejects duplicate tenant IDs and duplicate agent IDs inside a tenant
- records diagnostics without starting runtime processes

Normalized shape:

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

## Backward Compatibility

Existing `groups/` and existing DB group records remain supported. Introduce a legacy adapter:

```text
groups/<folder>/CLAUDE.md -> tenant: legacy, agent: <folder>
```

Legacy mode must not block the new tenant loader. The router can resolve both sources during this phase.

## Secrets

Tenant repos may reference secrets, but must not contain secret values.

Allowed:

```json
{ "envRefs": ["ANTHROPIC_API_KEY", "FEISHU_APP_SECRET"] }
```

Not allowed:

```json
{ "ANTHROPIC_API_KEY": "sk-..." }
```

Secret resolution remains host-side.

## Code Changes

- Add `src/tenants/types.ts`.
- Add `src/tenants/loader.ts`.
- Add `src/tenants/schema.ts` or JSON schemas under `schemas/`.
- Add `setup` support for `NANOCLAW_TENANTS_DIR`.
- Add diagnostics command or startup log output for loaded tenants/agents.
- Keep existing group registration path.

## Migration Script

Add a script:

```bash
npm run migrate:tenants -- --tenant acme --from-groups groups --to ../nanoclaw-tenants
```

It should:

- create `tenant.json`
- create one `agents/<group>/agent.json` per current group
- copy `CLAUDE.md` to `instructions.md`
- classify skills as `agent` unless explicitly mapped
- write a dry-run report before modifying files

## Tests

- Loader accepts valid tenant repo.
- Loader rejects duplicate IDs.
- Loader rejects secret-looking values in config.
- Legacy groups still load.
- Agent skill resolution is deterministic.
- Missing skill references produce actionable diagnostics.

## Acceptance Criteria

- A deployment can load at least one tenant repo and one legacy group simultaneously.
- No runtime behavior changes are required to pass existing tests.
- New tenant config files are sufficient to describe provider, model, instructions, and skills.
- Platform repository no longer needs company-specific skill content for new tenants.

