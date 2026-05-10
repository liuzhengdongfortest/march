# 决策记录

## 决策 1：采用全量 Fork 策略

**决定**：Fork `anomalyco/opencode`，在 OpenCode 代码基础上进行改造，注入 March 核心竞争力。

**理由**：
- 直接获得 ACP 协议兼容（Zed/VS Code 集成）
- 75+ provider、MCP、Plan、子 Agent、TUI、Desktop 等能力免于自建
- Roadmap 中约 70% 的条目归零
- 社区生态可被 March 继承

**权衡**：接受 Effect-TS + Bun 的技术栈迁移成本，接受上游 merge 的持续维护成本。

## 决策 2：保留 March 三大差异化

**决定**：以下 March 独有能力必须作为 Fork 后的首要改造目标：

1. **上下文重建引擎**：替换 OpenCode 的 Compaction + 增强 SystemPrompt
2. **PTY 终端**：替换 OpenCode 的 shell 模块
3. **结构化记忆**：作为新的 Effect Service 注入

## 决策 3：技术兼容策略

**决定**：
- 保留 OpenCode 的 Bun + TypeScript + Effect-TS 技术栈
- March 现有 Node.js 代码（Phase 1-2 成果）作为参考实现，用 TypeScript 重写
- pi-ai 的 ModelRegistry 功能由 OpenCode 的 Provider 层替代
- node-pty 适配器通过 OpenCode 的 `#pty` 条件导入机制集成

## 决策 4：分阶段 Fork 计划

**决定**：不一次性完成迁移，分三个阶段：
1. **Fork + 跑通**：建立 fork 仓库，确保构建/测试通过，改名 March
2. **差异化注入**：逐一替换上下文引擎、终端、记忆
3. **体验打磨**：中文化 prompt、TUI 定制、品牌替换
