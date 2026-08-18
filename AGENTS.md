# AGENTS.md

## Purpose

PiQuest is an experimental game-development AI agent built on the Pi monorepo (currently upstream Pi v0.84.2). It layers game-development behavior on top of Pi instead of rewriting it. This file gives agents practical, repository-specific rules for working on PiQuest safely and consistently. Product goals live in `README.md`; this file is operational.

## Before Making Changes

- Inspect the repository before proposing or implementing architecture. Read the relevant files in full.
- Identify existing Pi functionality that already solves part of the problem before building anything new.
- Understand the relevant extension points (see Repository Map) before touching core code.
- Repository evidence takes priority over README goals and speculation.
- Keep changes scoped to the current task. Do not redesign unrelated Pi systems.

## Core Engineering Rules

### Reuse Pi Before Rebuilding

PiQuest evolves from Pi. Before creating PiQuest-specific infrastructure, check whether existing Pi mechanisms already cover the need:

- filesystem access and source editing: `read`, `write`, `edit`, `grep`, `find`, `ls` tools
- shell/process execution: `bash` tool, `src/core/bash-executor.ts`, `src/core/exec.ts`
- tools, context, agent loops, CLI behavior, testing: see Repository Map

Do not duplicate generic coding-agent functionality merely to give it a game-specific name. Extend or specialize only where actual game-development requirements justify it.

### Unity First, Generalize Later

Unity is the first concrete engine target.

- Prefer working Unity implementations over speculative cross-engine abstractions.
- Do not introduce Unreal, Godot, or generic engine layers without a concrete current requirement.
- Use real Unity tasks to determine which abstractions are useful. Generalization follows evidence.

### Game Project Is Not Just Source Code

A game project may involve source code, scenes, prefabs, serialized data, assets, input configuration, UI, project/editor settings, logs, runtime state, profiler data, builds, and observable output. New architecture must not assume source files are the only meaningful project state. This does not mean every capability must be implemented now.

### Prefer Workflows Over Specialized Agents

Do not create separate agents merely because a domain exists. Avoid premature structures such as `GameplayAgent`, `ArtAgent`, `DesignAgent`, `DebugAgent`, `OptimizationAgent`, `QAAgent`. First determine whether a specialized workflow, tool, context source, or validation method is sufficient. Introduce a separate agent only when experiments show separation provides a meaningful advantage.

### Prefer Small Experiments

PiQuest is experimental. When testing an idea:

1. start from a real game-development task;
2. identify the limitation of the current agent;
3. implement the smallest capability needed to test the hypothesis;
4. validate the result;
5. keep the design reversible.

Avoid creating frameworks before the underlying need is demonstrated.

### Validate Outcomes Appropriately

A successful source edit is not automatically a successful game-development task. Use the cheapest meaningful validation available:

```text
static/source validation
→ engine compile/import validation
→ runtime execution
→ controlled interaction
→ behavioral or visual verification
→ performance measurement
```

Do not require high-cost runtime verification for trivial changes. Do not stop at compilation when the outcome is inherently behavioral or runtime-dependent.

### Debug From Evidence

For bugs, prefer:

```text
reproduce → collect evidence → form hypothesis → inspect relevant systems
→ make targeted change → reproduce again → verify
```

Do not make broad speculative changes because a patch looks plausible. Unity failures may originate outside C# code.

### Measure Optimization

Performance work starts from evidence:

```text
baseline → measure → identify bottleneck → change → measure again
```

Do not describe speculative cleanup as optimization without relevant measurements.

### Keep Current Reality Separate From Direction

- Distinguish existing functionality from planned functionality in docs and comments.
- Do not describe experimental ideas as established architecture.
- Do not implement abstractions solely because they may be useful someday.
- Keep architecture easy to change when experiments contradict earlier assumptions.

## Architecture Guidelines

Three tiers, in decreasing order of mergeability with upstream.

### Upstream-owned packages

