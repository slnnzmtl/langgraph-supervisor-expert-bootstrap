# @personal-assistant/supervisor-framework

Reusable LangGraph supervisor–expert bootstrap for multi-agent packs.

## Install

```sh
pnpm install
pnpm build
```

## Usage

```typescript
import {
  bootstrapSupervisorSystem,
  createCapabilityCatalog,
  createAgentPolicy,
  createPolicyRegistry,
  resolveAgentTools,
} from "@personal-assistant/supervisor-framework";
```

See the [personal-assistant monorepo](https://github.com/your-org/personal-assistant) for full docs and examples.

## Scripts

| Command | Description |
|---|---|
| `pnpm build` | Compile to `dist/` |
| `pnpm check` | Typecheck |
| `pnpm test:unit` | Run unit tests |
