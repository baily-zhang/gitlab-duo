# Comment for gitlab-lsp issue #2684

Target: https://gitlab.com/gitlab-org/editor-extensions/gitlab-lsp/-/work_items/2684

---

**Adding another consumer profile for these events — external terminal integrations (multiplexers, status bars) — and one event the current list doesn't cover.**

I've been evaluating GitLab Duo CLI support for [herdr](https://github.com/herdrdev/herdr), an agent-native terminal multiplexer that marks each pane as working / blocked / idle. The [terminal-tab-status](https://github.com/yaruchyo/terminal-tab-status) author hit the same wall and ended up intercepting escape sequences through `NODE_OPTIONS` injection: today there is no supported way for an external tool to observe the session lifecycle.

From that angle, two additions to this proposal:

1. **`PermissionRequest`** — fires when a tool call enters the approval-pending state. For observer-type consumers this is the highest-value event: it is the exact moment a human needs to return to the terminal. The proposed `PreToolUse`/`PostToolUse` fire around *execution*, so a tool call that sits waiting for approval (or is denied) never fires either of them. The CLI already models this state precisely — `TerminalProgressService.trackStream()` switches to *paused* when the stream settles on a tool element in `approval_request` state (`packages/cli/src/utils/terminal_progress_service.ts`) — so a natural emit point exists.

2. **Fire-and-forget semantics for observer events.** For `PermissionRequest` and `Stop`/`SessionEnd` (item 4 in the proposal), observers don't need the `hookSpecificOutput.additionalContext` stdout contract. A notification-only variant keeps these events cheap and side-effect-free, which might be a useful design distinction from the context-returning events (`UserPromptSubmit`, `PreCompact`).

For anyone landing here with the status-bar use case today: since 8.79.0 the CLI already writes OSC `9;4` progress sequences to `/dev/tty` (busy `9;4;3`, approval-pending `9;4;4;50`, idle `9;4;0`, error `9;4;2`, tmux-passthrough aware). That covers PTY-level observers such as multiplexers. Hooks would cover the non-PTY cases (separate-process status tools, telemetry bridges), so the two are complementary rather than redundant.

Happy to help implement the observer slice (`PermissionRequest` + `Stop`/`SessionEnd`) as a community contribution if the team is open to it.
