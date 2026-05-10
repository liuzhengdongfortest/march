# Session 状态事实源边界

最后更新：2026-05-10

## 背景

当前 March CLI 同时存在两套会话状态：

- March 文件层会话：`.march/sessions/<id>/session.json`，保存 `ContextEngine` 可恢复状态，包括 turns summary、pins、open files、skills、compaction summary 和 fork parent。
- pi JSONL 会话：`.march/pi-sessions/*.jsonl`，由 pi `SessionManager` 管理，保存 agent message tree、tool results、model/thinking changes、compaction、branch summary、labels 和 session info。

Roadmap 的目标是替换 runner 内部 `SessionManager.inMemory(cwd)`，但 March 的产品核心仍是“上下文不会腐烂”。因此不能把“启用 pi 持久化”误当成“所有 March 会话状态已经迁移完成”。

## 决策：会话状态分层，不做隐式双写合并

**为了谁**：服务于 [编码 Agent 能力需求](../requirements/coding-agent-capabilities.md) 和 [群聊上下文引擎](context-engine.md)，同时推进 [pi 能力补齐 Roadmap](../roadmaps/_resolved/2026-05/pi-capability-parity.md)。

**怎么设计**：

- pi JSONL 是 Agent transcript/tree 的事实源：模型消息、工具结果、思考级别、模型切换、compaction entry、branch summary 和 future tree navigation 由 pi `SessionManager` 负责。
- March ContextEngine 是上下文重建材料的事实源：pins、open files、skills、memory/glossary namespace、March 自己的 turn summary、工作区状态和未来群/Agent 私有层由 March 管理。
- `.march/sessions/session.json` 与 `.march/pi-sessions/*.jsonl` 不做透明合并。任何跨源读取必须走显式 adapter，并在 UI/命令上标清来源。
- 截至 2026-05-10 默认切换后，`/sessions`、`/sessions tree`、`/resume <id>` 表示 pi JSONL sessions；旧 March session.json 只能通过 `/sessions legacy`、`/sessions legacy tree`、`/resume-legacy <id>`、`/fork-legacy` 或启动参数 `--legacy-sessions` 显式访问。

**为什么这样**：pi 的 JSONL 能保留完整 agent message tree，但不知道 March 的上下文重建私有状态；March session.json 能恢复 ContextEngine，但没有 pi 的完整 tool/message tree。隐式合并会制造“看似恢复，实则丢状态”的风险。

## 决策：runtime replacement 是恢复 pi session 的前置条件

**为了谁**：服务于后续 `/resume`、fork/tree 和 session switch 的正确性。

**怎么设计**：

- 使用 pi 官方 `AgentSessionRuntime` 模式处理 new/resume/fork/import 这类 session replacement。
- 每次 runtime 替换 active session 后，必须重新绑定：
  - event subscription；
  - extension binding；
  - runner 对 session 的访问；
  - UI retry/tool/thinking/model 事件接线；
  - March ContextEngine 与当前 pi session 的关联元数据。
- 禁止只调用 `SessionManager.open()` 后把新 manager 塞进旧 `AgentSession`。这会让旧 session-local 对象、订阅和 extension context 继续指向旧实例。

**为什么这样**：pi SDK 文档明确 session replacement 需要 runtime rebind。March runner 当前持有 `session` 闭包，直接替换底层 manager 会造成 stale session 引用。

## 决策：迁移采用 opt-in 到 default 的两阶段

**为了谁**：服务于现有 CLI 用户和后续稳定迁移。

**怎么设计**：

### 阶段 A：显式试运行

- `--pi-sessions` 显式启用 pi JSONL SessionManager。
- `/session` 展示底层 persistence 状态。
- `/sessions pi` 只读展示 pi JSONL sessions。
- March `/save`、`/fork`、`/resume` 仍使用 `.march/sessions/session.json`。

### 阶段 B：受控替换

