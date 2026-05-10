# 架构总览

最后更新：2026-05-07

---

## 整体设计思路

March 的当前架构中心是“群聊协作编排器”。群是主容器，Agent 是可邀请进群的全局资产，Agent turn 是可被调度、打断、记录和恢复的运行单元。

旧架构文档保留了很多有价值的上下文、Provider、工具、UI 和记忆设计，但 Rust/Tauri/Task 三栏不再是当前默认架构。当前实现应优先服务愿景验证：群能不能跑、Agent 能不能协作、上下文能不能保持干净。

来源快照：

- [群聊数据模型 Brainstorm](../source/group-chat-model-brainstorm.md)
- [旧版设计总入口](../source/doc/DESIGN.md)

## 系统结构

顶层可以看成六个协作层：

- Product Model 层：用户、群、私聊、Agent 全局池、群成员、群公告、群工作空间、执行框架。
- Orchestrator 层：消息分发、Agent 激活、硬中断、闹钟、订阅轮询、运行状态广播。
- Context Engine 层：按稳定性和可见性构建 Agent Context，区分群公共层和 Agent 私有层。
- Workspace 层：每个 Agent 在群内使用独立 git worktree，所有文件修改通过 commit/diff 协议进入群。
- Execution Adapter 层：调用 LLM provider、Pi SDK 或自研 loop，执行工具并把 streaming delta 回写给 March。
- Local App 层：本地 Web 前端 + 后端 API/SSE/WS，后续再决定是否包桌面壳。

## 关键设计决策

### 决策：群取代 Task 成为主容器

**为了谁**：服务于 [群聊协作需求](../requirements/group-collaboration.md)。

**怎么设计**：群持久化成员、消息、公告、workspace、执行框架、订阅、hook 和 Agent 群内状态。旧 Task 概念剥离，未来只作为任务管理能力重新进入。

**为什么这样**：用户心智是“我在一个协作房间里推进事情”，不是“我管理一堆上下文窗口”。

### 决策：March 掌握事实源，Pi 只可能是执行层

**为了谁**：服务于 [Agent 运行时需求](../requirements/agent-runtime.md) 和 [Pi Adapter POC 报告](../wiki/pi-adapter-poc-report.md)。

**怎么设计**：群消息、上下文、draft、WorkingSummary、worktree、日志、执行框架和落库都由 March 管；Pi 或其他 SDK 只执行单个 Agent turn。

**为什么这样**：March 的核心竞争点是协作编排和上下文事实源，不是某个 provider 调用封装。

### 决策：硬中断优先于优雅等待

**为了谁**：服务于多 Agent 群聊的实时交互。

**怎么设计**：用户新消息进入群时，Orchestrator 立即 abort 相关 Agent 的 SSE 和工具调用，记录 `interrupted by user message`，然后基于新上下文重启。

**为什么这样**：群聊里用户发话意味着现场事实改变，继续等旧上下文跑完会让 Agent 越跑越偏。

### 决策：优先本地 Web 验证，而不是先做重桌面壳

**为了谁**：服务于最快验证 [愿景中心](../requirements/VISION.md)。

**怎么设计**：第一阶段用 Node/TypeScript 后端和浏览器前端跑通本地应用。需要桌面能力时再接轻量壳或桌面封装。

**为什么这样**：当前最大不确定性是产品和编排，不是 Rust/Tauri/Electron 选型。

## 模块/关注点导航

- [群数据模型](group-data-model.md) - 用户隔离、群、私聊、Agent 池和三库职责。
- [Agent 编排器](agent-orchestrator.md) - 生命周期、激活源、硬中断、闹钟、订阅和看板。
- [群聊上下文引擎](context-engine.md) - Agent Context 分层、WorkingSummary、草稿、工作日志和可见性规则。
- [工作区与 Git 协议](workspace-git.md) - worktree 隔离、`send_group_message`、commit、diff 和 hook。
- [本地 Web 运行时](local-web-runtime.md) - Node/TypeScript 后端、Web UI、SSE/WS 和桌面壳后置。
- [浏览器回归策略](browser-regression-strategy.md) - Chromium CDP smoke 与后续 Firefox/WebKit Playwright 覆盖路线。
- [Agent Board 事件重放设计](agent-board-event-replay.md) - Agent board 快照、Last-Event-ID、短期 replay ring 和持久 event log 边界。
- [订阅 Provider Auth 设计](subscription-provider-auth.md) - 真实订阅的 provider account、credential ref、OAuth/GitHub App 和本地 token 生命周期。
- [旧版上下文核心设计](context-core.md) - 从 Task 时代迁来的上下文材料，可继续复用思想。
- [Session 状态事实源边界](session-state-boundary.md) - pi JSONL SessionManager 与 March ContextEngine session.json 的职责、迁移和 runtime replacement 前置条件。
- [旧版工具与 Provider 设计](tooling-and-provider.md) - 命令、文件工具、LSP、Provider、Reasoning 和快照回退素材。
