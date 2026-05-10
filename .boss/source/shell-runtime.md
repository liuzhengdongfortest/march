# Shell Runtime 设计决策

## 为了谁

为 March 的 AI agent 和用户 TUI 提供同一个交互式终端事实源。AI 需要用工具启动、输入、搜索和观察交互式进程；用户需要看见这些进程并在必要时接管输入。

## 怎么设计

### 分层

- `shell/runtime`：headless PTY 事实源，负责 spawn/send/search/snapshot/kill/list、scrollback 和 cleanup。
- `agent/tools`：把 runtime 暴露成 `shell_*` 工具，不直接持有 PTY 进程。
- `context`：从 runtime 读取 `[shells]` 纯文本摘要，默认最近 200 行、剥离 ANSI。
- `cli/shell-*`：后续 `/shell` 命令和 TUI 抽屉，只读 runtime 状态并把用户输入转发给 runtime。

### Shell 状态

每个 shell 是一个内存对象：

- `id`：稳定短 id，工具调用使用。
- `name`：用户可读名称，允许自动去重。
- `command` / `args` / `cwd`：启动参数。
- `status`：`starting | running | exited | failed | killed`。
- `exitCode` / `signal` / `error`：退出与失败原因。
- `createdAt` / `updatedAt`：用于列表排序和诊断。
- `scrollback`：环形缓冲，保存 raw ANSI chunk 和 plain text line 两种视图。
- `screen`：由 `@xterm/headless` 维护的当前 viewport 模型，负责清屏、光标移动、当前屏幕 plain/ANSI 导出。

### 默认 shell 策略

- Windows 第一切片默认使用 `powershell.exe -NoLogo -NoProfile`。
- `cmd.exe` 暂不作为默认路径；阶段 0 验证出现挂住，需要独立兼容测试后再打开。
- Unix 后续默认使用 `$SHELL`，缺失时回退 `sh`。

### 输出视图

- `plain`：给 `[shells]`、`shell_search` 和普通工具结果使用；剥离 ANSI，限制行数。
- `screen.plain` / `screen.ansi`：给 `shell_snapshot`、`/shell <id>` 和 TUI 抽屉使用；这是当前屏幕事实源，能表达清屏、光标定位和全屏 TUI viewport。
- `ansi`：保留 raw ANSI scrollback，作为历史调试 fallback，不再作为 drawer 的首选视图。
- 默认上下文只用 `plain`，避免 ANSI token 噪音。

### Cleanup

- `killShell(id)` 先尝试向进程发送 shell 友好的退出输入；超时后再 `term.kill()`。
- `killAll()` 在 March dispose/exit 时调用，必须等待短超时并吞掉已知 Windows helper 噪音，但要把诊断保留给 `/shell` 或测试断言。
- Windows `AttachConsole failed` 是阶段 0 已知风险；不能把它当成未知异常静默丢弃。

### 错误模型

- spawn 失败：shell 进入 `failed`，工具返回错误文本和结构化 details。
- send 到已退出 shell：拒绝并提示当前 status。
- 输出过大：scrollback 环形截断，返回中明确标注 truncated。
- shell 不存在：工具返回可恢复错误，不抛穿 runner。

## 为什么这样

- PTY 生命周期是事实源，必须先独立于 UI 做稳定；否则终端抽屉、工具调用和上下文层会抢状态。
- `ui.mjs` 已接近 500 行换轨线，终端抽屉必须后置并拆模块，不能把 runtime 状态塞进 TUI 主文件。
- AI 日常只需要知道“shell 现在输出了什么”，完整 ANSI 是高成本信息，应该通过 `shell_snapshot` 按需获取。
- Windows 是当前主要验证环境，先把 PowerShell 路径做稳，比泛化所有 shell 更符合风险排序。