- 引入 runner runtime host，让 active `AgentSession` 可被 runtime 安全替换。
- 新增 pi source 的显式 resume 命令或模式，例如 `/resume-pi <session-id-or-path>`，先不复用 `/resume <id>`。
- resume pi session 时，March ContextEngine 只恢复可确定的 project/cwd/model metadata；pins、open files、skills 等 March 私有状态必须来自当前项目配置、用户显式选择或单独 sidecar。
- 通过 smoke 验证：切换后 session id/file、event subscription、model/thinking、tool events、ContextEngine buildContext 都仍可用。

截至 2026-05-10，阶段 B 的试运行链路已完成，并已进入阶段 C 默认切换：March CLI 默认启用 pi JSONL SessionManager、runtime host、sidecar sync 和 pi session command semantics；`--legacy-sessions` 保留旧 `.march/sessions` 启动期 resume 和默认命令语义。

### 阶段 C：默认切换

- 只有当 pi resume/fork/tree 与 March ContextEngine 私有状态 sidecar 都稳定后，才考虑让默认 runner 使用 pi persisted SessionManager。
- 默认切换时需要迁移提示和向后兼容：旧 `.march/sessions` 仍可列出和手动恢复，不能悄悄失效。

默认切换采用“默认命令指向 pi，旧 March session 显式保留”的命令语义：

| 命令 | 默认切换后语义 | 兼容/过渡入口 |
| --- | --- | --- |
| `/sessions` | 展示 pi JSONL session 列表，并标注来源为 pi | `/sessions legacy` 展示旧 `.march/sessions` |
| `/sessions tree` | 展示 pi JSONL 文件级 parentSessionPath 树 | `/sessions legacy tree` 展示旧 March parent 树 |
| `/resume <id>` | 恢复 pi JSONL session，且必须成功恢复对应 sidecar | `/resume-legacy <id>` 恢复旧 March session |
| `/fork` | 第一版不自动映射到 pi 历史 entry fork，避免误继承 ContextEngine 状态 | `/fork-legacy` 保留旧 March session fork |
| `/sessions pi` | 过渡期作为 `/sessions` alias，输出迁移提示 | 后续可移除 |
| `/resume-pi <id>` | 过渡期作为 `/resume <id>` alias，输出迁移提示 | 后续可移除 |
| `/clone-pi` | 保留为当前 pi active branch clone 的显式命令 | 无 |
| `/fork-pi <entry-id> --reset-context` | 保留为历史 entry reset-context fork 的显式命令 | 无 |

默认切换后的 `--pi-sessions` 和 `--pi-runtime-host` 不再是主路径必要条件：要么成为兼容 no-op 并输出提示，要么只作为 debug flag 保留；不能要求用户继续通过两个 opt-in flag 才获得默认能力。

默认切换门槛已在 2026-05-10 满足，后续回归仍需守住：

- pi JSONL 能恢复 Agent transcript/tree，且 `/resume-pi` 切换后模型、thinking、工具事件、自动 retry、compaction 和后续 prompt 都经过真实流程验证。
- March ContextEngine 私有状态 sidecar 已设计并实现，至少覆盖 pins、open files、active skills、memory/glossary namespace、turn summary 或其替代策略。
- `/session`、`/sessions`、`/sessions pi`、`/resume`、`/resume-pi` 的来源语义清楚；用户不会误以为 March session.json 与 pi JSONL 已自动合并。
- 旧 `.march/sessions` 仍有显式恢复路径，默认切换不会让历史 session 失效。
- 失败路径可回退：runtime host 创建失败、pi JSONL 找不到、session cwd 不匹配、sidecar 缺失时，有清晰错误和不破坏当前会话的行为。

**为什么这样**：先让用户和测试能观察 pi JSONL，再引入 runtime replacement，最后才改默认，可以把最大风险拆成可回退的步骤。

## 可迁移字段边界

