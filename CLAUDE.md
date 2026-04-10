# Forge — Claude Code Instructions

## Project Overview

Forge automates a multi-stage AI agent workflow (research → design → review-design → implement → review-code → verify) with a TUI progress interface and configurable human-in-the-loop approval gates. Read `ARCHITECTURE.md` before making any implementation decisions — it documents all technology choices, rejected alternatives, and the rationale behind every design decision.

## Tech Stack

- **Language:** TypeScript (Node.js / Deno compatible)
- **Agent runtime:** `@mariozechner/pi-agent-core` — used inside each stage, NOT as an orchestrator
- **LLM providers:** `@mariozechner/pi-ai` — `getModel(provider, modelId)` for per-stage model config
- **TUI:** `@mariozechner/pi-tui` — `Text`, `Markdown`, `SelectList`, `Editor`, `showOverlay()`
- **Orchestration:** Hand-written TypeScript state machine in `src/runner.ts`
- **Config:** `workflow.yaml` — stage names, providers, models, checkpoint mode, timeout, retry

## Critical Rules

### pi-mono is upstream — never modify it
`pi-agent-core`, `pi-ai`, and `pi-tui` are npm dependencies. Extend via their public APIs only:
- New stages: implement `StagePlugin` in `src/stages/`
- TUI customization: compose existing components (`Text`, `Markdown`, `Box`, `SelectList`, `Editor`)
- Agent extension: use `AgentTool`, `beforeToolCall`, `afterToolCall`, `subscribe()`, `transformContext`

### One Agent per stage execution — inject AgentRunner, never instantiate directly
Each stage receives an `AgentRunner` via constructor injection. Do not call `new Agent(...)` inside stage plugins — that bypasses the testability interface and makes unit tests impossible without real LLM calls. The production `AgentRunnerImpl` wraps pi-agent-core; test doubles use `FakeAgentRunner`.

### Routing is code, not prompting
All stage transitions live in `nextStage(current, decision)` in `src/runner.ts`. Cycle-back decisions (approve / request_changes / reject / pass / fail) are parsed from structured agent output by `src/result-parser.ts`. Do not rely on LLM reasoning to decide which stage runs next.

### Checkpoint gates are blocking Promises — Editor-based, not SelectList
At `checkpoint: post` stages, show an `Editor` overlay and `await` a Promise that resolves when the user submits a `CheckpointCommand`. The runner loop suspends at the `await` — no polling, no timers. `parseCheckpointCommand()` in `src/tui/command-parser.ts` parses the free-text input into structured decisions. `checkpoint: false` stages skip the gate entirely (auto-cycle).

### Never use `shell: true` or `exec()` for project commands
Commands from `.forge.yaml` are tokenised by `shell-quote` into `[executable, ...args]` and executed via `child_process.execFile(executable, args, ...)`. Never pass `shell: true` or call `child_process.exec()` — this prevents shell injection from attacker-controlled command strings.

### Run state is authoritative — always write atomically
Run state lives in `.forge/<runId>.json`. Write to `.forge/<runId>.tmp` then `fs.rename()` — never write directly to the JSON file. The concurrency lock is `.forge/<runId>.lock` created with `fs.openSync(path, 'wx')` (O_CREAT|O_EXCL, atomic).

## Code Style

- Follow Effective TypeScript: prefer `type` over `interface` for unions, `interface` for object shapes that will be implemented
- Strict mode (`"strict": true` in tsconfig)
- No `any` — use `unknown` and narrow
- Async/await throughout — no raw Promise chains
- Name files after what they export: `src/stages/research.ts` exports `researchStage`
- Format with Prettier (project config)

## Project Structure

```
src/
  main.ts               entry point
  config.ts             load + validate workflow.yaml
  runner.ts             state machine — nextStage(), runWorkflow()
  artifacts.ts          read/write files under artifacts/
  prompt-builder.ts     buildPrompt(stage, artifacts) → string
  result-parser.ts      parseResult(messages) → StageResult
  types.ts              Stage, Decision, StageResult, StagePlugin, ArtifactBag,
                        AgentRunner, CommandRunner, RunState, CheckpointCommand,
                        ReviewerOutput
  agent-runner.ts       AgentRunnerImpl — wraps pi-agent-core, honours AbortSignal
  command-runner.ts     CommandRunnerImpl — shell-quote tokeniser + execFile
  run-state.ts          RunStateStore — atomic read/write, lock, schema migration
  tui/
    index.ts            TUI assembly and layout
    progress.ts         ProgressPanel — Text component, stage list
    output.ts           OutputPanel — Markdown component, streaming
    checkpoint.ts       waitForCheckpoint() — Editor overlay, blocking Promise
    command-parser.ts   parseCheckpointCommand() — free-text → CheckpointCommand
  stages/
    research.ts
    design.ts
    review.ts           shared by review-design and review-code
    review-synthesizer.ts  dedup + quorum logic for multi-reviewer stages
    implement.ts
    verify.ts
.forge/                 runtime state (git-ignored)
  <runId>.json          run state — status, completedStages, artifactChecksums, etc.
  <runId>.lock          concurrency lock (O_CREAT|O_EXCL)
  trust.json            trust records — logical command-file names + SHA-256 hashes
```

