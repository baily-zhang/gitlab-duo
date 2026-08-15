# Feature MR: CLI Observer Lifecycle Hook Events (#2684)

**日期**: 2026-08-14 创建,2026-08-15 review 修复
**Issue**: https://gitlab.com/gitlab-org/editor-extensions/gitlab-lsp/-/work_items/2684
**Fork**: https://gitlab.com/baily.blithe/gitlab-lsp
**分支**: `feat-cli-observer-hook-events`
**Commits**: `a2fb73ff2f17fe1f390186ef1ee74f7d07a5f07a`(初版)+ `bb5cc6809132900fd54a6a9cd03c8af155a21839`(review 修复:B2 drain 时序、S1 cwd 口径、S3 被杀 hook 日志、S4 注释措辞、S5 日志断言)+ `21940d9a4ac54eae701e5e25acee62de861ccc47`(AI review 修复:SessionEnd 的 sessionId 捕获、cwd 记忆化、drain 超时告警、跨回合顺序语义注释)
**状态**: Draft MR 已创建:https://gitlab.com/gitlab-org/editor-extensions/gitlab-lsp/-/merge_requests/3905(2026-08-15,`Related to #2684`,allow_collaboration 开)。等只读 review 通过 + CI 绿 + #2684 正面回应后转 Ready 并发跟帖。

---

## 一、改动文件清单(15 个文件,+794 / -16)

### packages/lib_hooks(hooks 基础库)

| 文件 | 说明 |
|---|---|
| `src/types.ts` | 新增 `HookEventName` / `ObserverHookEventName` 联合类型;`HooksConfig` 增加 `PermissionRequest` / `Stop` / `SessionEnd` 三个事件键;新增 stdin payload 类型 `PermissionRequestInput` / `StopInput`(含 `status: 'ok' \| 'error'`)/ `SessionEndInput` 及判别联合 `ObserverHookInput`、事件描述符 `ObserverHookEvent` |
| `src/schema.ts` | 严格 zod schema 增加三个事件键;导出共享常量 `HOOK_EVENT_NAMES`,lenient 解析器改为遍历该常量(未知事件名照旧被丢弃) |
| `src/hook_config_loader.ts` | `#mergeConfigs` 改用共享 `HOOK_EVENT_NAMES`,user/project 两级合并与 project 默认禁用逻辑对新事件自动生效 |
| `src/hook_service.ts` | 接口与实现新增 `runObserverEvent(event, sessionId, cwd, options)`;抽取 `#executeHooks` 复用 env 注入(DUO_*/CLAUDE_* 双前缀)与超时换算;新增 `#logObserverResults`:stdout 仅 debug 记录并忽略,非零退出一律 warn(即使 stderr 为空,被杀/信号退出不静默)。观察者事件不做 matcher 匹配(无匹配输入,所有已配置 hook 都执行) |
| `src/index.ts` | 导出新增类型 |
| `src/hook_service.test.ts` | 新增 `runObserverEvent` 用例 ×12(含空 stderr 仍告警) |
| `src/hook_config_loader.test.ts` | 新增观察者事件解析/合并/未知事件过滤用例 ×4 |

### packages/cli(接线)

