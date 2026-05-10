# Agent 编排器

最后更新：2026-05-07

## 决策：Orchestrator 负责所有激活源

**为了谁**：服务于 [Agent 运行时需求](../requirements/agent-runtime.md)。

**怎么设计**：统一处理用户消息、Agent 消息、`mention_agent`、闹钟和订阅事件。每个事件都转成明确 activation source。

**为什么这样**：Agent 不应该各自轮询世界；系统层统一调度才能控制中断、并发和落库。

## 决策：用户消息触发硬中断

**为了谁**：服务于实时群聊体验。

**怎么设计**：用户消息落库后，Orchestrator 立即 abort 该群正在运行的 Agent turn，保留已写日志，重建上下文并并发启动新 turn。

**为什么这样**：新消息是更高优先级事实，等待旧工具调用完成会牺牲交互确定性。

## 决策：Agent 看板走快照加增量

**为了谁**：服务于可观察性。

**怎么设计**：看板打开时读取当前 Agent state snapshot，之后订阅该 Agent 的 SSE delta；关闭即取消订阅。

**为什么这样**：看板是调试和监督工具，按需拉取比长期广播全部脑内状态更省资源。

## 决策：Pi Adapter 位于执行层边界

**为了谁**：服务于复用 Pi，同时不牺牲 March 主架构。

**怎么设计**：Orchestrator 把 March 构建好的 context、工具 allowlist、worktree cwd 和 abort signal 传给 Pi Adapter；Pi 返回 streaming delta、tool events 和最终消息。

**为什么这样**：POC 证明 Pi 可跑单 Agent turn，但群、draft、summary、worktree 和消息落库必须由 March 控制。
