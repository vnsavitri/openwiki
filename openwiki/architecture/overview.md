# Architecture overview

OpenWiki has a small but layered architecture:

1. `src/cli.tsx` provides the interactive terminal application and orchestrates runs.
2. `src/commands.ts` parses argv and defines help text and supported options.
3. `src/credentials.tsx` manages interactive onboarding for the OpenRouter key, model selection, and optional LangSmith tracing.
4. `src/env.ts` reads and writes `~/.openwiki/.env` and surfaces credential diagnostics.
5. `src/agent/index.ts` runs the documentation agent, collects Git context, and writes update metadata.
6. `src/agent/prompt.ts` builds the system and user prompts that tell the model how to behave.
7. `src/agent/utils.ts` gathers Git evidence and records `.last-update.json` after successful init/update runs.
8. `src/constants.ts` centralizes environment keys, default model IDs, and the wiki directory names.

## Runtime shape

The CLI starts in `src/cli.tsx`, parses the command, and then either:

- prints help and exits,
- opens the interactive chat UI,
- runs an init/update command against the current repository, or
- performs a dry-run in development mode.

For non-chat runs, the agent receives a `RunContext` that includes last-update metadata and a Git summary generated from:

- `git status --short`
- `git rev-parse HEAD`
- `git log --max-count=20 --name-status --oneline`
- `git diff --name-status HEAD`
- a change window since the previous successful OpenWiki update when metadata exists

The agent then uses a DeepAgents `LocalShellBackend` rooted at the repository, but configured with `virtualMode: true`, `maxOutputBytes: 100_000`, and a 120 second timeout.

## Why the architecture is shaped this way

The current design reflects a documentation product rather than a general-purpose agent framework:

- The CLI owns user experience and credential bootstrap so the tool is install-and-run friendly.
- Git evidence is collected in the host process before the agent starts so the model sees stable repository context.
- Update metadata is written only after successful non-chat runs, which lets later updates diff from the last known good state.
- Model fallback is handled in the agent runtime, allowing OpenWiki to retry across a small set of models when OpenRouter returns server-side errors.

## Major extension points

- Add or refine CLI commands in `src/commands.ts` and the corresponding UI behavior in `src/cli.tsx`.
- Change onboarding or local credential storage in `src/credentials.tsx` and `src/env.ts`.
- Adjust model defaults or validation in `src/constants.ts`.
- Extend the documentation prompt or Git evidence in `src/agent/prompt.ts` and `src/agent/utils.ts`.
- Modify run persistence behavior in `src/agent/utils.ts`.

## Things to watch when editing

- `src/cli.tsx` and `src/commands.ts` must stay aligned; help text and parser behavior are intentionally coupled.
- Credential setup writes to a real home-directory file, so permission handling matters.
- The agent is expected to work from repository-local virtual paths like `/README.md` and `/openwiki/quickstart.md`; the prompt explicitly warns about this.
- `openwiki/` in the target repository is both the docs output location and the metadata location for `.last-update.json`.

## Source map

- `src/cli.tsx`
- `src/commands.ts`
- `src/credentials.tsx`
- `src/env.ts`
- `src/agent/index.ts`
- `src/agent/prompt.ts`
- `src/agent/utils.ts`
- `src/constants.ts`
- `package.json`
- Git evidence: commits `7bfaeb2`, `1473a12`, `ceded10`, `4f7bb4c`
