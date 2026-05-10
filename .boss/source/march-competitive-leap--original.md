# March 竞争力跃升路线图

最后更新：2026-05-11 (evening)

状态：active

会议记录：[2026-05-10-competitive-analysis](../meetings/2026-05-10-competitive-analysis/notes.md)

## 目标

让 March 在功能完备度、产品体验、Prompt 质量和自进化能力四个维度上全面逼近并超越 Claude Code / OpenCode / Codex，同时保持上下文重建引擎、PTY 终端、结构化记忆三大核心差异化。

## 竞品参照

- Claude Code：产品体验和生态标杆（MCP、权限、后台任务、撤销）
- OpenCode：TUI 和模型无关性标杆（Mission Control、75+ 提供商）
- Codex：安全沙箱和自主循环标杆（bubblewrap、/goal）

## 需求锚点

- 补齐桌面级缺失能力（MCP、Web 搜索、权限、多提供商）
- TUI 视觉全面升级（对标 OpenCode Mission Control）
- Prompt 质量从竞品萃取 + 元调试双轨提升
- 建立元调试闭环——March 自诊断 → 发现 → 修复 → 验证
- 差距清单是开放集，执行中持续发现和补充

---

## 当前边界

- 不在此 Roadmap 推进群聊协作（VISION 里的群+多 Agent+worktree）
- 不在此 Roadmap 做 IDE 插件（插件是独立产品，先聚焦 CLI）
- March 自举（自己改自己）为远期目标，当前由 AI 助手扶着执行

---

## 阶段 1：桌面级能力补齐

**目标**：消除进入门槛——MCP、Web 搜索、多提供商、权限分级。不补就无法站在同一起跑线。

| ID | 条目 | 状态 |
|---|---|---|
| 1.1 | MCP 客户端——连接和管理 MCP Server，工具发现、调用、MCP 工具注入 March 工具集 | done |
| 1.2 | Web 搜索工具——`web_search`（tavily/brave API），搜索结果注入上下文 | done |
| 1.3 | Web 抓取工具——`web_fetch`（单页内容提取），Markdown 转换 | done |
| 1.4 | 权限分级系统——只读/写/执行/外部网络四级，首次使用审批，会话记忆 | done |
| 1.5 | OpenAI 兼容提供商——在现有 DeepSeek 基础上加入 OpenAI API 兼容，Provider 抽象层 | done |
| 1.6 | 提供商配置 CLI——`--provider` / `--model` 完善，`/providers` REPL 命令切换 | done |

## 阶段 2：TUI 和体验重设计

**目标**：推倒 Codex 写的烂 UI，对标 OpenCode Mission Control 的终端视觉质量。与阶段 1 并行推进。

| ID | 条目 | 状态 |
|---|---|---|
| 2.1 | TUI 视觉审计——逐屏截图对比 OpenCode，列出所有视觉差距点 | done |
| 2.2 | 主题和色彩系统——统一调色板、字体样式、组件 token，参考 OpenCode + March 既有 design-system | done |
| 2.3 | 消息渲染重设计——用户消息/AI 回复/工具调用折叠/流式输出的视觉层次 | done |
| 2.4 | 状态栏重设计——当前模型、token 用量、上下文压力、活跃技能，实时刷新 | done |
| 2.5 | Diff 渲染升级——语法高亮 + 行号 + 变更类型标注（新增/修改/删除） | done |
| 2.6 | 终端抽屉升级——shell 列表、标签切换、ANSI 渲染保真度 | done |
| 2.7 | 一键安装脚本——Windows (install.ps1) + macOS/Linux (install.sh) | done |
| 2.8 | 特性开关系统——`~/.march/features.toml`，实验性功能按需开启 | done |

## 阶段 3：Prompt 竞品萃取与优化

**目标**：提取三大竞品的系统提示词精华，对比 March 当前 prompt，针对性优化。

| ID | 条目 | 状态 |
|---|---|---|
| 3.1 | 提取 Claude Code 系统提示词——从 pengchengneo/Claude-Code 源码定位 prompt 注入点 | in_progress |
| 3.2 | 提取 OpenCode 系统提示词——从 anomalyco/opencode 源码定位 prompt 注入点 | planned |
| 3.3 | 提取 Codex 系统提示词——从 openai/codex 源码定位 prompt 注入点 | planned |
| 3.4 | Prompt 对比矩阵——四家（March + 三竞品）在工具调用、错误恢复、安全边界、代码风格、上下文利用五个维度的写法对比 | planned |
| 3.5 | March Prompt v2——基于对比结果重写 March 系统提示词，重点优化工具调用精度和边界情况引导 | planned |
| 3.6 | 验证——Prompt v2 回归测试（同等任务对比 token 效率、成功率和输出质量） | planned |