| 状态 | 当前事实源 | 迁移策略 |
| --- | --- | --- |
| Agent 消息树 | pi JSONL | 使用 pi `SessionManager` / runtime |
| 工具结果 | pi JSONL | 使用 pi message entries |
| 模型 / thinking 变化 | pi JSONL | 使用 pi entries，March UI 同步读取当前 session |
| compaction / branch summary | pi JSONL | 使用 pi context builder，不转换到 March turn summary |
| turns summary | March session.json | 保留为 March ContextEngine 私有摘要；不反写 pi |
| pins / open files | March ContextEngine | 需要 sidecar 或项目配置；不能从 pi JSONL 推断 |
| skills | March ContextEngine / CLI args | 继续由 March 激活逻辑管理 |
| memory / glossary namespace | March project id | 继续由 `.march/project-id` 管理 |
| fork parent | 两边都有 | March session parent 与 pi `parentSession` 不自动等价，UI 需标注来源 |

## 下一步实现顺序

1. 实现完整交互式会话树 UI 时，必须沿用下方“session explorer + entry detail”的两层信息架构，避免把 pi JSONL 文件级 session tree 与文件内 entry tree 混在同一控件里。
2. `/fork` 默认继续只做 pi branch guidance；真正写入使用 `/clone-pi` 或 `/fork-pi <entry-id> --reset-context`，旧 March fork 使用 `/fork-legacy`。
3. 持续检查 help、autocomplete、错误信息和 `/session` 输出，确保默认 pi 与 legacy 显式回退的来源语义一致。
4. 进入群聊多 Agent 阶段前，设计 group/agent/workspace 到 pi session 的绑定策略，不能只用 cwd 推断 session 归属。

## 决策：交互式会话树 UI 使用两层信息架构

**为了谁**：服务于“每轮修改可以回看和撤销”的编码 Agent 需求，以及 March “上下文不会腐烂”的产品承诺。用户需要理解自己恢复的是哪个运行 transcript、哪个 ContextEngine sidecar，以及 fork 是从当前分支还是历史 user entry 出发。

**怎么设计**：

- 第一层是 `Session Explorer`：展示 pi JSONL 文件级 session 列表/树。数据来自 `SessionManager.list()`，树关系只使用 `parentSessionPath`。这一层的主操作是 `/resume <id>`、`/clone-pi`、查看 sidecar 状态，以及进入某个 session 的 detail。
- 第二层是 `Session Detail / Entry Tree`：进入单个 pi JSONL 文件后，展示文件内 message/entry 时间线和可 fork 的 user entry 候选。数据来自 active `AgentSession.getUserMessagesForForking()` 或后续 pi entry tree API。这一层的写操作只能是 `/fork-pi <entry-id> --reset-context` 或未来显式的其它 fork 策略。
- UI 不能把文件级 session node 和文件内 entry node 放进同一个树控件。它们使用不同事实源、不同恢复语义、不同 sidecar 风险。
- 每个 session node 必须显示来源：`pi JSONL`、当前 marker、cwd、savedAt、message count、sidecar 是否存在/可恢复。legacy session 只能在显式 legacy 分组或 legacy filter 下出现。
- 每个 entry node 必须显示来源：当前 session id、entry id、user prompt 摘要、是否可 fork、fork 后 ContextEngine 策略。第一版只允许 reset-context，不提供默认继承当前 sidecar 的按钮。
- 群聊多 Agent UI 不能直接复用 CLI 的 cwd 过滤心智；进入群聊后，Session Explorer 的顶层来源必须是 `AgentSessionRef`，先按 group/agent/workspace 分组，再展示对应 pi session。

**为什么这样**：文件级 session tree 回答“我要恢复哪次运行历史”，entry tree 回答“我要从同一次运行里的哪个用户消息重新分叉”。这两个问题的状态边界不同：前者需要 pi JSONL + sidecar 一起恢复，后者会制造历史 transcript 与当前 ContextEngine 私有状态的时间点错位。拆成两层可以让 UI 既可导航，又不会暗示 March 已经能隐式重建任意历史 entry 的完整上下文。

