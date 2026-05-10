# sandbox_bash 隔离路线

最后更新：2026-05-08

## 当前边界

`sandbox_bash` 现在是 March-owned tool，不裸用 Pi 内置 shell。已具备：

- `cwd` 固定在 Agent worktree。
- `shell: false`，不经系统 shell 拼接命令。
- 禁止嵌套 `powershell`、`pwsh`、`cmd`、`bash`、`sh`。
- 默认 `timeoutMs=10000`，调用方可在 `1000..60000` 范围内调整。
- 返回 `code`、`signal`、`timedOut`、`durationMs`、`timeoutMs`、stdout 和 stderr。
- 工具文本会对 stdout/stderr 做 12000 字符上限截断，并在 details 记录原始长度、limit、truncated 和 omitted。
- timeout/abort 已有进程树清理第一切片：Windows 使用 `taskkill /T /F`，Unix 使用 detached process group。
- 命令能力分级已有第一切片：嵌套 shell、明显破坏性系统/文件命令、`git clean` 和 `git reset --hard` 会返回结构化 `blocked`。
- 外部 hard abort 仍可中断长命令并触发同一套清理。

## 仍然不足

- 进程树清理仍是 best-effort；Windows 尚未接入 Job Object，显式 detached 的更深层后代仍需后续平台级验证。
- 没有 CPU、内存、文件数量、网络访问等资源限制。
- 命令能力分级仍是最小 denylist，无法识别 `node`/`python` 脚本内部的破坏性行为。
- 没有平台隔离，Windows/macOS/Linux 的进程和权限语义不同。

## 推荐路线

### 第一层：进程树清理

为了短期可靠性，先保证 timeout/abort 能清理同一命令拉起的进程树。第一切片已落地底层 helper 和 smoke。

- Windows：当前用 `taskkill /T /F`；后续可评估 Job Object。
- Unix：当前用 detached process group，并对 group 发送 signal。
- 验收：`npm run command-tree-timeout-smoke` 启动父进程拉起子进程，timeout 后验证子进程结束。

### 第二层：命令能力分级

为了降低误伤，把命令分成可默认运行和需确认/禁用的层级。第一切片已落地最小 denylist 和 smoke。

- 默认允许：`git`、语言包管理器的只读/测试类命令、项目脚本。
- 默认禁用：磁盘格式化、系统关机、权限修改、裸删除命令、嵌套 shell、破坏性 git 清理。
- 验收：`npm run command-policy-smoke` 覆盖允许和拒绝策略。

### 第三层：资源和网络策略

为了防止复杂命令拖垮本机，引入资源策略。

- 已先记录耗时和输出量，并对工具文本做 stdout/stderr 上限截断。
- 后续再按平台能力限制运行时间、并发数和更底层的输出捕获。
- 网络访问先作为策略开关，不默认做深度拦截。

### 第四层：容器/微虚拟化评估

等本地 Web MVP 稳定后，再评估是否引入容器或轻量虚拟化。

- 目标不是“绝对安全”，而是把高风险命令从用户主工作区隔开。
- 需要评估 Windows/macOS 开发体验、文件同步、启动成本和 Electron/桌面壳路线。

## 下一步建议

输出量/资源观测上限第一切片已完成。下一刀优先把命令策略升级为可配置 allow/deny policy，或继续补运行时间/并发数的 March-owned resource policy。不要先上容器；那会把 MVP 的主要风险从群聊协作转移到平台工程。
