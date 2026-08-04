# @personal-assistant/supervisor-framework

Reusable LangGraph supervisor–expert runtime for multi-agent packs.

Packs supply config, models, capability providers, and seeding; the framework compiles a supervisor graph that routes work to runtime agents, optionally with cron, skills, and a system (configuration) agent.

## Install

```sh
pnpm install
pnpm build
```

## Scripts

| Command | Description |
|---|---|
| `pnpm build` | Compile TypeScript to `dist/` |
| `pnpm check` | Typecheck package + tests |
| `pnpm test` / `pnpm test:unit` | Run Vitest unit tests |

## Architecture

```
src/
  capabilities/   Capability catalog contract (providers → tools)
  core/           Kernel: supervisor graph, agents, skills, policies, persistence
  framework/      Pack bootstrap, runtime, cron, system agent, defaults, logging
  index.ts        Public API
```

| Layer | Role |
|---|---|
| **Capabilities** | Agents declare `capabilityIds`; providers resolve tools from deps |
| **Core** | LangGraph compile (`createAssistant`), routing, sub-agent execution, skills |
| **Framework** | Pack bootstrap, hot-recompile runtime, cron kit, system agent, defaults |

## Quick start

Minimal pack bootstrap:

```typescript
import {
  bootstrapSupervisorSystem,
  buildDefaultRuntimeExecution,
  createCapabilityCatalog,
  NONE_CAPABILITY_PROVIDER,
} from "@personal-assistant/supervisor-framework";

const result = await bootstrapSupervisorSystem({
  config: {
    runtimeAgentsFilePath: "./data/runtime-agents.json",
    cronJobsFilePath: "./data/cron-jobs.json",
  },
  // Exactly one of buildCapabilityProviders or capabilityCatalog
  capabilityCatalog: createCapabilityCatalog([NONE_CAPABILITY_PROVIDER]),
  supervisorLlm, // ILLMConnector
  loadSupervisorPrompt: () => "Supervise requests.",
  seedAgents: async () => [/* RuntimeAgentDefinition[] */],
  buildRuntimeExecution: (_agents, _skills, ctx) =>
    buildDefaultRuntimeExecution(ctx.capabilityCatalog, {
      loadPromptByKey: () => "You are a helpful agent.",
    }),
  buildModels: () => ({ generic: chatModel }),
  buildCapabilityDeps: () => ({}),
});

await result.graph.invoke(/* … */);
```

For long-lived hosts that need soft recompile (agent file changes) and a stable checkpointer:

```typescript
import { createSupervisorRuntime } from "@personal-assistant/supervisor-framework";

const runtime = await createSupervisorRuntime(pack, {
  onBeforeRecompile: async (adapters) => { /* close MCP, etc. */ },
  onShutdownAdapters: async (adapters) => { /* cleanup */ },
  onRecompiled: (fingerprint) => { /* … */ },
});

runtime.getGraph();
await runtime.recompile(); // no-op if agent fingerprint unchanged
await runtime.shutdownAdapters();
```

## Pack contract (`SupervisorPackBootstrap`)

Required:

- `config` — paths (`runtimeAgentsFilePath`, `cronJobsFilePath`) and optional `allowDataWrites` / `messageHistoryMaxTokens`
- Exactly one of `buildCapabilityProviders` or `capabilityCatalog`
- `supervisorLlm`, `loadSupervisorPrompt`
- `seedAgents`
- `buildRuntimeExecution`, `buildModels`, `buildCapabilityDeps`

Common optional hooks:

| Hook | Purpose |
|---|---|
| `initializeDefaults` | Seed prompts/skills before repos load (skipped when `allowDataWrites: false`) |
| `setupAdapters` | Async adapters (e.g. MCP) closed over by providers |
| `systemAgent` | Opt-in configuration agent + system-config capabilities |
| `buildSkillCatalog` | Skills for agents (default: empty) |
| `createCronJobRepository` | Persist cron jobs (default: noop) |
| `createCheckpointer` | LangGraph checkpoint saver |
| `buildGraphHooks` | Reply UX, prompt logging, cron trigger resolver |
| `validatePersistedAgents` | Custom capability validation |

Helpers: `seedAgentsIfMissing`, `createDefaultContentSeeder`, `buildDefaultRuntimeExecution`, `resolveAgentTools`, default prompt/skill XML constants.

## Capabilities

Agents list `capabilityIds`. Providers expose tools when available:

```typescript
createCapabilityCatalog([
  {
    descriptor: { id: "search", description: "Web search" },
    isAvailable: (deps) => Boolean(deps.searchClient),
    resolveTools: (deps) => [/* StructuredToolInterface[] */],
  },
  NONE_CAPABILITY_PROVIDER, // id: "none"
]);
```

`validatePersistedAgentCapabilities` runs at bootstrap unless overridden. Non-grantable capabilities can use `reservedForAgentIds`.

## System agent

Set `systemAgent: {}` (or `{ modelKey, maxSteps }`) on the pack to enable the virtual **Configuration** agent (`SYSTEM_AGENT_ID`). Bootstrap wraps the agent repository and merges `system-config` / `system-config-read` providers for cron, skills, and runtime-agent CRUD tools (with destructive confirm tokens).

## Cron

| Export | Role |
|---|---|
| `createCronJobRepository` / `createCronJobRepositoryForConfig` | File-backed jobs |
| `setupCron` / `startCronBootstrap` | Schedule + launch |
| `createRuntimeCronService` / `reconcileRuntimeCron` | Live reconcile |
| `watchCronJobDefinitions` | Reload on file change |
| `createCronRunner` | Invoke graph for a job (continuation cap: `MAX_GRAPH_CONTINUATIONS`) |

Jobs can target a runtime agent or the supervise cron route. Pair with `watchRuntimeAgentDefinitions` + `runtime.recompile()` for agent-definition hot reload.

## Skills

Skill catalog + file loaders under `core/skills` (list/read/write XML skills, attachments, actions, prompt enrichment). Default skill XML for cron, runtime agents, skill management, and bootstrap is exported from `framework/defaults/content`.

## Logging

- `createFilePromptLogger` / `noopPromptLogging` — prompt logging hooks for graph compile
- `getLogger` / `setLogger` / `createConsoleLogger` / `createFileLogger` / `createCompositeLogger` — app logging

## Dependencies

Runtime: `@langchain/core`, `@langchain/langgraph`, `@langchain/langgraph-checkpoint`, `node-cron`, `zod`.

Peer: `@langchain/core`, `@langchain/langgraph`.