## 阶段 4：元调试闭环（第一轮）

**目标**：建立 March 自诊断能力。让 March 里的 DeepSeek 同时作为使用者和代码审查者，从三层深度发现问题。

| ID | 条目 | 状态 |
|---|---|---|
| 4.1 | 元调试提示词设计——构建三层审查提示词模板（表层体验 + 源码审查 + 交叉验证），覆盖所有工具和交互模式 | planned |
| 4.2 | 工具体验审查——对每个 March 工具，让 AI 使用后评估 prompt 清晰度、返回格式、边界情况 | planned |
| 4.3 | 源码审查——对每个核心模块（context engine、tools、memory、shell、cli），让 AI 读源码找状态管理/并发/边界问题 | planned |
| 4.4 | 交互模式审查——多轮对话流、上下文切换、技能激活/停用、shell 操作等复合场景的体验审查 | planned |
| 4.5 | 诊断报告——汇总所有发现，按严重程度和修复成本排序，生成可执行的修复清单 | planned |
| 4.6 | 修复执行——根据诊断报告逐一修复，每个修复可追溯到诊断条目 | planned |

## 阶段 5：工作流能力升级

**目标**：补齐 Plan 模式、子 Agent、撤销/重做、图片/PDF 等工作流关键能力。

| ID | 条目 | 状态 |
|---|---|---|
| 5.1 | Plan 模式——`/plan` 命令，AI 先分析→提计划→用户确认→执行，读-only 探索阶段 | planned |
| 5.2 | 子 Agent 并行——`task` 工具，启动子 Agent 执行独立任务，父 Agent 收集结果 | planned |
| 5.3 | 撤销/重做——基于 git 的操作级撤销（`/undo` 回退最后一条 AI 消息及其文件改动），`/redo` 恢复 | planned |
| 5.4 | 图片输入——粘贴/拖拽图片到终端，base64 编码注入上下文（已有 image-clipboard 雏形，需完善） | planned |
| 5.5 | PDF 读取——`read_pdf` 工具，提取文本内容注入上下文 | planned |
| 5.6 | 自动重试和退避——工具调用失败时的指数退避重试策略 | planned |
| 5.7 | 后台任务——`&` 前缀或 `/bg` 启动后台 Agent，完成后通知，状态栏显示运行中任务 | planned |

## 阶段 6：差异化增强

**目标**：建立 March 独有的技术壁垒——自主循环、安全沙箱、Hooks、SDK。

| ID | 条目 | 状态 |
|---|---|---|
| 6.1 | 自主循环（March Loop）——对标 Codex /goal，AI 规划→执行→验证→迭代直到目标达成或预算耗尽 | planned |
| 6.2 | Hooks 系统——生命周期钩子（session_start/stop、pre_tool/post_tool、user_prompt），参考 Codex hooks engine | planned |
| 6.3 | 沙箱安全加固——命令执行白名单/黑名单、文件系统边界限制、网络访问控制 | planned |
| 6.4 | March SDK——可编程接口，`march run "prompt" --json` 模式完善，供 CI/CD 和其他工具调用 | planned |
| 6.5 | 性能与冷启动优化——启动时间、首次响应延迟、上下文组装耗时 | planned |

---

## 执行策略

- **阶段 1 和阶段 2 并行**：功能底线和 UI 体验没有依赖关系，可以同时推进
- **阶段 3 和阶段 4 交错**：元调试的结果会反向修正 Prompt 优化方向，形成反馈闭环
- **阶段 5 和 6 依赖前四个阶段的完成**：工作流升级需要在功能完备、体验良好、Prompt 精准的基础上才有意义
- **差距清单动态更新**：每个阶段执行过程中发现的新差距，直接追加到对应阶段条目，不做全局重排

## 参考

- 竞品分析会议：`.boss/meetings/2026-05-10-competitive-analysis/`
- 上一条 Roadmap（已完成）：[March CLI 实施路线图](_resolved/2026-05/march-cli.md)
- March CLI 源码：`march-cli/src/`
- 竞品仓库（本地克隆）：
  - Claude Code: `D:\playground\claude-code\`（源：<https://github.com/pengchengneo/Claude-Code>）
  - OpenCode: `D:\playground\opencode\`（源：<https://github.com/anomalyco/opencode>）
  - Codex: `D:\playground\codex\`（源：<https://github.com/openai/codex>）