Keep `packages/agent`, `packages/ai`, `packages/tui`, `packages/client`, `packages/protocol`, `packages/server`, `packages/session-backends`, `packages/telemetry`, `packages/evals` as close to upstream Pi as reasonably possible. Do not introduce game-specific concepts into these packages unless there is a demonstrated architectural need.

### `packages/coding-agent`

This is the primary Pi integration surface. Prefer its public SDK, session APIs, extension APIs, tool factories, and presentation infrastructure. Avoid scattering PiQuest conditionals (`if (isPiQuest)`, `if (gameMode)`, `if (engine === "godot")`) through unrelated Pi files. Prefer explicit extension points and dependency injection.

### PiQuest-owned code

PiQuest-specific functionality belongs in a future `packages/piquest/` package: the `piquest` CLI entry point, branding and presentation, project/game context, engine detection, engine adapters, game-specific tools, and run/playtest/verification workflows.

- Keep it one coherent package. Split only when real architectural pressure appears.
- Do not fork generic TUI primitives; compose above `@earendil-works/pi-tui`.
- Do not duplicate the entire Pi interactive UI; make presentation configurable and supply a PiQuest presentation layer.
- If created, it must satisfy the same root checks as every other package and be added to root `package.json` workspaces and `tsconfig.json` `paths`.

### Upstream policy

"Do not modify Pi core merely because a feature is game-development-related. First prove the feature cannot cleanly live in PiQuest-owned code."

- Minimize edits to upstream-owned files. Keep PiQuest-specific code isolated.
- Avoid large cosmetic rewrites and unrelated formatting churn.
- Prefer small, reviewable integration patches. Preserve existing Pi behavior unless a PiQuest requirement explicitly changes it.
- Check whether an existing Pi extension point solves the problem before modifying core code.
- Small upstream-friendly refactors are acceptable to expose clean extension points. Keep them generic enough to be plausibly useful outside PiQuest, and document important deviations from upstream.

## Repository Map

### Primary integration surface: `packages/coding-agent`

- Programmatic SDK: `createAgentSession`, `createAgentSessionRuntime`, `createCodingTools` and per-tool factories in `src/core/sdk.ts`.
- Extension API (`src/core/extensions/types.ts`): `registerTool`, `registerCommand`, `registerShortcut`, `registerFlag`, `registerProvider`, `registerMessageRenderer`, `registerEntryRenderer`, `registerMarkdownTransformer`; UI hooks `setHeader`, `setFooter`, `setWidget`, `setStatus`, `setTheme`, `setTitle`, `setEditorComponent`; lifecycle/event hooks via `on(...)` (e.g. `agent_start`, `agent_end`, `agent_settled`, `tool_call`, `tool_result`, `session_start`).
- Identity: `package.json` `piConfig` (name, configDir) drives `APP_NAME`, `APP_TITLE`, `CONFIG_DIR_NAME`, env-var prefixes, and the terminal title (`src/config.ts`). A rebranded package automatically skips upstream-specific first-time-setup branding (`src/cli/startup-ui.ts`).
- Shipping resources: `package.json` `pi` manifest field lists extensions/skills/prompts/themes (`src/core/pi-manifest.ts`).
- Extensions load from `.pi/extensions/` (project), `~/.pi/agent/` (global), and packages; extension modules resolve bundled packages (`typebox`, `@earendil-works/pi-*`, `@earendil-works/pi-coding-agent`) via the loader's virtual modules.
- Built-in tools: `read`, `bash`, `edit`, `write`, `grep`, `find`, `ls` (`src/core/tools/`).
- CLI entry: `src/cli.ts` → `src/main.ts`; binary `pi` → `dist/cli.js`. Modes: interactive, print, rpc (`src/modes/`).
- Process execution: `src/core/bash-executor.ts`, `src/core/exec.ts`.

### Agent loop

`packages/agent/src/agent-loop.ts` (`agentLoop`, `runAgentLoop`). Session logic lives in `packages/coding-agent/src/core/agent-session.ts`.

### Project-local PiQuest config

