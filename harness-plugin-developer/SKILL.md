---
name: harness-plugin-developer
description: Expert workflow for building, debugging, and deploying DeepSeek Harness (dsh) plugins — tools, services, LLM adapters, and distributable bundles. Use when creating or modifying Harness plugin projects, writing cordis.yml / cordis.patch.yml config layers, or running dsh CLI commands to mount, install, or verify plugins. Contains source-verified schemas and code skeletons; runtime facts are verified against the local checkout when present.
metadata:
  short-description: Build DeepSeek Harness plugins
---

# DeepSeek Harness Plugin Development

## Role & Goal

You are a senior architect for the DeepSeek Harness plugin ecosystem. The Knowledge Base below is the **primary** source of truth; it was extracted from the real checkout at `<path-to-deepseek-harness>` and its official docs. If a fact is not in the KB, verify it in the local checkout with `rg` before using it; if the checkout is unavailable, rely on the KB alone and mark anything outside it as "needs verification". Never invent, and never write unverified findings back into this skill.

## allowed-tools

- File read/write/edit: create and modify plugin sources (`src/*.ts`), `cordis.yml`, `cordis.patch.yml`, `package.json`
- Terminal execution: `pnpm dsh ...`, `dsh ...`, `pnpm install`, `pnpm run build`, `tsc`, test runners
- Code search: `rg` inside `<path-to-deepseek-harness>` to verify APIs, config keys, and usage patterns

## Ground Rules (Anti-Hallucination)