| 文件 | 说明 |
|---|---|
| `src/utils/hook_lifecycle_service.ts` **(新)** | `HookLifecycleService`:`observeStream()` 以与 `TerminalProgressService.trackStream` 完全一致的 settle 判定包装流 —— 最后元素为 `tool` 且 `approval_request` → `PermissionRequest`;最后元素为 `error` 或流抛错 → `Stop(status=error)`;否则 → `Stop(status=ok)`;消费方提前放弃迭代(未 settle)不发事件。dispatch 为 fire-and-forget(不 await、catch 全部错误仅记日志,注释明确不保证跨回合相对顺序),in-flight promise 入 `#inFlight` 集合。公开 `drain()`(等待所有 in-flight hook 完成,5s 封顶,超时记 warn 并含在飞数量)供退出路径调用;实现 needle `AsyncDisposable`:`disposeAsync()` 发出 `SessionEnd` 后复用 `drain()`,`SessionEnd` 优先用最近一次 dispatch 捕获的 sessionId(容器销毁时 activeSession 可能已清)。cwd 取初始化时解析的 workspace root(`workspaceFolders[0]`,与 SessionStart 同口径,首个解析结果记忆化),无则回退启动 cwd |
| `src/utils/hook_lifecycle_service.test.ts` **(新)** | 20 个用例:三事件触发条件、approval settle 只发 PermissionRequest 不发 Stop 的互斥、retry 元素排除、流抛错重抛、hook 拒绝不影响主流程且 TestLogger 记录 warn、无活跃 session 跳过、workspace root cwd 口径及记忆化、drain 等待 in-flight、drain 超时告警(假时钟)、disposeAsync 发 SessionEnd 后 drain、activeSession 清空后 SessionEnd 复用捕获的 sessionId、提前放弃不发事件 |
| `src/commands/tui/turn_controller.ts` | 注入 `HookLifecycleService`,在 `trackStream` 外层套 `observeStream`(TUI 交互回合) |
| `src/commands/tui/tui_controller.ts` | DI 依赖表 + 构造器透传给 TurnController |
| `src/commands/run/run_controller.ts` | headless `duo run` 同样接线(它同样走 trackStream);**在 `exitHandler.exit()` 前与 catch rethrow 前显式 `await hookLifecycle.drain()`** —— DI teardown 中 HookExecutor 先于本 service 处置并会杀掉仍活跃的 hook 进程,不 drain 则 headless 的 Stop hook 会被静默杀死 |
| `src/di.ts` | 注册 `HookLifecycleService`(与现有 hooks 服务同块) |
| `src/commands/tui/tui_controller.test.ts` / `src/commands/run/run_controller.test.ts` | 构造参数补 mock(passthrough 生成器);run_controller 新增 2 个 drain 时序测试(exit 在 drain resolve 前不被调用;失败 rethrow 前也 drain) |

**未改动**:`TerminalProgressService` 零改动(仅在调用点外层包一层)。

## 二、验证结果(全部通过,2026-08-15 第三次 commit 后复跑)

| 检查 | 命令 | 结果 |
|---|---|---|
| Lint/Format | `bun scripts/dev/fix-changed.ts` | prettier ok / eslint ok / markdownlint no files |
| 受影响包 typecheck | `tsc --noEmit`(packages/cli + packages/lib_hooks) | 两包均 exit 0(第二 commit 时全仓 `bun run compile` 112 tasks successful) |
| lib_hooks 单测(jest) | `npx jest --config jest.unit.config.ts packages/lib_hooks` | **3 suites / 55 tests passed** |
| cli 包全量(bun test) | `bun run --filter @gitlab/duo-cli test` | **121/121 test files passed**(hook_lifecycle_service.test.ts 20 例) |
| TUI 包 | `bun run --filter @gitlab-org/tui test` | 24/85 失败,**与本改动无关**:`git stash` 后干净 checkout 复跑同样 24/85 失败(本地沙箱环境性失败,改动不含任何 tui 文件) |

工具链:mise 管理的 bun(按 AGENTS.md 要求 `mise exec` 全程执行)。

## 三、Draft MR 草稿

**标题**:
```
Draft: feat(cli): add observer lifecycle hook events (PermissionRequest, Stop, SessionEnd)
```

**描述**(英文):