`.pi/` holds project extensions (`.pi/extensions/`), prompts (`.pi/prompts/`), and skills (`.pi/skills/`). These are the natural place for PiQuest-specific behavior before `packages/piquest/` exists.

### Unity project

`BlockLab/` is a Unity 2D URP project (input system, URP packages). It is untracked; `Library/`, `Logs/`, `Temp/`, `UserSettings/` are ignored by its own `.gitignore`. There is currently no Unity validation workflow in the repo. Future Unity work should use available validation mechanisms (compile, tests, manual runs) until dedicated support exists.

## Commands

- After code changes (not docs): `npm run check` (full output, no tail). Fix all errors, warnings, and infos before committing. Runs biome (with `--write`), pinned-dep/import/shrinkwrap/install-lock checks, `tsgo --noEmit`, and a browser smoke test. Does not run tests.
- Never run `npm run build` or `npm test` unless requested by the user.
- Never run the full vitest suite directly: it includes e2e tests that activate when endpoint/auth env vars are present. For all non-e2e tests, run `./test.sh` from the repo root. Otherwise run specific tests from the package root:
  - Vitest: `node "$(git rev-parse --show-toplevel)/node_modules/vitest/dist/cli.js" --run test/specific.test.ts`
  - `packages/tui` (`node:test`): `node --test test/specific.test.ts`
- If you create or modify a test file, run it and iterate on test or implementation until it passes.
- For `packages/coding-agent/test/suite/`, use `test/suite/harness.ts` + the faux provider. No real provider APIs, keys, or paid tokens.
- Put issue-specific regressions under `packages/coding-agent/test/suite/regressions/` named `<issue-number>-<short-slug>.test.ts`.
- For ad-hoc scripts, `write` them to a temp file (e.g. `/tmp`), run, edit if needed, remove when done. Don't embed multi-line scripts in `bash` commands.
- Never commit unless the user asks.

## Code Quality

- No `any` unless absolutely necessary.
- Inline single-line helpers that have only one call site.
- Check node_modules for external API types; don't guess.
- No inline imports (`await import()`, `import("pkg").Type`, dynamic type imports). Top-level imports only.
- Never remove or downgrade code to fix type errors from outdated deps; upgrade the dep instead.
- Use only erasable TypeScript syntax (Node strip-only mode) in code checked by the root config (`packages/*/src`, `packages/*/test`, `packages/coding-agent/examples`): no parameter properties, `enum`, `namespace`/`module`, `import =`, `export =`, or other constructs needing JS emit. Use explicit fields with constructor assignments.
- Always ask before removing functionality or code that appears intentional.
- Do not preserve backward compatibility unless the user asks for it.
- Never hardcode key checks (e.g. `matchesKey(keyData, "ctrl+x")`). Add defaults to `DEFAULT_EDITOR_KEYBINDINGS` or `DEFAULT_APP_KEYBINDINGS` so they stay configurable.
- Never modify `packages/ai/src/models.generated.ts` directly; update `packages/ai/scripts/generate-models.ts` instead, then regenerate. Including the resulting `models.generated.ts` diff is always OK, even if regeneration includes unrelated upstream model metadata changes.

## Dependency and Install Security

- Treat npm dep and lockfile changes as reviewed code. Direct external deps stay pinned to exact versions.
- When updating `undici`, you MUST read its changelog/release notes for the target version and evaluate whether any changes may affect functionality before applying the update.
- Hydrate/update locally with `npm install --ignore-scripts`; clean/CI-style with `npm ci --ignore-scripts`. Don't run lifecycle scripts unless the user asks.
- If dep metadata changes, refresh `package-lock.json` with `npm install --package-lock-only --ignore-scripts`.
- If `packages/coding-agent/npm-shrinkwrap.json` needs regen, run `node scripts/generate-coding-agent-shrinkwrap.mjs` (verify with `--check` or `npm run check`). New deps with lifecycle scripts require review and an explicit allowlist entry in that script; never add one silently.
- Pre-commit blocks lockfile commits unless `PI_ALLOW_LOCKFILE_CHANGE=1`. Don't bypass unless the user wants the lockfile change committed.

