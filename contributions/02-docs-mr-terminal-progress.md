# Docs MR: Terminal progress signals section (v2 — 双 agent 审定版)

Target file: `doc/user/gitlab_duo_cli/reference.md` in gitlab-org/gitlab
Insert position: after the "Environment variables" section (end of file)
Branch: `docs-duo-cli-terminal-progress-signals` — **建在 community fork(gitlab-community/gitlab-org 侧)上**,个人 fork 超配额不可用
开 MR 时选仓库自带 "Documentation" 描述模板;标签开 MR 后走 `@gitlab-bot` 回帖链接加:`~documentation ~docs-only ~"type::maintenance" ~"maintenance::refactor"`

## Section content (append to reference.md)

```markdown
## Terminal progress signals

{{< history >}}

- [Introduced](https://gitlab.com/gitlab-org/editor-extensions/gitlab-lsp/-/releases/v8.79.0) in GitLab Duo CLI 8.79.0, during the GitLab 18.11 release.

{{< /history >}}

In both interactive and headless modes, the GitLab Duo CLI reports its status by
writing OSC `9;4` progress escape sequences to `/dev/tty`. Terminals that
support this sequence, such as Windows Terminal and ConEmu, display a progress
indicator on the tab or window that runs the GitLab Duo CLI. Terminal
multiplexers and status tools can parse the same sequences to detect the
GitLab Duo CLI state.

The GitLab Duo CLI emits `ESC ] 9 ; 4 ; <state> ST` with these states:

| Sequence   | State         | Description                                       |
|------------|---------------|---------------------------------------------------|
| `9;4;3`    | Indeterminate | The GitLab Duo CLI is processing a request.       |
| `9;4;4;50` | Paused        | A tool call is waiting for your approval.         |
| `9;4;0`    | Clear         | The GitLab Duo CLI is idle and waiting for input. |
| `9;4;2`    | Error         | The last request ended with an error.             |

The GitLab Duo CLI writes these signals only when its standard output is
attached to a terminal and the `/dev/tty` device is available. Because the
signals go to `/dev/tty` instead of standard output, they do not appear in
redirected or captured output. When the GitLab Duo CLI exits, it clears the
progress state.

When the GitLab Duo CLI runs in a `tmux` session, it wraps the sequences in the
`tmux` passthrough escape sequence. In `tmux` 3.3 and later, the sequences
reach the outer terminal only if the `allow-passthrough` option is turned on.

The GitLab Duo CLI can also send
[system notifications](use.md#system-notifications) when a session needs your
attention.
```

## MR title

Document GitLab Duo CLI terminal progress signals

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
```

## Commit message

```
Document GitLab Duo CLI terminal progress signals

Document the OSC 9;4 sequences the Duo CLI writes to /dev/tty since
8.79.0 so terminal and multiplexer integrators can rely on them.

Signed-off-by: Baily Zhang <blitheshuo@gmail.com>
```

## 审核记录

- v1 → v2 变更依据:两个独立 review agent(CI lint 实测 + 维护者视角逐条核对)。
- 必改三项(均已落实):①headless 模式同样发信号,去掉 "interactive mode" 限定(证据 run_controller.ts:176);②history 补 ", during the GitLab 18.11 release."(availability_details.md 规范 + tag 日期定标);③tmux 措辞改为 `allow-passthrough` 条件式(tmux 3.3+ 默认 off)。
- 增补两条集成方关心的行为:信号不进重定向输出(走 /dev/tty);退出时清理进度状态(dispose→idle)。
- lint 状态:v1 已过官方 markdownlint(同 CI 版本)+ Vale error 级 0 条;v2 需复跑确认(工具与配置在 /tmp/verify-docs/repo)。
