# Docs MR: Terminal progress signals section

Target file: `doc/user/gitlab_duo_cli/reference.md` in gitlab-org/gitlab (branch from master)
Insert position: after the "Environment variables" section (end of file)
Branch: `docs-duo-cli-terminal-progress-signals`

## Section content (append to reference.md)

```markdown
## Terminal progress signals

{{< history >}}

- [Introduced](https://gitlab.com/gitlab-org/editor-extensions/gitlab-lsp/-/releases/v8.79.0) in GitLab Duo CLI 8.79.0, during the GitLab 18.11 release.

{{< /history >}}

In interactive mode, the GitLab Duo CLI reports its status by writing OSC `9;4`
progress escape sequences to `/dev/tty`. Terminals that support this sequence,
such as Windows Terminal and ConEmu, display a progress indicator on the tab or
window that runs the CLI. Terminal multiplexers and status tools can parse the
same sequences to detect the CLI state.

The CLI emits `ESC ] 9 ; 4 ; <state> ST` with these states:

| Sequence   | State         | Meaning                                   |
|------------|---------------|-------------------------------------------|
| `9;4;3`    | Indeterminate | The CLI is processing a request.          |
| `9;4;4;50` | Paused        | A tool call is waiting for your approval. |
| `9;4;0`    | Clear         | The CLI is idle and waiting for input.    |
| `9;4;2`    | Error         | The last request ended with an error.     |

When the CLI runs in a `tmux` session, it wraps the sequences in the `tmux`
passthrough escape so they reach the outer terminal. The CLI writes these
signals only when it is attached to a TTY.
```

## MR title

Docs: add GitLab Duo CLI terminal progress signals reference

## MR description

```markdown
## What does this MR do?

Documents the OSC `9;4` terminal progress signals that the GitLab Duo CLI has
emitted since 8.79.0
([`packages/cli/src/utils/terminal_progress_service.ts`](https://gitlab.com/gitlab-org/editor-extensions/gitlab-lsp/-/blob/main/packages/cli/src/utils/terminal_progress_service.ts)
in gitlab-lsp). The signals are currently undocumented, but they are the way
for terminals, multiplexers, and status tools to detect whether the CLI is
busy, waiting for tool approval, idle, or in an error state.

Documenting them as a public contract lets integrators (terminal multiplexers,
tab/status tools such as
[terminal-tab-status](https://github.com/yaruchyo/terminal-tab-status)) rely on
them without reading the source. The new section sits next to the existing
`AI_AGENT` environment variable documentation, which serves the same
integrator audience.

## Related issues

Related discussion about external observability of CLI state:
https://gitlab.com/gitlab-org/editor-extensions/gitlab-lsp/-/work_items/2684

## Author's checklist

- [x] Follow the:
  - [Documentation process](https://docs.gitlab.com/development/documentation/workflow/).
  - [Documentation guidelines](https://docs.gitlab.com/development/documentation/).
  - [Style Guide](https://docs.gitlab.com/development/documentation/styleguide/).

/label ~documentation
/label ~"docs-only"
/label ~"type::maintenance" ~"maintenance::refactor"
```

## Commit message

```
Docs: add Duo CLI terminal progress signals reference

Documents the OSC 9;4 sequences the Duo CLI writes to /dev/tty since
8.79.0 so terminal and multiplexer integrators can rely on them.

Signed-off-by: Baily <blitheshuo@gmail.com>
```
