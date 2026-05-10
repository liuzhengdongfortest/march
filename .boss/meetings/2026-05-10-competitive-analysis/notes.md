# 会议记录：竞品分析与 March 改进方向

日期：2026-05-10

## 议题

老板要求对 March 与三大竞品（Claude Code、OpenCode、Codex）做全面差距分析，找出 March 该补什么、该优化什么。同时指出 March 的代码是 Codex 写的，UX 很差、UI 难看，需要重点打磨。

## 三大竞品

### Claude Code（Anthropic）

- 发布时间：2025-02，176+ 版本发布
- 技术栈：ripgrep 全文搜索 + LSP 语义导航（~50ms）
- 年收入：$1B（2025年12月）
- 关键能力：Agent Skills API、MCP（300+ 集成）、后台任务/异步 Agent、IDE 集成（VS Code/JetBrains/Cursor/Windsurf）、Slack 集成、安全审查、图片/PDF 读取、自动记忆、权限分级系统、子 Agent/委托模式、Plan 模式、撤销/重做（快照）、Hooks 系统、Web 搜索/抓取、自动模型切换

### OpenCode（anomalyco）

- 发布时间：2025-05，725+ 版本发布
- 95K+ GitHub Stars，2.5M 月活开发者，778 贡献者
- 技术栈：Bun + TypeScript + Hono HTTP Server + Zig 渲染后端
- 关键能力：75+ LLM 提供商（模型无关）、TUI Mission Control、内置双 Agent（build/plan）、Agent SDK、IDE 集成（VS Code/Cursor）、MCP、Hooks/扩展、Web 搜索、本地模型支持、图片/多模态、自动测试运行、多文件编辑、沙箱/权限、文件监听器、LSP 工具、持久化服务端模式、桌面应用

### Codex（OpenAI）

- 发布时间：2025-05
- 技术栈：Rust 95.6%，Apache 2.0 开源
- 关键能力：沙箱执行（bubblewrap/Docker devcontainer）、/goal 自主循环、Hooks 引擎、MCP、并行工具调用、AGENTS.md、apply_patch 工具、Python SDK、CI/CD 集成（Autofix）、企业 CA 证书/代理、设备码登录、特性开关系统、远程 WebSocket 访问、Transcript 模式

来源：tavily 搜索 GitHub README、官方文档、第三方评测。

**注意**：老板指出实际体验中的差距不止会上列出的这些。当前清单是起点，执行过程中需要持续发现和补充。

## March 当前能力基线

## March 当前能力基线

- 上下文重建引擎（分层模型：[tools] → [session_status] → [active_skills] → [open_files] → [runtime_status] → [shells] → [recent_chat]）
- 工具集：read_file、write_file、edit_file（行号区间 + fallback 文本匹配）、glob、grep、list_files、sandbox_bash、open_file/close_file
- PTY 交互式终端（node-pty）：shell_spawn/send/search/snapshot/kill/list
- 技能系统（.march/skills/、activate_skill/deactivate_skill）
- 文件监听器（open_files watcher）
- nocturne_memory 4 实体图记忆系统
- send_turn_summary + Pi compaction 双层压缩
- TUI 界面 + 终端抽屉

## 讨论要点

### 老板的核心观点

- 三大竞品各有优势，但 March 不需要妄自菲薄——上下文重建引擎和 PTY 终端是真正的差异化
- Codex 写的代码导致 UX 差、UI 丑，这是必须要修的
- 竞品分析的核心目的是"知道差距在哪，针对性补"

### March 的核心优势

1. **上下文重建引擎**——分层注入、前缀缓存友好、每轮从事实源重建。这是 March 的根本差异化，三个竞品都没有做到这个程度。
2. **PTY 交互式终端**——不只是 spawn→等结果，而是真正的伪终端控制（发送按键、搜索 scrollback、获取 ANSI 快照）。Claude Code 的 Bash 工具和 Codex 的沙箱都没有这个。
3. **nocturne_memory**——4 实体图模型 + 版本链 + 循环检测，比 Claude Code 的 auto-memory 更结构化。
4. **edit_file 行号区间**——省 token（11-45%），竞品都用文本匹配。
5. **群聊协作愿景**——虽然还没建，但产品模型上比三个竞品的单 Agent 模式更先进。

### 识别到的差距

按 Tier 分：

**Tier 0 — 桌面级缺失（不补就落后）：**
- MCP 支持（三个竞品全有，300+ 集成生态）
- Web 搜索/抓取（三个竞品全有）
- 权限分级系统（Claude Code 和 Codex 都有）
- 多提供商支持（目前只连 DeepSeek，OpenCode 支持 75+）

**Tier 1 — 工作流短板：**
- Plan 模式（Claude Code 和 OpenCode 都有）
- 子 Agent / 并行任务（三个竞品都有）
- 撤销/重做（Claude Code 用快照，OpenCode 用 git）
- IDE 集成（三个竞品都有 VS Code 集成）
- 图片/PDF 读取

**Tier 2 — 体验打磨（重点）：**
- TUI 视觉质量（OpenCode 的 Mission Control 是最佳参考）
- 安装体验（一键脚本 vs 当前手动）
- 特性开关/配置系统
- 自动模型切换
- 后台任务流式输出

**Tier 3 — 差异化增强：**
- /goal 式自主循环
- Hooks 系统（生命周期钩子）
- 沙箱安全（参考 Codex bubblewrap）
- CI/CD 集成
- Agent SDK / 可编程接口

### Prompt 微调策略

老板提出两层 Prompt 优化方法：

**第一层：竞品 Prompt 参考。** 直接研究 Claude Code、OpenCode、Codex 的系统提示词（从源码中提取），对比 March 当前的 prompt，学习竞品在以下维度的写法：
- 工具调用指令的精确度
- 错误恢复和边界情况的引导
- 代码风格和输出格式约束
- 安全边界和权限说明
- 上下文窗口利用策略

**第二层：元调试（Meta-Debugging）。** 这是一个很聪明的自举技巧——让 March 给自己看病：
- 在 March 里问 DeepSeek："你现在在一个名为 March 的 Agent 容器里，试一下 XXX 功能，觉得哪里有不顺手的地方"
- AI 作为工具的使用者和执行者，能从内部感受到 prompt 矛盾、工具返回格式难解析、上下文切换不自然、响应延迟体感等问题
- 这些问题从开发者视角很难发现，但从 AI 视角一目了然
- 可以系统性覆盖：每个工具、每种交互模式、边界情况（空状态、错误态、长输出截断）

**第三层：元调试 + 源码审查。** 老板指出只让 AI 体验功能不够——容器暴露的是接口，实现里的问题（状态机缺陷、竞态条件、缓冲区边界、错误传播断裂）从接口看不出来：
- AI 在体验功能的同时，读对应的源码实现，从"原始角度"发现问题
- 提示词组织由 AI 助手负责设计，关键是让 AI 同时扮演"使用者"和"代码审查者"两个角色
- March 理论上可以自举（自己改自己），但当前还小，由 AI 助手扶着执行——助手根据 March 的自诊断结果来改代码

### TUI 设计参考

老板明确指出 March 的 UI 是 Codex 写的，难看。OpenCode 的 TUI Mission Control 是当前终端 AI Agent 的 UI 标杆：
- Zig 渲染后端保证流畅
- 双 Agent 切换（Tab 键）
- 持久化服务端模式（消除冷启动）
