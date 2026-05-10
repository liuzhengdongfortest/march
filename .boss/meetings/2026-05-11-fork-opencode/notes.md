# 会议记录：Fork OpenCode 改造方案

时间：2026-05-11

## 背景

老板提出直接 fork OpenCode（anomalyco/opencode），在其之上叠加 March 的核心差异化能力（上下文重建引擎、PTY 终端、结构化记忆），而不是从零复刻。核心诉求：

1. 复用 OpenCode 已具备的 75+ provider、MCP、Plan、子 Agent、TUI、Desktop、ACP 协议等能力
2. 通过 ACP 协议兼容 Zed / VS Code 等外部工具
3. 保持 March 的差异化技术壁垒

## 技术发现

### OpenCode 规模

- 20+ packages 的 Bun monorepo
- 主包 `packages/opencode/src/` 60+ 模块
- 核心架构：Effect-TS（函数式依赖注入）贯穿所有模块
- 数据库：SQLite via Drizzle ORM
- TUI：自研 SolidJS-based UI 组件库
- AI SDK：Vercel AI SDK (`ai` 包)

### 关键集成点

OpenCode 的核心上下文链路：

```
Session → MessageV2 → Processor → LLM
                ↓
         Compaction (overflow detection + summarization)
         SystemPrompt (model-specific prompt assembly)
         Instruction (dynamic instruction injection)
```

March 引擎的映射：

- 上下文重建引擎 → 替换/增强 `Compaction` + `SystemPrompt.environment()`
- PTY 终端 → 替换 `shell/` 模块（OpenCode 的 shell 是单文件，March 的更成熟）
- 结构化记忆 → 作为新的 Effect Service 注入到 `SystemPrompt`
- Provider 层 → OpenCode 的 provider 抽象更成熟，pi-ai 作为兼容层

### 技术风险

1. Effect-TS 学习曲线 — 所有模块都用 Effect 的 `Layer` + `Context` + `Effect.gen()` 模式
2. Bun 运行时 — OpenCode 主路径是 Bun，但关键模块（pty、db）有 Node.js fallback
3. 上游追版 — OpenCode 版本 1.14.46，活跃开发中，merge conflict 是持续成本
4. 代码量 — 仅主包就有估计 5 万+ 行 TypeScript，全面理解需要时间

## 讨论要点

- 方案 A（全量 Fork）vs 方案 B（SDK 集成）vs 方案 C（选择性迁移）
- 老板选择方案 A：全面 Fork，承担维护成本
- 关键理由是：直接获得 ACP 兼容 + 完整能力矩阵，比从零追赶性价比高

## 后续

见 decisions.md
