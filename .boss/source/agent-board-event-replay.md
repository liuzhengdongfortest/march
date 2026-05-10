# Agent Board 事件重放设计

最后更新：2026-05-08

## 当前边界

Agent board 当前已具备：

- 群聊 board：`GET /api/groups/:groupId/agents/:agentId/board` 快照。
- 群聊专用流：`GET /api/groups/:groupId/agents/:agentId/events`，连接后先推 `bootstrap` 和 `agent_board` 快照，再推 `agent_status`、`draft`、`tool`、`message` 增量。
- 私聊 board：`GET /api/private-chats/:chatId/board` 快照。
- 私聊专用流：`GET /api/private-chats/:chatId/board/events`，连接后先推 `bootstrap` 和 `agent_board` 快照，再推 `private_status`、`tool`、`message` 增量。

当前 `EventBus` 已有进程内 ring replay 第一切片：按 channel 写 `id:`，读取 `Last-Event-ID`，短暂断线期间的近窗口增量可重放；服务进程重启后的持久重放仍未覆盖。

## 决策：快照负责当前事实，replay 负责连接间隙

**为了谁**：服务于用户打开 Agent board 时“看到当前状态”的需求，而不是追求每个 UI delta 永久可回放。

**怎么设计**：

- 每次连接仍先发送 `agent_board` 快照，快照覆盖当前状态：status、draft、last summary、recent tool logs、context。
- `Last-Event-ID` 只用于补“快照之后、live subscribe 之前”或短暂断线重连期间错过的事件。
- 快照能重建的状态不强依赖 replay；例如最新 draft/status/summary 优先读 DB。
- replay 重点覆盖短生命周期但用户会感知的事件顺序：tool start/end、draft delta、message/diff 到达提示。

**为什么这样**：Agent board 是监督界面，不是审计日志。已有 DB 事实源比重放所有 SSE 更可靠；replay 的价值是让 UI 重连时不闪烁、不漏掉刚发生的可见事件。

## 决策：第一切片做进程内 ring buffer，持久 event log 后置

**为了谁**：服务于本地 Web MVP 的稳定性，同时控制实现复杂度。

**怎么设计**：

第一切片已完成：

- `EventBus` 给每个 channel 维护递增 `eventId`。
- SSE 输出增加 `id: <eventId>`。
- `EventBus` 按 channel 保留最近 N 条事件，当前默认 500 条。
- `subscribe(channel, res, { filter, lastEventId })` 先按 filter 重放 `id > lastEventId` 的 ring 事件，再注册 live client。
- 群聊和私聊专用流从 `Last-Event-ID` header 读取游标。
- `event-bus-replay-smoke` 验证：发布事件后，带 last id 重连只收到后续事件，filter 仍生效，ring 截断和 close 后 fanout 也可验证。

后续持久切片：

- 新增 `event_stream_entries` 表：`channel`、`event_id`、`type`、`payload_json`、`created_at`、`ttl_kind`。
- 只持久化需要跨进程重放的事件：message、diff、tool start/end、status transition；高频 draft delta 仍可只保留最新 draft 快照。
- 启动时从 DB 恢复最近窗口，或直接由 DB 查询补发。

**为什么这样**：进程内 ring 能解决最常见的 EventSource 断线重连；持久 event log 需要清理策略、隐私边界和 DB 膨胀控制，应该在验证 ring 有价值后再做。

## Event ID 语义

- event id 只在单个 channel 内单调递增。
- 群聊 channel：`group_demo`。
- 私聊 channel：`private:private_march_assistant`。
- Agent board 专用流不单独生成 channel，而是在 channel 事件上套 filter。
- `agent_board` 快照可以使用当前 channel 最新 id 之后的新 id，避免客户端把快照当成历史事件重复应用。

## 客户端策略

- `EventSource` 自动携带 `Last-Event-ID`；前端不需要手写 header。
- 前端仍按现有 `syncAgentBoardFromEvent()` 幂等 patch。
- 如果服务端返回的第一条是 `agent_board` 快照，前端先用快照覆盖，再应用后续 replay/live 增量。
- 如果 ring 过期，服务端可发 `event: replay_reset`，前端重新接受快照即可。

## 风险

- 只做 ring buffer 时，服务进程重启后仍无法 replay。
- 高频 draft delta 如果全部入 ring，可能放大内存和 UI 抖动；需要只保留最近窗口。
- 私聊当前 draft/status 快照较弱，后续若要强一致，需要把私聊运行状态也落库。
- Event id 如果全局共用，过滤后会出现 id 跳跃；这是可接受的，客户端只关心“下一次从这个 id 后继续”。

## 下一刀建议

EventBus ring replay 第一切片已完成。下一步建议：

1. 将 Agent board 的一条连接快照场景迁入 Playwright Firefox/WebKit runner，验证 `id:` 输出不影响跨引擎 EventSource。
2. 观察 replay ring 是否足够；如果出现服务重启后仍要补事件，再进入持久 `event_stream_entries` 切片。
3. 对高频 draft delta 做合并或窗口限速，避免 ring 内存和 UI 抖动被放大。
