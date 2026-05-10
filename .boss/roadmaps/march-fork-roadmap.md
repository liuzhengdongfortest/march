# March Fork 改造路线图

最后更新：2026-05-11

状态：active

上游：`anomalyco/opencode`（版本 1.14.46）
Fork：`liuzhengdongfortest/march`

## 目标

基于 OpenCode fork，注入 March 三大差异化能力（上下文重建引擎、PTY 终端、结构化记忆），打造 March CLI 产品。复用 OpenCode 的 75+ provider、MCP、ACP 协议、Plan/子 Agent、TUI 等能力，聚焦差异化。

## 决策依据

见 [2026-05-11 会议记录](../meetings/2026-05-11-fork-opencode/decisions.md)。

## 竞品参照

- OpenCode（上游）：TUI、模型无关性、ACP 协议
- Claude Code：产品体验和生态标杆
- Codex：安全沙箱和自主循环标杆

## March 差异化（不可丢失）

1. **上下文重建引擎**：open files tracking + content diffing + 自动刷新 + context pressure monitoring
2. **PTY 终端**：node-pty 多 shell + screen buffer + ANSI 渲染 + 多 tab
3. **结构化记忆**：user/feedback/project/reference 四类 + MEMORY.md 索引

设计细节参考 `.boss/source/` 下的旧架构文档。

---

## 阶段 1：Fork 基线

**目标**：fork 跑通，改名，建立开发环境。

| ID | 条目 | 状态 |
|---|---|---|
| 1.1 | Fork + clone + upstream remote 配置 | done |
| 1.2 | `bun install` 依赖安装 | done |
| 1.3 | `bun run build` 构建通过 | done |
| 1.4 | `bun test` 测试套件通过（2604 pass, 65 fail — Windows/网络基线） | done |
| 1.5 | 品牌改名：opencode → march（CLI 入口、TUI 标题、显示字符串） | done |
| 1.6 | `.boss/` 迁移 + CONVENTIONS 更新 | done |
| 1.7 | 初始 commit + push | done |

## 阶段 2：差异化注入

**目标**：在 fork 上实现 March 三大差异化。

| ID | 条目 | 状态 |
|---|---|---|
| 2.1 | 上下文重建引擎——分析 OpenCode Session/Compaction/Processor 层，设计注入点 | planned |
| 2.2 | 上下文重建引擎——实现 open files tracking + content diffing | planned |
| 2.3 | 上下文重建引擎——替换/增强 Compaction，接入 context pressure monitoring | planned |
| 2.4 | PTY 终端——分析 OpenCode shell 模块和 `#pty` 条件导入 | planned |
| 2.5 | PTY 终端——移植 March node-pty adapter 作为 `#pty.node.ts` | planned |
| 2.6 | PTY 终端——screen buffer + ANSI 渲染 + multi-tab 集成 | planned |
| 2.7 | 结构化记忆——新建 `packages/opencode/src/memory/` Effect Service | planned |
| 2.8 | 结构化记忆——实现 remember/forget 工具 + MEMORY.md 索引 | planned |
| 2.9 | 结构化记忆——在 SystemPrompt 中注入相关记忆 | planned |

## 阶段 3：体验打磨

**目标**：中文化 prompt、TUI 品牌定制、端到端验证。

| ID | 条目 | 状态 |
|---|---|---|
| 3.1 | 系统提示词中文化 + March 行为风格注入 | planned |
| 3.2 | TUI 品牌替换（logo、配色、状态栏 March 标识） | planned |
| 3.3 | 默认配置调优（provider、model、permission） | planned |
| 3.4 | 端到端验收测试：三大差异化场景 | planned |

## 阶段 4：上游追版与发布

**目标**：建立 upstream merge 流程，发布 March CLI。

| ID | 条目 | 状态 |
|---|---|---|
| 4.1 | 建立 upstream merge SOP（fetch → merge → resolve → test） | planned |
| 4.2 | 一键安装脚本（install.ps1 / install.sh）适配 | planned |
| 4.3 | 发布到 npm（march-cli 或 march 包名） | planned |

---

## 执行策略

- **阶段 1 一次性完成**：fork 基线跑通后立即 commit，作为后续改动的参照点
- **阶段 2 串行但独立**：三大差异化可以独立设计，但代码注入有依赖（都需要先理解 OpenCode 架构）
- **阶段 3 在 2 完成后**：差异化不跑通就不打磨
- **向上游贡献**：通用改进优先提交到 upstream PR，March 特有代码保持隔离