## Git

Multiple pi sessions may be running in this cwd at the same time, each modifying different files. Git operations that touch unstaged, staged, or untracked files outside your own changes will stomp on other sessions' work. Follow these rules:

Committing:

- Only commit files YOU changed in THIS session.
- Stage explicit paths (`git add <path1> <path2>`); never `git add -A` / `git add .`.
- Before committing, run `git status` and verify you are only staging your files.
- `packages/ai/src/models.generated.ts` may always be included alongside your files.
- Message format: `{feat,fix,docs}[(ai,tui,agent,coding-agent,piquest)]: <commit message> (optionally multiple lines)`. Message is informative and concise.

Never run (destroys other agents' work or bypasses checks):

- `git reset --hard`, `git checkout .`, `git clean -fd`, `git stash`, `git add -A`, `git add .`, `git commit --no-verify`.

If rebase conflicts occur:

- Resolve conflicts only in files you modified.
- If a conflict is in a file you did not modify, abort and ask the user.
- Never force push.

## Testing Interactive Mode with tmux

Run the TUI in a controlled terminal (from the repo root):

```bash
tmux new-session -d -s pi-test -x 80 -y 24
tmux send-keys -t pi-test "./pi-test.sh" Enter
sleep 3 && tmux capture-pane -t pi-test -p     # capture after startup
tmux send-keys -t pi-test "your prompt here" Enter
tmux send-keys -t pi-test Escape               # special keys (also C-o for ctrl+o, etc.)
tmux kill-session -t pi-test
```

## Changelog

Location: `packages/*/CHANGELOG.md` (one per package).

Sections under `## [Unreleased]`: `### Breaking Changes` (API changes requiring migration), `### Added`, `### Changed`, `### Fixed`, `### Removed`.

Rules:

- All new entries go under `## [Unreleased]`. Read the full section first and append to existing subsections; never duplicate them.
- Released version sections (e.g. `## [0.12.2]`) are immutable; never modify them.
- Do not create changelog entries when working on a branch other than `main`.

## Releasing

**Lockstep versioning**: all packages share one version; every release updates all together. `patch` = fixes + additions, `minor` = breaking changes.

The upstream release flow (`scripts/release.mjs`, local smoke via `npm run release:local`, CI publish/announcement in `.github/workflows/build-binaries.yml`) is inherited from upstream Pi and assumes upstream infrastructure. Before publishing this fork:

- Revalidate the release scripts, npm package scope, OIDC publish environment, and announcement flow for PiQuest's own publishing targets.
- Do not assume `pi.dev`, R2 markers, or upstream npm accounts apply to PiQuest.

## Scope and Non-Goals

Avoid, without evidence:

- rebuilding Pi capabilities unnecessarily;
- premature multi-agent architecture;
- premature multi-engine abstraction;
- universal project graphs without demonstrated value;
- infrastructure without a concrete game-development task;
- overengineering the first Unity experiment;
- treating compile success as universal proof of task success;
- speculative performance optimization;
- claiming roadmap features are already implemented.

Experimentation is welcome. The desired behavior: build the smallest useful experiment, observe the result, and let evidence shape the architecture.

## Conversational Style

- Keep answers short and concise.
- No emojis in commits, issues, PR comments, or code.
- No fluff or cheerful filler text. Technical prose only, be direct.
- Use concise, clear, simple language. Define unavoidable jargon before using it.
- Explain non-trivial designs and problems as: problem, concrete example or short trace, then solution. State why the solution is necessary and distinguish it from optional complexity.
- When the user asks a question, answer it first before making edits or running implementation commands.
- When responding to user feedback or an analysis, explicitly say whether you agree or disagree before saying what you changed.

## User Override

If the user's instructions conflict with any rule in this document, ask for explicit confirmation before overriding. Only then execute their instructions.
