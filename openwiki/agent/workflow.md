# Agent workflow

The documentation agent is implemented in `src/agent/`. It takes a command (`chat`, `init`, or `update`), gathers repository context, builds prompts, runs a DeepAgents session, and records successful update metadata.

## Main flow

`src/agent/index.ts` follows this sequence for non-chat runs:

1. Load `~/.openwiki/.env` into `process.env`.
2. Ensure `OPENROUTER_API_KEY` exists.
3. Resolve the model ID from CLI input, environment variables, or the default.
4. Create a run context from Git state and prior update metadata.
5. Build the system prompt and user prompt.
6. Create a DeepAgents `LocalShellBackend` rooted at the repository.
7. Stream messages and tool events back to the CLI.
8. For `init` and `update`, write `openwiki/.last-update.json` after success.

Chat runs skip metadata writes.

## Prompting strategy

`src/agent/prompt.ts` encodes the product rules directly into the system prompt. The agent is instructed to:

- inspect the current codebase and write documentation under `openwiki/`,
- use filesystem discovery tools and git history rather than inventing facts,
- keep the initial wiki focused and navigable,
- document the repository for both humans and future agents,
- respect the repository root as the only project in scope,
- avoid reading secrets or `.env` files,
- use git history for init and update runs,
- respect the temporary plan and update metadata requirements.

The user prompt changes with the command:

- `init` includes the current Git summary and asks for fresh documentation.
- `update` includes last update metadata and a Git change summary.
- `chat` just forwards the user message.

## Git evidence and update metadata

`src/agent/utils.ts` is responsible for the repository evidence that the prompt sees:

- current working tree status,
- current HEAD,
- the most recent 20 commits with changed files,
- a diff summary against HEAD,
- a delta since the last successful update when `.last-update.json` includes a `gitHead` or `updatedAt`.

On successful init/update runs, the agent writes JSON metadata with:

- `updatedAt`
- `command`
- `gitHead`
- `model`

That metadata is later used to scope update runs.

## Model fallback and retries

The agent runtime includes a small retry strategy:

- the selected model is tried first,
- server-side OpenRouter failures can fall back to `OPENROUTER_FALLBACK_MODEL_IDS`,
- retries keep the same command and repository context but may use a modified thread ID.

This behavior was added in recent commits to make automated documentation runs more resilient.

## Why this matters

The agent is not just a generic chat wrapper. It is intentionally constrained so it can:

- write repository-local docs without wandering outside the repo,
- preserve continuity across runs via checkpointing and metadata,
- keep updates grounded in Git evidence,
- support both interactive and scheduled maintenance use cases.

## Things to watch when changing agent behavior

- Keep the prompt in sync with the actual filesystem tools and path conventions used by the CLI.
- Be careful with `.last-update.json` semantics, because update runs use it to decide what changed since the previous successful run.
- Credential loading happens before model resolution; changes there affect both onboarding and agent startup.
- The DeepAgents backend is configured with `virtualMode: true`, which is important for documentation-only behavior.

## Source map

- `src/agent/index.ts`
- `src/agent/prompt.ts`
- `src/agent/utils.ts`
- `src/agent/types.ts`
- `src/constants.ts`
- `src/env.ts`
- Git evidence: commits `ceded10`, `1473a12`, `7bfaeb2`
