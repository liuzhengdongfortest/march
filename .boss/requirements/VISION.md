# 愿景中心

最后更新：2026-05-07

---

## 项目概述

March 的当前愿景是一个本地优先的“群聊式 AI 协作操作系统”。

用户打开的不是一个单 Agent 聊天窗，也不是一个 Task 列表，而是一组长期存在的协作群。每个群有自己的目标、公告、工作空间、成员、订阅、执行框架和历史；多个 Agent 作为常驻成员在群里接话、干活、互相 @、汇报进展。March 的差异化不是“多几个角色”，而是把多人协作的产品模型、Agent 上下文、文件隔离和执行状态做成同一个系统。

旧文档里的核心资产仍然成立：**上下文不会腐烂**。March 必须让 Agent 每轮尽量面对当前事实，而不是被线性累积的历史污染。不同的是，当前事实不再只来自一个 Task，而来自“群 + 大框架 + Agent 私有状态 + 工作区 + 记忆 + 订阅事件”的组合。

来源快照：

- [群聊数据模型 Brainstorm](../source/group-chat-model-brainstorm.md)
- [旧版设计总入口](../source/doc/DESIGN.md)
- [旧版架构入口](../source/doc/ARCHITECTURE.md)

## 全局视图

当前主线整理为五块：

- 群聊协作范式：群取代 Task 成为主容器；私聊独立存在；Agent 从全局池邀请进群。
- Agent 运行时：Agent 常驻、可中断、可恢复，有草稿、WorkingSummary、工作日志和只读看板。
- 上下文事实源：每轮重建 Agent Context，按稳定性分层，区分群公共层与 Agent 私有层。
- 工作区与交付协议：每个 Agent 使用独立 git worktree，`send_group_message` 前后绑定 commit、summary、diff 和 hook。
- 框架、记忆与自动化：用户和 Agent 先维护大框架，执行中补细节；记忆、闹钟、订阅源和 shadow prompt 调优提供支撑。

技术栈不是愿景的一部分。Rust/Tauri、Node、本地 Web、Pi Adapter 都是实现选项；当前判断是优先用最快路径验证产品范式，避免过早绑定重桌面壳。

## 子系统/模块导航

- [群聊协作需求](group-collaboration.md) - 群、私聊、用户隔离、群设置和成员管理。
- [Agent 运行时需求](agent-runtime.md) - Agent 生命周期、激活、中断、草稿、WorkingSummary 和看板。
- [框架记忆自动化需求](task-memory-automation.md) - 大框架、执行细节、记忆、闹钟、订阅源和 shadow prompt。
- [产品壳层需求](product-shell.md) - 本地 Web / IM 风格 UI、双通道事件和设置入口。
- [上下文工作台需求](context-workbench.md) - AI 每轮看到当前项目真实状态，用户能管理上下文材料。
- [编码 Agent 能力需求](coding-agent-capabilities.md) - AI 能安全、可追踪地执行命令、读写文件、调用语义工具。
- [旧版产品 UI 需求](product-ui.md) - 旧 Task 三栏 UI 的可复用设计素材，不再是当前默认产品壳。
- [扩展与长期记忆需求](extensions-and-memory.md) - 旧版 Skills、Agents、Memory、Subconscious 的素材，需按群聊范式重解释。

## 关键用户故事

### 故事：用户在群里指挥一组 AI

**为什么**：复杂工作不是一个 AI 连续聊天能稳定承担的。用户需要像开团队群一样，把架构、编码、审查、测试、产品判断放进同一个协作现场。

**需要什么**：用户需要群列表、群成员、群公告、群工作空间、Agent 常驻状态和群消息流。

**怎么做**：群是主容器；Agent 从全局池邀请进群；群消息、@mention、订阅、闹钟共同触发 Agent。

### 故事：群聊不会越跑越脏

**为什么**：多 Agent 协作比单 Agent 更容易被旧消息、旧文件、旧工具结果污染。

**需要什么**：每个 Agent 下一轮都必须从群公共事实、自己的私有状态和当前工作区重新建立上下文。

**怎么做**：March 自己构建上下文层级；自己的 WorkingSummary 可见，别人的 WorkingSummary 不可见；工具结果只在需要的生命周期内存在。

### 故事：多个 Agent 能并发改代码而不互相踩

**为什么**：如果多个 Agent 共用一个工作目录，冲突会破坏协作。

**需要什么**：每个 Agent 在每个群里有独立工作区、独立 open files、独立日志和清晰汇报。

**怎么做**：群声明多个本地 workspace；Agent 进入群后创建自己的 git worktree branch；`send_group_message` 固定执行 commit、summary、落库和 diff 广播。

### 故事：用户能看到 Agent 的脑内工作状态

**为什么**：群里多个 Agent 同时运行时，用户只看最终气泡会失去控制感。

**需要什么**：用户需要看见每个 Agent 当前上下文、提示词、草稿、工作记忆、执行框架和运行状态。

**怎么做**：Agent 看板做成独立只读弹窗，打开时拉快照，之后订阅增量 SSE。

### 故事：产品先跑起来，再选择重壳

**为什么**：当前最大风险是群聊协作范式是否成立，不是桌面壳选型是否优雅。

**需要什么**：团队需要一个最快能验证群、Agent、上下文、worktree、消息流和 DeepSeek/Pi Adapter 的本地形态。

**怎么做**：优先本地 Web App：Node/TypeScript 后端 + 浏览器前端；Pi 只作为可选单 Agent 执行层；桌面壳后置。