## Adding a New Stage

1. Create `src/stages/<name>.ts` implementing `StagePlugin` from `src/types.ts`
2. Add an entry under `stages:` in `workflow.yaml`
3. Add the stage name to the `Stage` union type in `src/types.ts`
4. Add one `case` to `nextStage()` in `src/runner.ts`

All four changes are required. No other files need to change.

## TUI Patterns

### Streaming agent output to Markdown panel
```typescript
let buffer = ""
agent.subscribe((event) => {
  if (event.type === "agent_start") buffer = ""
  if (event.type === "message_update"
      && event.assistantMessageEvent.type === "text_delta") {
    buffer += event.assistantMessageEvent.delta
    outputMarkdown.setText(buffer)
    tui.requestRender()
  }
})
```

### Blocking checkpoint overlay (Editor-based)
```typescript
async function waitForCheckpoint(): Promise<CheckpointCommand> {
  return new Promise((resolve) => {
    const editor = new Editor(theme)
    const handle = tui.showOverlay(editor, { anchor: "bottom-center" })
    editor.onSubmit = (text) => {
      handle.hide()
      resolve(parseCheckpointCommand(text))
    }
  })
}
```

## Run State

Run state is persisted in `.forge/<runId>.json`. Key fields:

```typescript
interface RunState {
  schemaVersion: number          // increment on breaking changes; migrate_N_to_M() on load
  runId: string                  // e.g. "gh-owner-repo-42"
  status: "running" | "paused" | "failed" | "done"
  failureReason?: string         // set when status === "failed"
  completedStages: Stage[]
  currentStage: Stage | null
  artifactChecksums: Record<Stage, string>   // SHA-256 per stage; verified on resume
  implementBaseSha?: string      // HEAD SHA when implement branch was created
  wipCommitSha?: string          // SHA of WIP auto-commit before reject-to-design reset
  commandFileHashes: Record<string, string>  // full 64-char SHA-256 per logical command file
}
```

Use `RunStateStore` from `src/run-state.ts` for all reads and writes — never read/write the JSON file directly.

## Trust Model

Before executing any project commands from `.forge.yaml`, forge obtains explicit user consent and persists approval in `.forge/trust.json` (git-ignored). The trust record stores:

```json
{
  "make-cpp": {
    "grantedAt": "2026-04-10T10:00:00Z",
    "commandFileHash": "<64-char SHA-256 of the full command file content>"
  }
}
```

On each run, forge recomputes the SHA-256 and compares against the stored hash. Hash mismatch → re-prompt. The same hash is stored in run state under `commandFileHashes`. Use the `forge trust` subcommand to list, revoke, or inspect trust records.

## Git Branch Strategy

The implement stage operates on a dedicated `forge/<runId>` branch:
- **On first entry:** check for a clean working tree, create the branch, record HEAD SHA as `implementBaseSha`
- **On reject-to-design:** if tree is dirty, auto-commit (`git add -A && git commit`), store SHA as `wipCommitSha`, print recovery instructions, then `git reset --hard implementBaseSha`
- **On verify pass:** clear `wipCommitSha`, print completion message

The user's main branch is never checked out or modified by forge.

## What NOT to Do

- Do not modify anything under `node_modules/@mariozechner/`
- Do not use a single `Agent` as an orchestrator that spawns other agents via tool calls — this project uses a state machine for routing
- Do not add LangGraph, LangChain, or any other orchestration framework without updating ARCHITECTURE.md with full rationale
- Do not add a web server or web UI — the interface is TUI only
- Do not commit files under `artifacts/` or `.forge/` — they are runtime outputs
- Do not call `new Agent(...)` inside stage plugins — inject `AgentRunner` instead
- Do not use `child_process.exec()` or `execFile` with `shell: true` — always tokenise with `shell-quote` first

## Dependencies to Add

When implementing, the likely additional deps are:
- `js-yaml` — parse `workflow.yaml`
- `zod` — validate config schema
- `shell-quote` — POSIX-aware tokeniser for `run:` strings in command files
- `chalk` or `ansis` — ANSI colors for stage status in the progress panel (check if pi-tui already bundles an ANSI lib before adding)