## 决策：未来群聊模式下 pi session 必须绑定到 group/agent/workspace

**为了谁**：服务于 [群聊协作需求](../requirements/group-collaboration.md)、[Agent 运行时需求](../requirements/agent-runtime.md) 和 March “上下文不会腐烂”的核心承诺。单 Agent CLI 可以暂时用 cwd 发现 pi session；群聊多 Agent 不能这样做。

**怎么设计**：

- 引入 March 自己的 `AgentSessionRef` 概念，至少包含：
  - `scope`：`cli`、`group` 或 `private`；
  - `groupId`：群聊作用域必填；
  - `agentId`：群成员或私聊 Agent 必填；
  - `workspaceId`：群绑定 workspace 的稳定 ID，不能只存路径字符串；
  - `cwd`：运行时实际工作目录，用于校验和展示；
  - `piSessionFile` / `piSessionId`：指向 `.march/pi-sessions/*.jsonl`；
  - `sidecarFile`：指向对应 ContextEngine sidecar；
  - `createdFrom`：`new`、`resume`、`clone`、`fork-reset` 等来源；
  - `activeBranch` / `entryRef`：后续区分文件级 session tree 与文件内 entry tree。
- CLI 默认路径仍可用 `cwd + piSessionId` 简化查找，但一旦进入群聊 runtime，恢复、clone、fork、Agent 看板和 WorkingSummary 都必须先通过 `AgentSessionRef` 找到 pi session 和 sidecar。
- 同一个 cwd 下允许多个群、多个 Agent、多个 workspace binding 共享或分叉不同 pi session；UI 必须显示 group/agent/workspace 来源，不能只显示 session id。
- `AgentSessionRef` 的存储位置应归属 March 产品数据层，而不是 pi JSONL 文件本身；pi JSONL 继续只做 agent transcript/tree 事实源。
- sidecar metadata 需要逐步补齐 `scope`、`groupId`、`agentId`、`workspaceId`，但 CLI-only sidecar 可以为空或 `scope: "cli"`，避免提前伪造群聊关系。

**为什么这样**：未来一个本地项目可能同时被多个群使用，同一群也可能有多个 Agent 在不同 worktree 中工作。仅用 cwd 查找 pi session 会把不同协作现场混在一起，导致错误恢复、错误 WorkingSummary 注入和错误 Agent 看板状态。March 的长期事实源是群/Agent/workspace 组合；pi session 是该组合下的运行 transcript，不是全局唯一上下文容器。

## 决策：pi clone 可以继承当前 sidecar，pi fork 不能默认继承历史 sidecar

**为了谁**：服务于 Stage 2 的会话树/分支/fork 能力，同时保护 March “上下文不会腐烂”的核心承诺。

**怎么设计**：

- `/clone-pi` 作为第一版显式写命令，只复制当前 pi active branch 到新 JSONL 文件。实现上使用 pi runtime host 的 `fork(entryId, { position: "at" })`，其中 `entryId` 来自当前 `SessionManager.getLeafId()`。
- clone 成功且 runtime 已 rebind 后，立即用当前 March `ContextEngine` 重新写入新 pi session 的 sidecar；sidecar metadata 记录 `derivedFromPiSessionFile`、`derivedFromPiSessionId`、`derivedBy: "clone"`、`derivedAt`。不从旧 sidecar 文件盲拷贝，因为内存里的 ContextEngine 才是当前 turn 后的最新事实。
- `/fork-pi <entry-id>` 面向历史 user entry。pi 会把 selected user message 放回编辑器并从该点之前开新 session，但 March 当前没有历史点对应的 ContextEngine 快照；pins、open files、skills、turn summary 可能包含被 fork 点之后才产生的状态。因此第一版不得默认继承完整 sidecar。
- fork 第一版应先提供候选能力：使用 `AgentSession.getUserMessagesForForking()` 展示可 fork 的 user entry id/text；真正写命令必须选择明确策略，例如只写最小 sidecar metadata，或要求用户显式确认继承当前上下文。
- `/sessions pi tree` 继续表示 pi JSONL 文件级 `parentSessionPath` 树；pi 文件内 `/tree` entry navigation 另做设计，不能混在同一 UI 语义里。
- 默认模式下 March `/fork` 不再执行 legacy 文件层 fork，只提示使用 `/clone-pi` 或 `/fork-pi`；旧 `.march/sessions/session.json` 文件层 fork 保留为 `/fork-legacy`。