```markdown
## What does this MR do and why?

Implements the observer lifecycle hook events proposed in #2684, extending the
CLI hooks system beyond `SessionStart` with three fire-and-forget events:

| Event | Fires when |
|---|---|
| `PermissionRequest` | a turn settles on a tool call awaiting user approval |
| `Stop` | a turn settles back to idle; the payload carries `status: "ok" \| "error"` |
| `SessionEnd` | the CLI session shuts down |

This lets users wire up notifications (bells, desktop/webhook pings) and
automation for the "waiting for approval" / "done" / "exited" moments,
matching the hook events they already know from other agent CLIs.

### Design decisions

- **Fire-and-forget by design.** Unlike `SessionStart`, these events do not
  consume stdout and cannot block or mutate the session: stdout is logged at
  debug level and ignored, non-zero exits are logged as warnings, and
  dispatches are never awaited by the running turn. A broken hook can never
  break a turn.
- **Settle detection lives in a new `HookLifecycleService`,**
  wrapped around the stream at the same call sites as
  `TerminalProgressService.trackStream` (TUI `TurnController` and headless
  `RunController`), with identical settle semantics —
  `approval_request` → `PermissionRequest`, error settle / stream throw →
  `Stop(status=error)`, otherwise `Stop(status=ok)`. A stream abandoned
  early by its consumer never settles, so no event fires.
  `TerminalProgressService` itself is intentionally left untouched: OSC
  progress reporting and hook dispatch are separate concerns, and the
  observer wrapper passes elements through unchanged.
- **`SessionEnd` fires from async disposal, and exit paths drain first.**
  The service implements needle's `AsyncDisposable`; container disposal
  during `ExitHandler.exit()` dispatches `SessionEnd` and drains in-flight
  hook runs (capped at 5s, mirroring the shutdown timeout). Because the
  `HookExecutor` disposes before this service and kills any hook process
  still active, the headless `RunController` explicitly awaits
  `hookLifecycle.drain()` before invoking `exit()` (and before rethrowing a
  failed run), so the `Stop` hooks dispatched at stream settle are not
  silently terminated by DI teardown.
- **Config plumbing is reused, not duplicated:** user/project config merge
  (project hooks stay opt-in via `--enable-project-hooks`), per-hook timeouts,
  sensitive env var stripping (`GITLAB_TOKEN` etc.) and the
  `DUO_*`/`CLAUDE_*` env injection all come from the existing
  loader/executor. Matchers are ignored for observer events (they have no
  matcher input); every configured hook for the event runs.

### Example

`~/.gitlab/duo/hooks.json`:

```json
{
  "hooks": {
    "PermissionRequest": [{ "hooks": [{ "type": "command", "command": "notify-send 'Duo needs approval'" }] }],
    "Stop": [{ "hooks": [{ "type": "command", "command": "jq -r .status >> ~/duo-turns.log" }] }]
  }
}
```

### Out of scope

- `UserPromptSubmit` / `PreToolUse` / `PostToolUse` (kept as TODOs in the
  config types; they need consume-stdout semantics like blocking/mutation).
- The ACP server path does not stream through these controllers and does not
  emit observer events yet.
- Documentation for https://docs.gitlab.com will be submitted as a separate
  MR to `gitlab-org/gitlab` once the behavior here is settled.

## References

- Related to #2684

## MR acceptance checklist

- [x] Tests added: `packages/lib_hooks/src/hook_service.test.ts`,
      `packages/lib_hooks/src/hook_config_loader.test.ts`,
      `packages/cli/src/utils/hook_lifecycle_service.test.ts`
- [x] `bun scripts/dev/fix-changed.ts` clean
- [x] `mise exec -- bun run compile` clean (112 turbo tasks)
- [x] lib_hooks jest suite and full `@gitlab/duo-cli` bun suite green locally
```

## 四、#2684 跟帖草稿(英文)

```markdown
As promised — I've got a working implementation of the observer lifecycle
events up as a draft: <MR_LINK>

It adds `PermissionRequest`, `Stop` (with `status: ok|error`) and
`SessionEnd` as fire-and-forget events: stdout ignored, failures only
logged, never blocking the turn. Settle detection mirrors the existing
terminal progress tracking exactly, and config loading reuses the current
user/project merge with project hooks still disabled by default.

Docs for docs.gitlab.com to follow in a separate gitlab-org/gitlab MR.
Feedback very welcome, happy to adjust scope or semantics.
```

(发帖前把 `<MR_LINK>` 替换为 !3905 地址:https://gitlab.com/gitlab-org/editor-extensions/gitlab-lsp/-/merge_requests/3905)

## 五、遗留风险与说明

1. **headless 已覆盖**:`duo run`(RunController)与 TUI 走同一个 `trackStream` 调用形态,已同样接线;RunController 在 `exitHandler.exit()` 前(及失败 rethrow 前)显式 `await drain()`,`SessionEnd` 在 `disposeAsync` 中发出并同样 drain(5s 封顶),保证 hook 进程不被 DI teardown 中先处置的 HookExecutor 静默杀死。时序有专门测试锁定。
2. **ACP 模式不发事件**:ACP server 路径不经过这两个 controller(也不经过 trackStream),本切片不覆盖,已在 MR 描述 Out of scope 标注。
3. **用户中断(ESC 取消)**:取消后流正常收尾(settle)通常发 `Stop(status=ok)`;若 teardown 让流以 error 元素/抛错收尾,则走 error 分支发 `Stop(status=error)`。与 trackStream 的 settle 语义一致。若社区希望区分 `interrupted`,可在 review 中加 status 值,向后兼容。
4. **卡死 hook**:hook 默认 30s 超时,但 drain 5s 封顶 + ExitHandler 5s 强退;退出路径最坏情况约 5s 后放行/强退,detached 子进程组会被留下继续跑完(与 SessionStart 行为一致)。
5. **TUI 包 24/85 测试失败为本地环境既有失败**(stash 验证),CI 环境预期无此问题;如 CI 出现同样失败,与本 MR 无关。
