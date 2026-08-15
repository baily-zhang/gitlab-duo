# gitlab-duo

Working log of my upstream contributions to the [GitLab Duo CLI](https://docs.gitlab.com/user/gitlab_duo_cli/)'s terminal-integration surface.

## Context

GitLab Duo CLI (`duo` / `glab duo cli`) is GitLab's interactive terminal coding agent. While evaluating how it integrates with agent-aware terminal multiplexers (such as [herdr](https://github.com/herdrdev/herdr)) and status-bar tools, two gaps surfaced:

1. **The hooks system only exposes a single event** (`SessionStart`), so external tools cannot observe the session lifecycle — when the agent is working, waiting for tool approval, or done. A third-party author ran into the same wall and had to [intercept escape sequences via `NODE_OPTIONS` injection](https://github.com/yaruchyo/terminal-tab-status).
2. **The OSC `9;4` terminal progress signals the CLI already emits are undocumented.** Since v8.79.0 the CLI writes precise machine-readable state (busy / waiting-for-approval / idle / error) to `/dev/tty`, tmux-passthrough aware — but integrators can only discover this by reading the source.

## Contributions

| # | What | Where | Status |
|---|------|-------|--------|
| 1 | Proposal: observer lifecycle hook events (`PermissionRequest`, `Stop`/`SessionEnd`) for terminal multiplexer and status-bar integrations | [gitlab-lsp#2684 comment](https://gitlab.com/gitlab-org/editor-extensions/gitlab-lsp/-/work_items/2684#note_3689352199) | Posted 2026-08-14 |
| 2 | Docs: document the OSC `9;4` terminal progress signals as a public contract | `doc/user/gitlab_duo_cli/reference.md` in gitlab-org/gitlab ([draft](contributions/02-docs-mr-terminal-progress.md)) | Drafted, lint-verified against the official markdownlint/Vale configs; awaiting community-fork access to submit |
| 3 | Feature: implement the observer hook events (`PermissionRequest`, `Stop`, `SessionEnd`) | [Draft MR gitlab-lsp!3905](https://gitlab.com/gitlab-org/editor-extensions/gitlab-lsp/-/merge_requests/3905) · [patch](contributions/feat-cli-observer-hook-events.patch) · [notes](contributions/03-feature-mr-hooks.md) | Draft MR open (15 files, +794/−16; under independent review before marking ready) |

## Technical notes: the OSC `9;4` signals

Emitted by [`packages/cli/src/utils/terminal_progress_service.ts`](https://gitlab.com/gitlab-org/editor-extensions/gitlab-lsp/-/blob/main/packages/cli/src/utils/terminal_progress_service.ts) since [v8.79.0](https://gitlab.com/gitlab-org/editor-extensions/gitlab-lsp/-/releases/v8.79.0):

| Sequence   | State         | Meaning                                   |
|------------|---------------|-------------------------------------------|
| `9;4;3`    | Indeterminate | The CLI is processing a request.          |
| `9;4;4;50` | Paused        | A tool call is waiting for your approval. |
| `9;4;0`    | Clear         | The CLI is idle and waiting for input.    |
| `9;4;2`    | Error         | The last request ended with an error.     |

Written to `/dev/tty` only when attached to a TTY; wrapped in the tmux passthrough escape (`DCS tmux; … ST`) when `$TMUX` is set. This makes PTY-level observers (multiplexers) work today, while lifecycle hooks (contribution #1/#3) would cover non-PTY observers — complementary, not redundant.