**为什么这样**：clone 当前分支与当前 ContextEngine 状态时间点一致，继承是可解释的；fork 历史 user entry 会把 agent transcript 回退到旧点，但 March 私有状态没有对应历史快照，默认继承会制造状态穿越。先 clone、后 fork，可以让可验证路径先闭环，同时不把错误上下文写入新 session。

## 决策：历史 entry fork 必须显式选择降级上下文

**为了谁**：服务于想从早期 prompt 重新尝试的 CLI 用户，同时避免 March 把当前上下文材料伪装成历史上下文。

**怎么设计**：

- `/fork-pi` 无参数保持只读候选展示，列出 `AgentSession.getUserMessagesForForking()` 返回的 entry id 和用户消息摘要。
- 历史 fork 写入命令第一版必须写成 `/fork-pi <entry-id> --reset-context`。没有 `--reset-context` 时只返回 usage，不创建新 session。
- `--reset-context` 表示用户明确接受降级：pi JSONL 从历史 user entry fork；March ContextEngine sidecar 不继承当前 turns、pins、open files、active skills 或 compaction summary。
- fork 成功后写一个降级 sidecar，metadata 至少包含：
  - `derivedBy: "fork-reset"`；
  - `derivedAt`；
  - `derivedFromPiSessionId`；
  - `derivedFromPiSessionFile`；
  - `derivedFromPiEntryId`；
  - `sidecarMode: "reset-context"`。
- 降级 sidecar 的基础字段只保留可确定的项目级元数据：`cwd`、`modelId`、`provider`、`namespace`。`turns`、`pins`、`openFiles`、`skills` 必须为空；`compactionSummary` 必须为 `null`。
- 命令输出必须明确提示：新 pi session 已创建，但 March ContextEngine 以 reset context 启动；需要用户重新 pin/open/activate skill。
- 如果 pi runtime 返回 `selectedText`，March CLI 第一版只把它作为提示输出，不自动塞回编辑器；编辑器恢复要等 REPL 输入层有明确接口后再做。

**为什么这样**：历史 fork 的 agent transcript 和当前 ContextEngine 状态处在不同时间点。`--reset-context` 把风险变成用户显式选择，并让 sidecar 可以通过 `/resume-pi` 正常存在但不假装具备历史私有状态。空 sidecar 比缺失 sidecar 更好，因为缺失会让用户以为实现坏了；带当前状态的 sidecar 更糟，因为它会制造错误上下文。

## 风险

- pi JSONL 文件在没有 assistant message 前可能不会落盘；列表和 resume 要处理“新 session 尚不可列出”的状态。
- 单纯用 pi message history 重建 March turn summary 会丢语义质量；turn summary 不应自动从 assistant text 截断生成。
- `SessionManager.list()` 是文件级列表，tree navigation 是文件内 entry tree；March UI 需要区分“session 列表”和“session 内分支树”。
- 未来群聊模式下，一个群内多个 Agent 可能各自有 pi session；session id 需要绑定 group/agent/workspace，而不是只绑定 cwd。
- fork 历史 user entry 如果继承当前 sidecar，会把后续 pins/open files/skills 带回旧时间点；如果丢弃 sidecar，又会让用户以为 ContextEngine 已恢复但实际缺失私有状态。实现前必须把交互提示和失败路径设计清楚。
- `--reset-context` 会牺牲便利性；后续若要支持“继承当前上下文”，必须使用另一个显式 flag，并在命令输出里标注这是时间点不一致的强制继承。