1. Use only the Knowledge Base below as fact. If a detail is not covered, verify it in the local checkout first; if it cannot be found, say "needs verification" — never invent.
2. Local authority root (default): `<path-to-deepseek-harness>` (official repo [deepseek-ai/deepseek-harness](https://github.com/deepseek-ai/deepseek-harness)). If that path does not exist, ask the user for their checkout path and use it instead; without a checkout, verify nothing — mark out-of-KB facts as "needs verification".
3. Offline doc index (all paths relative to `<path-to-deepseek-harness>\`):
   - Getting started: `docs/user/develop/basic/index.md` (first plugin), `tool.md`, `config.md`, `publish.md`
   - Development guide: `docs/user/develop/framework/` (lifecycle, services, events), `docs/user/develop/practice/` (LLM adapter, capability layering)
   - Reference: `docs/cookbook/adding-a-tool.md` (full tool contract), `docs/config-catalog.md` (every settable config key), `docs/tool-catalog.md`, `docs/subsystems/` (built-in service APIs), `docs/testing.md`, `apps/cli/reference/README.md` (CLI behavior)
   - Core mechanism sources: `packages/bundle/base/cordis.patch.yml` (the 78 built-in rows), `packages/core/system-prompt`, `packages/compaction`, `packages/llm/token-meter`, `packages/spill`, `packages/session`, `packages/session-query`
   - Copyable minimal template: `scratch-plugin/` (`cordis.yml` + `src/my-plugin.ts`)
4. Reality-check table (wrong ideas to actively correct):

| Hallucinated form | Verified reality |
| --- | --- |
| `plugin.yaml` | Does not exist. The manifest is `package.json`'s `"dsh": { "bundle": { "patch": "..." } }` plus a `cordis.patch.yml` layer |
| A `harness` field in `package.json` | The real field is `dsh.bundle` (see bundle manifest below) |
| `harness plugins --patch ./my-plugin` | Hot-mounting is `dsh web --patch <overlay.yml>` — the `--patch` argument is a YAML overlay, not a plugin directory. Permanent install is `dsh plugin --profile <name> add <pkg>` |
| A `resources/` directory convention | No such convention. A dev plugin is "one TS module + one cordis.yml overlay"; a distributable bundle is "package.json + cordis.patch.yml + entry JS" |
| Override a row's `name` to swap implementations | `name` mismatch warns and skips the patch (`applyEntryPatches`). Disable the row and insert your own (§10) |
| Loading a second provider swaps a service at runtime | One implementation per context — a duplicate provider throws (see `SpillStore` docstring). Disable the old row first |
| `intercept` wraps or replaces a service implementation | It only merges per-entry service **config** (`Config.merge` or shallow assign); no wrapping (§3) |
| Every core mechanism can be swapped by plugin | Only the verified surfaces in §9; e.g. `agent-loop` is a concrete driver — swap the row, customize the surrounding rows |

## Knowledge Base Version Anchor

This KB is bound to a specific revision of the upstream checkout. Read this anchor before trusting any number below.

- **Pinned revision**: `<path-to-deepseek-harness>` at commit `99f6f02fecdb7dff40c3fbc9470f5907c29f74ca` (branch `master`), pulled 2026-08-18 09:09:43 +0800 — read from the plain-text files `.git/refs/heads/master` and `.git/logs/HEAD`. If the checkout has moved past this sha, re-verify affected facts before use.
- **Bound fact set**:
  - The 78 built-in rows and their ids in `packages/bundle/base/cordis.patch.yml` (§9, §9.5, §10).
  - The offline doc index in Ground Rule 3 (`docs/user/develop/*`, `docs/cookbook/*`, `docs/subsystems/*`, ...).
  - The verified interface signatures: system-prompt section/event surface, `CompactionEngine`, `TokenMeter`, `SpillStore`, `SessionPersistence`, `SessionQueryEngine`, `defineTool`, `LlmAdapter` (§5, §9, §11).
- **Upstream-change comparison** (run after the checkout is updated):
  1. Fetch upstream and fast-forward the checkout (an environment step, not a KB fact).
  2. Re-read `.git/refs/heads/<branch>` and `.git/logs/HEAD` for the new sha and date.
  3. Compare the key evidence: row count and ids in `packages/bundle/base/cordis.patch.yml` (was 78), `docs/user/develop/*`, and the signatures above.
  4. Differences → update the affected sections, re-verify each claim with `rg`, update the pinned sha/date here, then re-sync the three copies (authoritative → single-file → Codex install).
  5. No differences → update only the pinned sha/date.
- **Authority rule**: when the KB and the checkout disagree, **the checkout wins**. Record the difference, correct the KB, and never argue from the stale text.

## Quick Start (the 90% path)

Minimal local plugin (function form is the default; object and class forms are in the Knowledge Base):

```ts
// my-plugin.ts
import type { Context } from '@deepseek-ai/cordis'

export const name = 'my-plugin'

export function apply(ctx: Context) {
  console.log('[my-plugin] plugin loaded!')
}
```

```yaml
# cordis.yml overlay
- insert:
    - id: hello
      name: '/absolute/path/to/my-plugin.ts'   # local source paths MUST be absolute
```

```sh
# From a source checkout:
pnpm dsh web --patch ./cordis.yml             # Web UI: http://127.0.0.1:3080
# Installed CLI:
npx @deepseek-ai/dsh web --patch ./cordis.yml
```

Path rules: the `--patch` argument points at the overlay file itself (a path relative to the current directory is fine); inside the overlay, each entry's `name` (the module path) must be absolute.

Only add features after this loads and prints the log line. Every further capability (config, tool, service, adapter) is one incremental step on top of a running prototype.

## Development Workflow (do not skip)

### Phase 1 — Clarify only what blocks you

For a coding task, do not stall on ceremony: pick sensible defaults and state them. Ask the user **one round of numbered questions** only for genuinely blocking unknowns. If you can proceed with a default, the question is not blocking. Candidate questions with safe defaults:

- Plugin identity → derive from the project name (lowercase, `dsh-` prefix for bundles)
- Capability → single-file tool plugin unless stated otherwise
- Service dependencies (`inject`) → none unless the feature needs a known service (`tools`, `llm`, `shell`, ...)
- `Config` fields → sensible defaults for timeouts etc.; ask only when integration facts are unknowable (API base URL, API key, auth)
- Delivery form → local `--patch` debugging by default; bundle only when the user wants distribution
- Runtime → source checkout (`pnpm dsh`, loads `.ts` via the `tsx` hook) when `<path-to-deepseek-harness>` exists, else installed CLI (`npx @deepseek-ai/dsh`, bundles need built JS)

Proceed with defaults for anything unanswered; call the assumptions out in one line before writing code.

### Phase 2 — MVP prototype first

Ship the smallest mountable thing (Hello World or an empty `apply`), plus a minimal overlay and the run command. Get the load log on screen before adding any complexity.

### Phase 3 — Feedback loop (strict branching)

- **User is satisfied**: extend incrementally — one capability per step, keeping the prototype runnable after each step.
- **First negative feedback**: analyze the specific complaint and fix locally. Do not rewrite, do not expand scope.
- **Second negative feedback**: stop immediately; do not guess intent. Produce at least **5 concrete questions** (tech choices or business logic — e.g., persistence requirements, parameter schema shape, error-reporting needs, UI card needs, config placement, whether the current architecture is acceptable) for one-shot answers, and wait for them. **Only after the user answers**: judge by the answers plus current code size and architecture — a fundamental, architecture-level divergence means refactor/rewrite the project; a detail-level mismatch means patch in place.

### Phase 4 — Deploy and mount

1. Debug with hot-mounting first: `--patch` overlays, leveraging HMR for iteration.
2. When it works, ask the user's final preference:
   - **Option A (permanent)**: `dsh plugin --profile <name> add <pkg>` — writes into the profile's `dsh.profile.bundles`, active on every launch.
   - **Option B (on-demand)**: keep using `--patch` per session; no global pollution.
3. Before delivery: run `--dump-config` to confirm the patch layer applies in the right order; from a source checkout, run `pnpm run build` first.

---

## Knowledge Base

### 1. Core concepts

- A plugin is a TS/JS module exporting an `apply` function. The framework calls `apply(ctx, config)` and you register capabilities through `ctx`.
- The runtime is Cordis (`@deepseek-ai/cordis`). Each plugin instance owns a Fiber with the state machine `PENDING → LOADING → ACTIVE` (`FAILED` if `apply` throws), and `ACTIVE → UNLOADING → DISPOSED` on unload.
- **Auto-cleanup**: everything registered via `ctx.on`, `ctx.tools.register`, `ctx.llm.registerAdapter`, or `ctx.effect(() => cleanup)` is revoked automatically on unload — no manual `removeListener`/`clearInterval`.
- Dependency-driven loading: `apply` runs only after every `inject`ed service is ready. If a required service disappears, dependent plugins auto-dispose and reload when it returns.
- HMR: with `@deepseek-ai/cordis-plugin-hmr` loaded, editing a plugin source file — or the `config` of a `cordis.yml` row — unloads the old instance and loads the new one; no stale registrations survive.

### 2. Project layouts

Local dev overlay (the `scratch-plugin/` template):

```text
scratch-plugin/
├── cordis.yml        # overlay declaring the plugin row
└── src/
    └── my-plugin.ts  # module exporting name + apply
```

Distributable bundle (`docs/user/develop/basic/publish.md`):

```text
hello-plugin/
├── package.json       # declares dsh.bundle
├── cordis.patch.yml   # the layer applied when a profile lists this bundle
└── index.js           # the entry module referenced by the patch row
```

For a capability with swappable providers, split into three packages (capability layering, `docs/user/develop/practice/index.md`): **Service Definition** (`dsh-shell`: service + request/result types), **Service Provider** (`dsh-bash-local`: the implementation), **Consumer** (`dsh-tool-bash`: exposes it as a model-facing tool). Provider and Consumer each depend on the Definition, never on each other. Production reference: `packages/shell/tool-bash`.

### 3. Entry row fields and patch layer rules

A config file (`cordis.yml` / `cordis.patch.yml`) is a **top-level YAML array of patch objects** (verified in `vendor/include/src/index.ts`). A patch object is one of exactly two forms:

1. `insert` — a list of entries to append. With no `id`, entries go to the top level; with an `id`, they go into the group entry with that id (`config` of that group must be an entry list).
2. By-id override — `id` plus any entry fields to change (`name`, `config`, `disabled`, ...). If `name` is provided and does not match the target row's `name`, the patch is **skipped with a warning**; a patch matching no row also warns and is skipped.

There is no separate `override`/`unmount` operation in this version.

Entry fields (loader `EntryOptions` plus the isolation extensions in `vendor/loader/src/config/isolate.ts`): `id`, `name` (module specifier), `config`, `group`, `disabled`, `inject`, `intercept`, `isolate` (service-isolation realms, e.g. `isolate: { shell: true }` on a group row).

`intercept` semantics (verified in `vendor/loader/src/config/isolate.ts` + `vendor/cordis/lib/types/service.js`): per-entry **config** interception for services the entry consumes — not an implementation wrapper. The loader applies `entry.options.intercept` to the entry's context; when a service resolves its config, intercept entries merge over it (ancestors first; the service's `Config.merge` when declared, else a shallow assign). The programmatic equivalent is `ctx.intercept('llm', { temperature: 0.2 })`, which returns a child context carrying the intercept (see `docs/cordis-api/context.md`). An entry's `inject` may also use the object form, mapping services to intercept config.

```yaml
- id: scoped-consumer
  name: './src/consumer.ts'
  intercept:
    llm:
      temperature: 0.2      # merged into ctx.llm's resolved config for this entry's subtree only
```

```yaml
- insert:
    - id: hello                        # stable row id; later layers override by id
      name: '/absolute/path/my-plugin.ts'
      config:                          # optional; the WHOLE config value, validated by the plugin's exported Schema
        greeting: 'Hi there'
        maxRetries: 5
```

- Row semantics: `disabled: true` stops a row (and its descendants); `group: true` makes the row a nested group whose `config` is a child entry list.
- **Later layers win per row.** An override assigns each provided key onto the target row, and `config` is replaced **wholesale — no deep merge** — so when overriding a row, restate every key it needs.
- **An override cannot change a row's `name`**: a patch whose `name` does not match the target row's `name` warns and is skipped (`applyEntryPatches`). To swap a built-in implementation, disable the row and insert your own (recipe in §10).
- Layer order (empty root → composed): bundle patches listed in the profile's `dsh.profile.bundles` (in order, `@deepseek-ai/dsh-base` first) → the profile's own `cordis.patch.yml` → home-level `$DSH_HOME/cordis.patch.yml` → each `--patch <path>` overlay in argv order.
- `!!js` expressions in config values are evaluated by the loader against the row's injected services, e.g. `port: !!js ctx.webStartup.port ?? 3080`. They stay unevaluated in `--dump-config` output. When overriding a row's `config`, **keep the `!!js` expression** — replacing it with a literal removes the runtime read:

```yaml
# Earlier layer:
# - id: my-app
#   name: '@example/my-app'
#   config: { port: !!js ctx.webStartup.port ?? 8080, timeoutMs: 30000 }

# WRONG: the runtime port read is gone, and timeoutMs is silently dropped.
- id: my-app
  config:
    port: 9090

# RIGHT: restate the whole config, keeping the !!js expression.
- id: my-app
  config:
    port: !!js ctx.webStartup.port ?? 8080
    timeoutMs: 60000
```

- Service isolation: use a `group: true` row with `isolate: { shell: true }` so different plugin groups see different instances of a service (see `docs/user/develop/framework/service.md`).

### 4. Bundle manifest (package.json)

```json
{
  "name": "dsh-hello-plugin",
  "version": "0.1.0",
  "type": "module",
  "main": "index.js",
  "files": ["index.js", "cordis.patch.yml"],
  "dsh": { "bundle": { "patch": "./cordis.patch.yml" } }
}
```

The patch row references the package by name so Node resolution finds the installed code:

```yaml
- insert:
    - id: hello
      name: dsh-hello-plugin
```

- A **bundle** (declares `dsh.bundle`) contributes one config layer; a **profile** (declares `dsh.profile.bundles`) describes a launchable composition. A package without `dsh.bundle` is a plain library dependency (importable by plugins, activates no layer; `dsh plugin` warns once).
- File boundaries: the profile's `package.json` manifest is **maintained by `dsh plugin` — never hand-written**. User-editable: the profile's own `cordis.patch.yml`, and the profile's `pnpm-workspace.yaml` (the profile directory is itself a pnpm workspace root, so `allowBuilds` goes in `$DSH_HOME/profiles/<name>/pnpm-workspace.yaml`).
- Git installs (`dsh plugin add github:you/hello-plugin`) fetch **source, not build output**: the author must ship a self-contained `prepare` script that builds the published entry, and the user must authorize it by adding the exact package key pnpm prints to `allowBuilds` in the profile's `pnpm-workspace.yaml` (pnpm ≥10 refuses otherwise). **Supply-chain rule: authorizing `allowBuilds` means the package's code runs on the user's machine during install.** Review the `prepare` script, only authorize sources you trust, and lock git installs to a commit (`github:you/hello-plugin#<sha>`). Prefer npm publishing or a `pnpm pack` tarball to skip the build authorization entirely.

### 5. Code skeletons

#### 5.1 Plugin forms

Function (default for most plugins):

```ts
import type { Context } from '@deepseek-ai/cordis'

export const name = 'my-plugin'
export const inject = ['tools']   // optional

export function apply(ctx: Context) {
  // Add a second `config` parameter once you define `Config` (see §5.2).
  ctx.tools.register(/* ... */)
}
```

Object form:

```ts
export default {
  name: 'my-plugin',
  inject: ['tools'],
  apply(ctx: Context) { /* ... */ },
}
```

Class form (when the plugin provides a service to other plugins):

```ts
import { Service, type Context } from '@deepseek-ai/cordis'

declare module '@deepseek-ai/cordis' {
  interface Context { metrics: MetricsService }
}

export default class MetricsService extends Service {
  static inject = ['llm']              // a service may depend on other services
  constructor(ctx: Context) {
    super(ctx, 'metrics')              // service name; consumers inject ['metrics'] and read ctx.metrics
  }
  record(event: string, value: number) { /* ... */ }
}
```

Optional dependencies: omit from `inject` and query at the use site with `ctx.get('name')?.method()`.

#### 5.2 Plugin config (Schemastery)

```ts
import type { Context } from '@deepseek-ai/cordis'
import Schema from '@deepseek-ai/schemastery'

export interface Config {
  apiKey: string
  timeoutMs: number
  mode: 'fast' | 'accurate'
}

export const Config: Schema<Config> = Schema.object({
  apiKey: Schema.string().required(),
  timeoutMs: Schema.number().default(30000),
  mode: Schema.union(['fast', 'accurate']).default('fast'),
})

export function apply(ctx: Context, config: Config) {
  // validated and default-filled; invalid config fails the load loudly
}
```

House rules: any parameter that may differ between deployments is a config field (no hardcoded tunables); express complete constraints in the schema so bad config fails at load time. Never export a plain object as `Config` — Cordis requires the Schemastery (Standard Schema) shape.

#### 5.3 Registering a tool (defineTool)

```ts
import type { Context } from '@deepseek-ai/cordis'
import { defineTool } from '@deepseek-ai/dsh-tools'

export const name = 'greet-tool'
export const inject = ['tools']

export function apply(ctx: Context) {
  ctx.tools.register(defineTool({
    name: 'greet',                                  // model-visible
    description: 'Greet someone by name.',
    parameters: {
      name: { type: 'string', required: true, description: 'Name to greet' },
    },
    output: {
      schema: { type: 'string' },                   // canonical JSON value schema (object/array/scalar/null roots)
      render: (_args, value) => [{ type: 'text', text: value }],  // model-facing content
    },
    async execute(args, exec) {                     // args validated & typed from parameters; exec.signal aborts
      return `Hello, ${args.name}!`
    },
  }))
}
```

Tool contract (`docs/cookbook/adding-a-tool.md`):

- `execute` returns exactly one canonical JSON value matched by `output.schema`; throwing or returning an invalid value marks the call `isError`. Hand-check constraints the DSL cannot express (non-empty strings, positive numbers, cross-field rules).
- Do not mutate the definition after registration; to hot-swap a tool, dispose its owning effect and register a replacement. Treat `args` as readonly input.
- Optional UI cards: `presentCall(args)` / `presentResult(args, { content, isError, meta })` return card intents (`generic`, `terminal`, `diff`, `search`, `web`). Presenters MUST be pure functions of `args` + result — they also run on session-log replay, so no I/O, session state, clock, or randomness.

#### 5.4 Lifecycle and cleanup

```ts
export async function apply(ctx: Context) {
  ctx.on('some-event', handler)         // auto-unsubscribed
  ctx.effect(() => {
    const timer = setInterval(() => {}, 5000)
    return () => clearInterval(timer)   // runs on unload; ordered cleanup steps belong in ONE disposer
  })
  const fiber = ctx.plugin(childPlugin) // child fiber, unloads with the parent
  // Manual early teardown demo — normally just let the parent own the child.
  await fiber.dispose()                 // resolves after async cleanup completes
}
```

#### 5.5 Events

```ts
ctx.emit('my-plugin/ready', payload)                       // broadcast; return values ignored
const result = ctx.bail('some-check', input)               // first value that is not null/false/undefined wins; returning false or undefined lets the chain continue
await ctx.serial('setup-phase', context)                   // sequential async, same short-circuit
const out = await ctx.waterfall('my-plugin/transform', input, async () => input)  // pipeline

ctx.on('my-plugin/transform', async (_input, next) => {
  const downstream = await next()                          // waterfall listeners MUST call next(), else the pipeline short-circuits
  return downstream.trim()
})
```

Type safety via declaration merging: `declare module '@deepseek-ai/cordis' { interface Events { 'my-plugin/ready': (p: { id: string }) => void } }`.

#### 5.6 LLM adapter

```ts
import type { Context } from '@deepseek-ai/cordis'
import { LlmAdapter, type GenerateOptions, type StreamChunk } from '@deepseek-ai/dsh-llm'

// Adapter config — separate from the §5.2 example's Config; declare what this adapter needs.
export interface Config {
  apiKey: string
  providers: string[]
}

class MyAdapter extends LlmAdapter {
  async *stream(options: GenerateOptions): AsyncIterable<StreamChunk> {
    yield { type: 'block-start', index: 0, blockType: 'text' }
    yield { type: 'text-delta', index: 0, text: 'Hello' }
    yield { type: 'block-end', index: 0, block: { type: 'text', text: 'Hello' } }
    yield { type: 'usage', usage: { inputTokens: 100, outputTokens: 50 } }   // usage BEFORE finish
    yield { type: 'finish', reason: { kind: 'stop' } }                        // finish is always last
  }
}

export const inject = ['llm']
export function apply(ctx: Context, config: Config) {
  ctx.llm.registerAdapter(config.providers, new MyAdapter(config.apiKey))
}
```

Protocol rules: every `block-start` pairs with a `block-end`; `index` starts at 0; tool-call blocks use `tool-call-delta` with `id: CallId(...)`, `name`, and `argumentsDelta` (raw JSON text deltas). Unsupported fields must throw an `LlmError` with a stable code — never silently drop. Override `resolveModel(provider, model, signal?)` to return the exact provider/model identity and optional `context`/`reasoning` metadata.

### 6. TypeScript vs JavaScript at runtime

- **Source checkout** (`pnpm dsh ...`): the launcher script starts the CLI with a TS loader hook (`node --import tsx/esm`, see `apps/cli/reference/README.md` §Source execution), so `.ts` plugin paths load directly — this is why the `scratch-plugin` tutorial works with `.ts` files.
- **Installed CLI** (`dsh ...`): runs the built launcher with no TS hook. Bundles must ship compiled JS (`index.js`, `lib/`); a git-installed TS-only package has no build output and fails to load — hence the `prepare` script requirement in section 4.

### 7. Command cheat sheet

```sh
# Installed CLI (no checkout needed; Web UI at http://127.0.0.1:3080)
npx @deepseek-ai/dsh web

# From a source checkout (first time)
git clone https://github.com/deepseek-ai/deepseek-harness.git && cd deepseek-harness
pnpm install && pnpm run build
pnpm dsh web                                    # Web UI: http://127.0.0.1:3080

# Hot-mount debugging (--patch points at an overlay yml, not a plugin dir; repeatable)
pnpm dsh web --patch ./scratch-plugin/cordis.yml        # source checkout
npx @deepseek-ai/dsh web --patch ./extra.cordis.yml     # installed CLI; `web` is an alias for --profile web

# Bundle install and verification
dsh plugin --profile demo add ./hello-plugin            # forwards to pnpm; relative paths anchor to the cwd
dsh --profile demo --dump-config                        # should show a "# == dsh-hello-plugin" layer
dsh --profile demo
dsh plugin --profile demo remove dsh-hello-plugin
dsh plugin --profile demo add github:you/hello-plugin#<sha>   # needs prepare script + allowBuilds authorization
dsh plugin --profile demo add ./hello-plugin-0.1.0.tgz  # tarball: no build authorization needed

# Other
dsh --profile web --dump-default-config                 # bundle layers only
dsh --profile headless "run the tests"                  # one-shot task: result on stdout, exit 0/1
```

### 8. Built-in services (verified subset; full API in `docs/subsystems/`)

- `tools` — tool registry: `ctx.tools.register(defineTool(...))`; row `tools` = `@deepseek-ai/dsh-tools`
- `agents` — live agent coordination registry
- `llm` — adapter rows `llm-deepseek` / `llm-pi-ai`; `ctx.llm.registerAdapter` and default provider/model routing are mapped in §9
- `systemPrompt`, `compaction`, `tokenMeter`, `spillStore`, `sessionPersistence`, `sessionQuery`, `sessionProjections` — mechanism → row → source is mapped in §9
- Others: `shell`, `workspace`, `terminal`, `web`, `skills`, ... — consult the generated blocks in `docs/subsystems/*.md` (or the source interfaces) at use time. Facts not in this KB are verified in the checkout, never assumed.

### 9. Core pluggability map

Harness is "everything is a plugin": the core mechanisms are rows of the base bundle (`packages/bundle/base/cordis.patch.yml`, 78 rows) that expose Cordis services. Verified customization surfaces:

| Mechanism | Verified surface | Row (`id` → package) | Source |
| --- | --- | --- | --- |
| System prompt assembly | `ctx.systemPrompt`: register `PromptSection` (`order`: `-100` harness identity, `0` persona, `100–199` tool guidance), `PromptContext`, `variable`; a section with `complete: true` becomes the sole prompt section. Events: `system-prompt/assemble` (scoped waterfall, returned value authoritative), `system-prompt/change` | `system-prompt` → `@deepseek-ai/dsh-system-prompt` (config `persona`) | `packages/core/system-prompt/src/index.ts` |
| Workspace instructions (AGENTS.md) | Config-driven loader plugin; consumers read injected baseline instructions | `agent-instructions` → `@deepseek-ai/dsh-agent-instructions` (config `maxBytes: 65536`) | `packages/context/agent-instructions/src/index.ts` |
| Conversation compaction | `ctx.compaction`: subclass abstract `CompactionEngine` (`compactIfNeeded`, `compactNow`, `compactRegion`) | `compaction-basic` → `@deepseek-ai/dsh-compaction-basic` | `packages/compaction/compaction/src/index.ts` |
| Token metering | `ctx.tokenMeter` (`TokenMeter` service) | `token-meter` → `@deepseek-ai/dsh-token-meter` | `packages/llm/token-meter/src/index.ts` |
| Spill storage (large tool results) | `ctx.spillStore`: subclass abstract `SpillStore` (`saveText`); one implementation per context | `spill-local` → `@deepseek-ai/dsh-spill-local`; budget row `spill-policy` (`maxInlineBytes: 50000`) | `packages/spill/spill/src/index.ts` |
| Session persistence | `ctx.sessionPersistence` (`SessionPersistence` service) | `session-persistence-jsonl` → `@deepseek-ai/dsh-session-persistence-jsonl` (config `root: !!js dshHomePath('sessions')`); a sqlite backend exists in `packages/session/session-persistence-sqlite` | `packages/session/session-persistence/src/index.ts` |
| Session query / retrieval | `ctx.sessionQuery`: subclass abstract `SessionQueryEngine` (static `inject = ['sessions']`) | `session-query-sqlite` → `@deepseek-ai/dsh-session-query-sqlite` (config `path`, `openAt`) | `packages/session-query/session-query/src/index.ts` |
| Session projections | `ctx.sessionProjections`: `.register()` projection definitions | `session-projection` → `@deepseek-ai/dsh-session-projection` | `packages/session/session-projection/src/index.ts` |
| Model routing | `ctx.llm.registerAdapter`; per-agent default provider/model from the row below | `llm` → `@deepseek-ai/dsh-llm`; `agent-default-model` → `@deepseek-ai/dsh-agent-default-model` (config `provider`, `model`) | `packages/llm/*` |
| Agent loop driver | Concrete plugin that creates `ReactLoopAgent`s and publishes them through the agent/session registries; the practical customization surface is the surrounding rows (system prompt, instructions, default model, policies) | `agent-loop` → `@deepseek-ai/dsh-agent-loop` (config `agents: []`) | `packages/core/agent-loop/src/index.ts` |
| Durability / pruning policies | Checkpoint before each model request; oversized tool-result pruning before compaction | `session-checkpoint-policy`, `tool-result-pruner` (`thresholdChars/headChars/tailChars`) | `packages/session/session-checkpoint-policy`, `packages/compaction/compaction-tool-result-pruner` |

Only these verified surfaces are asserted here. If a mechanism is not in this table, verify its shape in the checkout before claiming it is pluggable.

#### 9.5 Impact analysis and design decisions

Replacing a core component never happens in isolation: every surface in §9 has consumers, event contracts, and state readers. Walk this matrix and checklist before choosing an approach in §10. All rows are source-verified against the checkout pinned in the Knowledge Base Version Anchor; anything derived beyond them must be marked "needs verification".

**Linkage matrix (verified)**

| Surface | What a replacement inherits (source-verified) | Source |
| --- | --- | --- |
| system-prompt | Assembly runs as the scoped waterfall `system-prompt/assemble`; the returned assembly is authoritative. `complete: true` makes a section the sole prompt; two effective complete sections throw. A replacement must keep the `ctx.systemPrompt` surface and emit `system-prompt/change` on updates. | `packages/core/system-prompt/src/index.ts` |
| agent-instructions | Baseline instructions enter the session as **user messages** (`source.kind === 'agent-instructions'`) before the first request — it does not register prompt sections, so it stacks with system-prompt at a different layer (session content vs system prompt). It listens to `session/event`, `agent/pre-step`, `tools/result`, reads the optional `ctx.fs`, and mounts as a no-op without an fs provider. | `packages/context/agent-instructions/src/index.ts` |
| compaction | `compaction-basic` injects `['llm', 'tokenMeter', 'sessions']` and its pressure decision reads `ctx.tokenMeter.measure(session)` on `agent/pre-step` — a token-meter replacement therefore changes compaction triggering. A run replaces a balanced surface span with one summary node and appends the replacement as session events, so persistence and session-query see the summary like any other event. It also listens to `agent/status`, `session/event`, `agent/request-error`; no spill usage was found in the package. | `packages/compaction/compaction-basic/src/index.ts`, `packages/compaction/compaction/src/index.ts` |
| token-meter | `measure(session)` is O(surface) and returns a detached measurement. It registers its context-breakdown projection through `ctx.inject(['sessionProjections'])` — replacing it changes both compaction triggers and projected breakdowns. | `packages/llm/token-meter/src/index.ts` |
| spill / spill-policy | spill-policy is a `tools/post-execute` transformer: results over `maxInlineBytes` are saved through `ctx.spillStore` and replaced in-session by a notice; the spilled text stays outside the session file. Best-effort by design: no session owner, no backend, or a save failure logs and returns the original result — a spill failure must never fail the tool. `spill-local` stores under `<root>/session-<hash>/…` (0600 files, 0700 root; config `root` or a lazily-created private root). | `packages/spill/spill-policy/src/index.ts`, `packages/spill/spill-local/src/index.ts` |
| session persistence | `session-persistence-jsonl` writes one append-only JSONL file per session (config `root`, `writeBatchMaxDelayMs`). `session-checkpoint-policy` injects `['llm', 'sessionPersistence', 'sessions', 'tools']` and checkpoints model requests (`llm/stream`) and top-level tool calls **before** dispatch — a checkpoint rejection prevents the request. A replacement must keep the `SessionPersistence` contract and leave `session/flush` consumers working. | `packages/session/session-persistence-jsonl/src/index.ts`, `packages/session/session-checkpoint-policy/src/index.ts` |
| session-query | `static inject = ['sessions']` — it reads live sessions, not a storage backend. Anything that stops appending events to sessions silently degrades search/read results without breaking loading. | `packages/session-query/session-query/src/index.ts` |
| session projections | `.register()` units are driven by `session/event`; a replacement projection must re-register on reload (HMR-safety, §14). token-meter contributes a projection through this registry. | `packages/session/session-projection/src/index.ts`, `packages/llm/token-meter/src/index.ts` |
| model routing | Adapters register via `ctx.llm.registerAdapter`; `agent-default-model` picks the per-agent provider/model. `compaction-basic` routes its summarizer to the same provider/model as the latest request (`routedTarget` reads the session request header) — routing changes propagate to compaction summarization. | `packages/llm/*`, `packages/compaction/compaction-basic/src/index.ts` |
| agent-loop | Concrete driver, not a service seam: it injects `['agents', 'sessions', 'llm', 'tools', 'systemPrompt']`, creates `ReactLoopAgent`s, publishes them through the registries, and opens a child context injecting `sessionPersistence`. Swap the row; customize the surrounding rows (§9). | `packages/core/agent-loop/src/index.ts` |
| checkpoint-policy / tool-result-pruner | Durability pair around requests and compaction: the checkpoint policy delays model/tool dispatch until the logged prefix is durable; the pruner deterministically trims oversized tool-result surface nodes (head/middle/tail) before compaction. Both are policy rows — override their config, or disable + replace if you need different semantics. | `packages/session/session-checkpoint-policy/src/index.ts`, `packages/compaction/compaction-tool-result-pruner/src/index.ts` |
| intercept / isolate | From §3: `intercept` only merges **config** (no wrapping); `isolate` gives a plugin group a private service instance. Neither substitutes for a replacement — use them to scope a replacement or tune it per group. | `vendor/loader/src/config/isolate.ts` |
| events | Verified core events to respect: `system-prompt/assemble` (scoped waterfall), `system-prompt/change`, `agent/pre-step`, `agent/status`, `agent/request-error`, `session/created`, `session/event`, `session/disposed`, `session/flush`, `tools/post-execute`, `tools/code-dispatch-log`, `llm/stream`. A replacement must keep emitting every event its consumers filter on. | `packages/core`, `packages/compaction`, `packages/session`, `packages/spill`, `packages/context` |

**Decision checklist**

1. **Justify the replacement.** Config-only needs → §10(b) `intercept`. A new capability seam → §10(c) new service name. Swap an implementation → §10(a) disable + insert, only when config and events cannot express the need.
2. **Consumer audit.** `rg "<service>"` for `inject` in the checkout. Consumers inject the service name, so they keep working after a swap (§1) — but only if your replacement honors the same surface.
3. **Event audit.** List what the old component emits and consumes (matrix above) and keep the emits; dropping an event breaks listeners silently — nothing fails at load time.
4. **State audit.** Find who reads the component's state (e.g., `compaction-basic` reads `ctx.tokenMeter.measure`; the checkpoint policy reads `sessionPersistence`); replacing one changes the other's behavior even if it still loads.
5. **Config layering.** Profile `cordis.patch.yml` → `$DSH_HOME/cordis.patch.yml` → `--patch`; `config` replaces wholesale (§3); keep one row per concern.
6. **Rollback plan.** Keep the replacement in its own overlay file: removing that overlay restores the built-in. Snapshot `--dump-config` before and after, and disable the old row rather than editing it.
7. **Default composition (when nothing needs replacing).** `compaction-basic` + `token-meter` + `spill-local` with `spill-policy` + `session-persistence-jsonl` + `session-query-sqlite` + `agent-loop`, customizing only the surrounding rows (system prompt, instructions, default model, policies). This is the combination every linkage row above assumes.
8. **Unverified derivations.** Any relationship not in the matrix (or not found in the checkout) must be marked "needs verification" before it is asserted in code or conversation.

### 10. Replacing built-in components: three approaches

**(a) Disable the built-in row, insert your own — the default for swapping an implementation**

- An override cannot change a row's `name` (warn + skip, §3), and loading a second provider of a service throws (Cordis standard duplicate-service behavior; see the `SpillStore` docstring) — so disable the old provider first.
- Consumers `inject` the service name, not the row, so dependents keep working and reload automatically (§1).

```yaml
# profile cordis.patch.yml or a --patch overlay
- id: compaction-basic
  disabled: true
- insert:
    - id: my-compaction
      name: '/absolute/path/src/my-compaction.ts'
      config:
        myThresholdTokens: 50000
```

**(b) Intercept config — per-entry tuning, no new implementation**

- `intercept` (or the object form of `inject`, or `ctx.intercept(name, config)`) merges config into a service for a subtree of entries (§3).
- Use it for deployment-level knobs (timeouts, budgets, temperatures) on an existing implementation. It cannot change behavior the service's config schema does not express.

**(c) New service provider + isolation**

- Provide your own implementation with the class form (§5.1). Either take over an existing service name (disable the old provider first, per (a)) or introduce a new service name that consumers opt into via `inject`.
- Scope it with `isolate` (§3) when only some plugin groups should see your implementation: `isolate: { compaction: true }` gives a group a private instance; a string label (`isolate: { compaction: 'tenant-a' }`) shares one realm across entries.

Tradeoffs: (a) full control, most code, replaces globally unless scoped via group/isolate; (b) cheapest, limited to config-exposed knobs; (c) cleanest seam for multi-tenant or deployment-specific behavior, requires the consumer surface to be inject-driven.

### 11. Walkthrough: custom compaction engine

Goal: replace `compaction-basic` with a custom engine (verified against `packages/compaction/*`).

1. Locate the row: `- id: compaction-basic, name: '@deepseek-ai/dsh-compaction-basic'` in `packages/bundle/base/cordis.patch.yml`.
2. Choose approach (a): disable + insert.
3. Implement by subclassing `CompactionEngine` (the service behind `ctx.compaction`). Minimal compiling stub (a real backend implements the strategy bodies):

```ts
// src/my-compaction.ts
import type { Context } from '@deepseek-ai/cordis'
import { CompactionEngine } from '@deepseek-ai/dsh-compaction'
import type { CompactionAgentContext, CompactionResult, CompactionTrigger, ManualCompactAgentContext } from '@deepseek-ai/dsh-compaction'
import type { CommandId } from '@deepseek-ai/dsh-commands/brand'

export default class MyCompactionEngine extends CompactionEngine {
  async compactIfNeeded(
    _agent: CompactionAgentContext,
    _trigger: CompactionTrigger,
    _signal: AbortSignal,
  ): Promise<CompactionResult | null> {
    return null // automatic pressure policy: implement here
  }

  async compactNow(
    _agent: ManualCompactAgentContext,
    _signal: AbortSignal,
    _sourceCommandId?: CommandId,
  ): Promise<CompactionResult | null> {
    return null // manual /compact: implement here
  }

  async compactRegion(
    _start: number,
    _end: number,
    _agent: CompactionAgentContext,
    _signal?: AbortSignal,
  ): Promise<CompactionResult> {
    throw new Error('not implemented') // span replacement: implement here; async rejects the returned promise instead of throwing synchronously
  }
}
```

Add a Schemastery `Config` and a `constructor(ctx, config)` (§5.2) when the engine needs tunables (see `compaction-basic`'s constructor shape).

4. Overlay: disable + insert as in §10(a).
5. Verify:
   - `dsh --profile web --dump-config` — the `compaction-basic` row is `disabled: true` and `my-compaction` appears after it (the dump order is the load order).
   - Run `/compact` as a smoke test — the `command-compact` consumer (`inject = ['commands', 'compaction']`) still resolves `ctx.compaction`.
   - Add the HMR-safety test (§14): dispose the engine's fiber and assert cleanup and dependent reload.
   - Boot logs are auxiliary only: a class-loading line (if your implementation logs one) confirms loading, but its absence is not a failure signal.

Contract notes (from `CompactionEngine` JSDoc): a run replaces a balanced surface span with one summary node; the replacement user message must use `compactCheckpointSource` with the transaction's `CompactionId`; forward cancellation signals; prevent concurrent compaction of the same session.

### 12. Core components working together

- Consumers inject **services, not rows**: `command-compact` injects `['commands', 'compaction']`; `compaction-basic` reads `ctx.tokenMeter`. Replacing a provider behind the same service name keeps consumers intact (§1).
- Compose through events (§5.5): `system-prompt/assemble` is a scoped waterfall — core prompt plugins contribute sections without knowing each other, and the returned assembly is authoritative.
- Layer order decides (§3): profile `cordis.patch.yml` → `$DSH_HOME/cordis.patch.yml` → `--patch` overlays. Keep one row per concern and override its `config` there instead of adding competing providers.
- Isolation boundaries: group + `isolate` gives separate service instances per plugin group (§3) — use it when two plugin sets must not share a core component instance (e.g., per-tenant compaction or spill backends).

### 13. Performance-oriented customization

Candidate hot paths — confirm each one on the user's composition before optimizing: system-prompt assembly, tool-schema assembly, workspace-instruction injection (`maxBytes` budget), compaction trigger checks (token-meter pressure), spill inline budget (`maxInlineBytes`), and persistence flushes/checkpoints (`session-checkpoint-policy`).

Method — no claims without measurements:

1. Baseline first: measure the current composition on a representative workload (per-turn wall-clock, `tokenMeter` context-pressure/breakdown projections, spill usage) before touching anything.
2. Change ONE component at a time; keep the rest of the composition identical.
3. Re-run the same workload and compare; snapshot both compositions with `--dump-config`.
4. Automate the workload as a vitest benchmark or a reproducible headless run (see §14 Testing).

This skill documents the method only. Never state performance gains ("X% faster", "uses less tokens") that were not measured on the user's own workload.

### 14. Testing

- Follow `docs/testing.md`. Unit tests use vitest and live in `tests/**` next to the code they exercise.
- Required convention for registries: an HMR-safety test that disposes the contributing fiber and asserts cleanup (every registration is revoked).
- Prefer edge cases, error paths, event ordering, and concurrency races; mock only the expensive boundary (LLM adapter, network, clock).

### 15. Common pitfalls

- Relative plugin paths in local overlays fail to resolve — use absolute paths.
- Patch overrides replace the whole `config`; restate every key or you silently drop the rest.
- Forgetting `next()` in a waterfall listener short-circuits the entire pipeline.
- Returning prose/content blocks from `execute` instead of the canonical value breaks the tool contract.
- Impure tool presenters (file I/O, session state, clock) crash session-log replay.
- TS-only bundles fail under the installed CLI — ship built JS, and provide a `prepare` script for git installs.
- Hardcoded tunables: deployment-varying parameters must be `Config` fields with Schemastery defaults.

### 16. Delivery checklist

- [ ] Plugin exports `name` (and required `inject`) plus `apply(ctx, config?)` — the `config` parameter only when the plugin takes configuration
- [ ] `Config` interface and the same-name Schemastery `Schema` match; no hardcoded tunables
- [ ] Overlay uses an absolute path, unique `id`, and config keys aligned with the schema
- [ ] `pnpm dsh web --patch ...` runs and prints the load log
- [ ] If bundling: `dsh.bundle.patch` points at the real `cordis.patch.yml`, entry files are in `files`, built JS is shipped
- [ ] `--dump-config` shows the layer in the right order
- [ ] Core customization: the built-in row `id`/`name` being replaced is confirmed against `packages/bundle/base/cordis.patch.yml`
- [ ] The old provider row is `disabled: true` before inserting a provider for the same service (duplicate providers throw)
- [ ] Unverified facts are marked "needs verification" (Ground Rules)
- [ ] User was asked: Option A (permanent install) or Option B (on-demand `--patch`)
