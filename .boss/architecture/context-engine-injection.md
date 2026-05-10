# 上下文重建引擎——OpenCode 注入点设计

状态：草稿
日期：2026-05-11

## OpenCode 上下文组装链路

```
llm.ts:107  →  model-specific base prompt (gpt.txt / anthropic.txt / ...)
               + agent.prompt (自定义 agent prompt)
               + input.system (调用级自定义，极少用)

prompt.ts:1580 →  [...env, ...instructions, ...skills]
                  ├─ env: SystemPrompt.environment(model)
                  │     → 模型名、工作目录、workspace、git、平台、日期
                  ├─ instructions: Instruction.system()
                  │     → AGENTS.md / CLAUDE.md / 用户 config.instructions
                  └─ skills: SystemPrompt.skills(agent)
                        → 可用技能列表
```

完整 system prompt = base_prompt + env + instructions + skills + structured_output_prompt

## March 上下文引擎要注入的内容

March 的 pi-coding-agent 上下文引擎核心能力：

1. **Open files tracking**：`[open_files]` 段，每个文件带绝对路径 + 行号 + 完整内容
2. **Content freshness**：每次 turn 前自动刷新 open files 内容（文件可能被外部修改）
3. **Context pressure monitoring**：更精细的 token 压力检测，不止看 token 数，还看文件数、对话轮数
4. **Auto-compaction 决策**：结合压力检测 + 文件变更，更智能的压缩触发

## 注入点设计

### 注入点 A：`SystemPrompt.environment()` —— 追加 [open_files]

**位置**：`packages/opencode/src/session/system.ts:48`

**当前代码**：
```typescript
environment: Effect.fn("SystemPrompt.environment")(function* (model: Provider.Model) {
  const ctx = yield* InstanceState.context
  return [[
    `You are powered by the model named ${model.api.id}...`,
    `<env>`,
    `  Working directory: ${ctx.directory}`,
    ...
    `</env>`,
  ].join("\n")]
})
```

**改造方案**：新增 `OpenFiles.Service`，在 environment 里注入 open files 段：

```typescript
environment: Effect.fn("SystemPrompt.environment")(function* (model: Provider.Model) {
  const ctx = yield* InstanceState.context
  const openFiles = yield* OpenFiles.Service
  const files = yield* openFiles.snapshot()
  
  const envBlock = [
    `You are powered by the model named ${model.api.id}...`,
    `<env>`,
    `  Working directory: ${ctx.directory}`,
    `  Workspace root folder: ${ctx.worktree}`,
    `  Is directory a git repo: ${ctx.project.vcs === "git" ? "yes" : "no"}`,
    `  Platform: ${process.platform}`,
    `  Today's date: ${new Date().toDateString()}`,
    `</env>`,
  ].join("\n")

  const openFilesBlock = files.length > 0 ? [
    `<open_files>`,
    ...files.map(f => `${f.path} (${f.lineCount} lines):\n--- ${f.path} ---\n${f.content}\n---`),
    `</open_files>`,
    `Use open_file to add files to this list. Use close_file to remove them.`,
  ].join("\n") : ""

  return [[envBlock, openFilesBlock].filter(Boolean).join("\n\n")]
})
```

**难度**：低。追加新 Service，不修改现有逻辑。
**风险**：低。新增代码，不影响现有链路。
**优先级**：P0——上下文引擎核心。

### 注入点 B：新建 `OpenFiles.Service` —— 文件追踪引擎

**位置**：新建 `packages/opencode/src/context/open-files.ts`

**接口设计**：
```typescript
export interface Interface {
  readonly open: (path: string) => Effect.Effect<OpenFileInfo>
  readonly close: (path: string) => Effect.Effect<boolean>
  readonly isOpen: (path: string) => Effect.Effect<boolean>
  readonly snapshot: () => Effect.Effect<OpenFileInfo[]>
  readonly refresh: () => Effect.Effect<OpenFileInfo[]>  // re-read files from disk
  readonly get: (path: string) => Effect.Effect<OpenFileInfo | undefined>
}
```

**实现要点**：
- 使用 `InstanceState` 存储 open files 集合（跟随 session 生命周期）
- `refresh()` 在每次 turn 前被调用，检测文件变更
- `snapshot()` 返回当前快照，被 `SystemPrompt.environment()` 消费
- 文件内容加行号，格式化与 March 旧版一致

**难度**：中等。需要理解 Effect-TS 的 `InstanceState` 机制。
**风险**：低。独立模块，通过 Service 接口隔离。
**优先级**：P0。

### 注入点 C：增强 `Compaction` —— 上下文压力监测

**位置**：`packages/opencode/src/session/compaction.ts`

**当前逻辑**（`overflow.ts:19-26`）：
```typescript
export function isOverflow(input) {
  if (input.cfg.compaction?.auto === false) return false
  if (input.model.limit.context === 0) return false
  const count = input.tokens.total || input.tokens.input + ...
  return count >= usable(input)
}
```

纯 token 计数，不感知文件变更、对话轮数。

**改造方案**：
- 保留现有 `isOverflow` 作为主判定
- 新增 `pressureScore()` 返回多维指标：token 使用率、open files 数、对话轮数
- `pressureScore` 在 `Processor` 的 turn loop 中被调用，提前预警

**难度**：中等。扩展接口，不替换核心逻辑。
**风险**：中。compaction 触发时机影响用户体验，需调参。
**优先级**：P1——差异化增强，但不是首期 blocker。

### 注入点 D：Processor turn loop —— 文件 freshness 刷新

**位置**：`packages/opencode/src/session/prompt.ts` turn loop（~1530 行附近）

**改造方案**：在每次 turn 开始前，调用 `openFiles.refresh()` 重新读取文件内容：
```typescript
// 在 loop 的 while(true) 中，step 开始时
yield* openFiles.refresh()
```

这确保 AI 始终看到最新文件内容（用户可能在终端里手动改了文件）。

**难度**：低。一行调用，不改变核心流程。
**风险**：低。refresh 是纯 I/O 操作，失败时降级为 stale 内容。
**优先级**：P1。

### 不推荐的注入点

**不直接修改 `MessageV2`**：OpenCode 的消息格式已经很成熟，把 open files 注入到消息体里会破坏消息语义。通过 system prompt 注入更合适。

**不修改 `LLM.Service`**：LLM 层是 AI SDK 的薄封装，不感知业务逻辑。

## 实施顺序

1. **2.1（本阶段）**：创建 `OpenFiles.Service` + 集成到 `SystemPrompt.environment()`
2. **2.2**：在 Processor turn loop 中加入 `refresh()` 调用
3. **2.3**：增强 Compaction 的 pressure monitoring
4. **2.4**：端到端验证：open file → 对话 → 外部修改文件 → 上下文自动刷新

## 与 OpenCode 现有机制的关系

| OpenCode 机制 | March 增强 | 关系 |
|---|---|---|
| `Instruction.system()` (AGENTS.md) | `OpenFiles.Service` | 互补：AGENTS.md 是静态指令，open files 是动态上下文 |
| `Compaction` | pressure monitoring | 增强：保留现有压缩，加上多维指标 |
| `SystemPrompt.environment()` | `[open_files]` 段 | 扩展：追加新段落 |
| `MessageV2.TextPart` | 不修改 | 独立：open files 通过 system prompt 注入 |
