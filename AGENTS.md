# AGENTS.md — @justfortytwo/salience

Guidance for AI coding agents working in this repository.

## Purpose

`@justfortytwo/salience` is the salience-extraction engine for **fortytwo**. Given a
conversational turn, it asks an injected LLM to distil atomic, self-contained
candidate memories, each with a salience score in `[0,1]`. The write side
(`@justfortytwo/memory`'s enrichment loop) then dedupes, supersedes, and persists them.

It is a pure npm library: no Claude Code plugin, no marketplace entry, no provider SDK,
no credentials, and **zero runtime dependencies**.

## Layout

```
src/index.ts            # the whole library: types, prompt, parser, extractor, factory
test/extract.test.ts    # vitest suite (uses a fake LlmClient returning canned text)
.github/workflows/ci.yml# calls the shared justfortytwo/.github node-ci.yml workflow
tsconfig.json           # ES2022, NodeNext, strict, rootDir src -> outDir dist, emits .d.ts
dist/                   # build output (gitignored, the only code published)
```

`CLAUDE.md`, `.wolf/`, `.claude/`, `.codegraph/` are local tooling and are untracked.
Don't commit them unless the user asks.

## Commands (from package.json)

```bash
npm install          # devDeps only: typescript, vitest, @types/node (package-lock.json is committed)
npm run build        # tsc  -> dist/
npm test             # vitest run
npm run test:watch   # vitest
```

There is **no lint or format script** and no ESLint/Prettier config. Don't invent one;
`tsc` under `strict` is the only static check. `prepublishOnly` runs the build.

## Public API (all exported from `src/index.ts`)

- `Turn`: `{ text, source?, observed?, date?, meta? }`
- `Candidate`: `{ content, salience, source?, observed?, date?, tags?, meta? }`. It is
  intentionally shape-identical to memory's `EnrichmentCandidate`. Keep them in sync.
- `ExtractOptions`: `{ minSalience?, maxCandidates?, defaultObserved? }`
- `LlmClient`: `complete({ system, prompt }): Promise<string>`, the only model seam
- `SalienceExtractor`: `extractSalient(turn, opts?): Promise<Candidate[]>`
- `ModelSalienceExtractor` (reference impl), `createSalienceExtractor(llm)` (factory)
- `SALIENCE_SYSTEM_PROMPT`: exported `const`, passed as `system`

Changing any of these is a public API change: update `README.md` (Shape / Output
convention sections) and bump `version` in `package.json`.

## How extraction works

1. `ModelSalienceExtractor.extractSalient` calls `llm.complete({ system:
   SALIENCE_SYSTEM_PROMPT, prompt: turn.text })`. Only `turn.text` goes to the model.
   Provenance fields are not in the prompt.
2. `extractJsonArray` pulls a JSON array out of the raw text, tolerating code fences
   and surrounding prose.
3. Items are validated: `content` must be a non-empty string after trimming, and
   `salience` must be a finite number. Invalid items are skipped silently.
4. `salience` is clamped to `[0,1]`, then items below `opts.minSalience` are dropped.
5. Provenance is stamped. Model value wins, then the turn's value. For `observed`,
   `opts.defaultObserved` is the last fallback. `meta` is a merge of the turn's meta
   and the item's meta, with the item's keys winning. Empty `tags` are omitted.
6. Results are sorted by salience (highest first) and capped at `opts.maxCandidates`.

## Conventions and invariants

- **Never throw on bad model output.** Unparseable output returns `[]`. One bad turn
  must not crash an enrichment loop. Tests assert this fail-soft behaviour.
- **Provider-agnostic.** Never import Ollama/OpenAI/Anthropic or any other SDK, and
  never read credentials or env vars. The host injects `LlmClient`.
- **No runtime dependencies.** Keep `dependencies` absent.
- ESM only (`"type": "module"`), Node >= 18. With NodeNext resolution, relative imports
  need a `.js` extension, as in the tests: `from '../src/index.js'`.
- Optional fields are only set when they're defined (no `key: undefined`). Keep it
  that way so `toMatchObject`/`toEqual` comparisons and downstream spreads stay clean.
- Style: 2-space indent, single quotes, semicolons, and JSDoc on every exported symbol.
- Tests use `fakeLlm(reply)` stubs. Don't make network or model calls in tests.

## Relationship to sibling repos (`../`)

The sibling directories (`gate`, `memory`, `persona`, `runner`, `scheduler`,
`telegram`, `installer`, `marketplace`, `website`, `docs`) are separate fortytwo
repos/packages.

- **memory** depends on salience, never the reverse. `memory/package.json` lists
  `@justfortytwo/salience` as an **optional** `peerDependency` (`^0.1.0`).
- `memory/src/enrichment.ts` doesn't import salience yet. It has a `TODO(wire)` and
  **local mirror types** of `SalienceExtractor`/`Turn`. If you change the `Turn` or
  `Candidate` shape here, those mirrors and memory's `EnrichmentCandidate` will drift.
  Flag it, but don't edit other repos unless asked.
- CI notes that salience is a leaf package with no `@justfortytwo` siblings to link.

## Gotchas

- The package was earlier named `@justfortytwo/deepthought`, and old commits use that
  name. The current name is `salience`.
- "fortytwo" is the project name. `justfortytwo` is only the GitHub org and npm scope.
- `dist/` is gitignored. Run `npm run build` before testing consumers locally.

## fortytwo project context

This repository is part of **fortytwo**, a local-first personal-assistant spine built around existing agent runtimes and tool ecosystems.

The umbrella project is **fortytwo**. It is not intended to replace Claude Code, Codex, MCP servers, plugins, skills, or other agent runtimes. The project provides the durable personal-assistant infrastructure around them: memory, lifecycle, scheduling, channels, optional policy enforcement, and related supporting components.

Claude Code is currently the primary/reference runtime, but the architecture should avoid unnecessary coupling to a specific model provider. In particular, components should remain usable when Claude Code itself is configured against alternative compatible model providers.

The main bootstrap and lifecycle entry point is the **installer** repository (`justfortytwo/installer`).

### Canonical project locations

- Website: `forty-two.it`
- GitHub organization: `github.com/justfortytwo`
- Architecture/design documentation: `justfortytwo/docs`

### Repositories

The fortytwo project is intentionally split into small, focused repositories.

- **`justfortytwo/installer`**
  Main installer and lifecycle CLI (`create-fortytwo` / `fortytwo`). This is the primary bootstrap entry point for assembling a fortytwo installation.

- **`justfortytwo/runner`**
  Thin Claude Code process/session runtime. Owns process lifecycle and stream transport, including one-shot runs and persistent interactive sessions. It must not become an agent framework.

- **`justfortytwo/memory`**
  Durable semantic-memory MCP server backed by local storage and retrieval infrastructure.

- **`justfortytwo/scheduler`**
  Durable scheduling and proactive job execution. Owns *when* work should happen, not how the agent reasons about or performs that work.

- **`justfortytwo/telegram`**
  Telegram transport/channel adapter. Owns Telegram identity, pairing, message transport, attachment handling, and mapping chats to live agent sessions. It should delegate agent process lifecycle to `runner`.

- **`justfortytwo/persona`**
  Persona and context templates rendered by the installer into an individual fortytwo installation.

- **`justfortytwo/gate`**
  Optional external safety/policy enforcement layer for tool execution and approvals. Keep this separate from the agent runtime's own reasoning and permissions.

- **`justfortytwo/salience`**
  Optional model-driven salience extraction used to enrich durable memory.

- **`justfortytwo/marketplace`**
  Claude Code plugin marketplace and umbrella plugin used as a distribution surface for fortytwo components.

- **`justfortytwo/docs`**
  Cross-repository architecture, design, contracts, and project documentation.

- **`justfortytwo/website`**
  Public website for the project, served as `forty-two.it`.

- **`justfortytwo/.github`**
  GitHub organization metadata and shared organization-level project information.

### Cross-repository architecture

When changing one repository, treat the sibling repositories as parts of the same system.

The intended high-level ownership is:

```text
channels / scheduler
        |
        v
      runner
        |
        v
   agent runtime
  (Claude Code today)
        |
        +---- MCPs / plugins / skills / tools
        |
        +---- fortytwo memory

optional surrounding components:
- gate
- salience

bootstrap / distribution / documentation:
- installer
- persona
- marketplace
- docs
- website
```

A useful rule when deciding where code belongs:

> fortytwo should add continuity and infrastructure around an existing agent, not reimplement capabilities already owned by the agent runtime or its MCP/plugin ecosystem.

Examples:

- agent reasoning, planning, subagents, tools, MCP orchestration, and plugins belong to the agent runtime;
- Claude process/session lifecycle belongs to `runner`;
- durable memory belongs to `memory`;
- durable time and scheduled execution belong to `scheduler`;
- Telegram transport and Telegram identity belong to `telegram`;
- installation and lifecycle management belong to `installer`;
- browser automation should normally come from an existing MCP/plugin rather than a fortytwo-specific browser implementation.

Before introducing a new abstraction, check the relevant sibling repositories and the agent runtime's existing capabilities to avoid duplicating functionality elsewhere in the fortytwo stack.
