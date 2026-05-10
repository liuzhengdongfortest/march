# UI 运行时设计

最后更新：2026-05-08

---

## 适用状态

当前主壳是本地 Web 的群聊协作 UI。旧 Task 三栏资料只保留可复用的事件模型、状态边界和 token 思路，不再决定产品壳层。

当前实现参考了 `clowder-ai` 的成熟前端效果，但只吸收层级、密度、富块和协作现场感，不复制品牌、人设、文案或技术栈。

## 当前 Web UI 决策

### 决策：群聊三层壳层取代旧 Task 三栏

**为了谁**：服务于 [产品壳层需求](../requirements/product-shell.md) 和“用户在群里指挥一组 AI”的愿景故事。

**怎么设计**：左侧只放群/私聊会话列表；中间是 IM 风格聊天主舞台；右侧是当前群状态层，优先展示活跃 Agent、待处理提案、项目框架摘要和最近事件。群设置、完整框架和 Agent 详情进入低频面板或浮层。

**为什么这样**：March 的主心智是协作房间，不是任务管理工具。右侧如果承载太多编辑表单和日志，会重新滑回控制台/看板。

### 决策：消息流分三层渲染

**为了谁**：服务于聊天现场里同时出现人、Agent、环境事件、工具输出和代码变更的场景。

**怎么设计**：普通对话使用用户/Agent/private bubble；订阅、闹钟、系统事件使用 timeline notice；工具日志、diff、proposal 使用 work block。work block 可以挂在相关消息下，也可以在右侧状态层保留摘要。

**为什么这样**：不同信息的阅读方式不同。人和 Agent 的话要像聊天；环境事件要像现场通知；工具和 diff 要像技术证据。

### 决策：技术证据使用深色 work block

**为了谁**：服务于用户快速识别工具调用、stdout、错误和代码 diff。

**怎么设计**：diff、工具日志和 stdout 使用深色技术底板、monospace、可折叠 header 和短摘要；diff 顶部提供文件导航 chip，正文按文件分组并用 add/remove/hunk/meta/context 行级视觉区分代码证据，同时在文件块内保留 unified 原文并追加 split diff 审阅视图。diff API 按行分页，前端在同一个 work block 内加载更多，并支持对具体文件行添加轻量持久批注、resolved/reopened 状态和一层回复串，避免大 diff 一次性压垮聊天主舞台。外层仍保持 March 的暖色协作视觉。

### 决策：Agent board 使用专用实时流

**为了谁**：服务于用户打开单个 Agent 视角时持续观察状态、draft、工具日志和 WorkingSummary。

**怎么设计**：主群 SSE 仍负责群消息流；Agent board 打开后额外连接 per-agent SSE，连接建立后先发送 `bootstrap` 和 `agent_board` 快照基线，之后后端通过 EventBus filter 只推送目标 Agent 的 `agent_status`、`draft`、`tool` 和 `message` 事件。关闭 board 时断开专用流，避免长期持有无关连接。

**为什么这样**：技术输出和聊天气泡混在一起会降低扫读效率；深色底板能把“机器执行证据”从“协作发言”中分离出来。

### 决策：Agent board 是只读增量浮层

**为了谁**：服务于 [Agent 运行时需求](../requirements/agent-runtime.md) 中“用户能打开 Agent 看板”的故事。

**怎么设计**：群聊 Agent board 从右侧状态卡进入，先读取 Current snapshot，再用 per-agent SSE 连接快照建立实时基线，并增量更新当前回合、草稿、WorkingSummary 和最近工具日志。私聊 Agent board 从私聊头部进入，先读取 private context 快照，再用私聊 board 专用 SSE 建立连接快照并消费私聊状态增量。它只读，不承载群设置编辑，不演化成任务看板。

**为什么这样**：用户需要理解 Agent 正在怎样工作，但不需要在看板里管理任务。Agent board 的边界越清楚，群聊主舞台越稳定。

事件重放路线见 [Agent Board 事件重放设计](agent-board-event-replay.md)：快照负责当前事实，Last-Event-ID/ring replay 只补短暂连接间隙。

### 决策：输入区表达当前指挥目标

**为了谁**：服务于群聊/私聊并存、多个 Agent 同时运行的输入场景。

**怎么设计**：composer 显示当前会话类型、会话名、群内 Agent 数和 draft 状态；placeholder 随群聊/私聊切换。输入区不写使用说明，不暴露键盘教学，只保留当前目标状态。

**为什么这样**：用户在多群、多 Agent、多事件环境里最容易错发消息。输入区要帮助确认“我正在对谁说话”，而不是变成说明书。

### 决策：响应式先隐藏右侧状态层

**为了谁**：服务于普通笔记本和窄屏浏览器验证。

**怎么设计**：桌面保留三栏；中宽隐藏右侧状态层，保持左侧会话和聊天主舞台；窄屏把会话列表改成横向列表，消息宽度收敛，输入区不横向溢出。

**为什么这样**：MVP 阶段优先保证聊天主流程可用。右侧状态层是增强信息，不应该挤压主舞台。

## 设计背景

这一设计服务于 [产品 UI 需求](../requirements/product-ui.md)。March 的 UI 需要同时展示完整用户视图和 AI 实际上下文，不能把后端事实、运行时事件和前端视图状态揉成一个粗粒度 snapshot。

## 设计决策

### 决策：三栏布局承载三类核心对象

**为了谁**：服务于“桌面三栏工作台”的用户故事。

**怎么设计**：左栏 Task 列表，中栏聊天区，右栏上下文面板；左右栏支持折叠 rail。

**为什么这样**：任务、对话、上下文是 March 的三类一等对象，三栏布局能让用户持续保持方位感。

---

### 决策：后端 payload 按职责拆分

**为了谁**：服务于“聊天区展示完整用户视图”的用户故事。

**怎么设计**：后端拆成 Workspace Shell、Task Context、Task History；shell/context 不携带聊天 timeline。

**为什么这样**：避免一个大 snapshot 同时驱动全部 UI，造成状态抖动和重复缓存。

---

### 决策：运行态用事件增量驱动

**为了谁**：服务于工具反馈、等待态和右栏联动。

**怎么设计**：user_message_appended、turn_started、message_started、assistant_stream_delta、tool_started、tool_finished、message_finished、turn_finished、task_working_changed 等事件驱动前端更新。

**为什么这样**：真实 agent 运行是增量发生的，UI 应展示真实事件，而不是伪造“思考动画”。

---

### 决策：右栏是 AI 上下文投影

**为了谁**：服务于“右栏投影 AI 实际上下文”的用户故事。

**怎么设计**：右栏展示 Skills、Open Files、Notes、Memory、Runtime / Context Usage、Hints、Debug；Open Files 按材料清单而不是 IDE 树展示。

**为什么这样**：右栏不是普通侧边栏，而是让用户理解和操控 AI 下一轮材料的地方。

---

### 决策：设计系统三层 token 收敛

**为了谁**：服务于“视觉系统统一收敛”的用户故事。

**怎么设计**：primitive 定义原子值，semantic 定义语义颜色/间距，component 定义组件级消费规则。

**为什么这样**：UI 文档拆分后，视觉规则必须集中，否则组件实现会持续漂移。

---

### 决策：i18n 轻量自管理

**为了谁**：服务于国际化和桌面应用轻量化。

**怎么设计**：不用 vue-i18n，通过全局 composable 的 t() 和 TypeScript 嵌套语言对象管理。

**为什么这样**：MVP 阶段覆盖中文和英文即可，减少引入重量级框架带来的复杂度。
