# 扩展与后台认知设计

最后更新：2026-05-10

---

## 适用状态

这是旧版 Skills、Agents/Teams、Memory、Subconscious 架构资料。当前群聊协作里的 Agent 常驻、记忆和后台自动化，以 [Agent 编排器](agent-orchestrator.md) 和 [群聊上下文引擎](context-engine.md) 为准。

## 设计背景

这一设计服务于 [扩展与长期记忆需求](../requirements/extensions-and-memory.md)，也覆盖图片、浏览器等边界能力。目标是让扩展能力接入 March 的上下文分层，而不是另起一套不可控通道。

## 设计决策

### 决策：Skills 以 Markdown 文件形式发现和加载

**为了谁**：服务于“AI 能按任务加载技能”的用户故事。

**怎么设计**：injections 先注入技能索引；AI 需要时 open_file 读取具体 SKILL.md，watcher 继续负责更新。

**为什么这样**：技能可以像项目文件一样管理，不必一次性塞进上下文。

---

### 决策：Agent 是人格配置，不是进程

**为了谁**：服务于“用户可以召唤不同角色”的用户故事。

**怎么设计**：Agent 配置包含身份、描述、模型绑定、技能配置等；@mention 触发角色接话。

**为什么这样**：角色切换应该复用 March 的上下文架构，不应制造多个互不透明的进程。

---

### 决策：共享层和私有层隔离角色上下文

**为了谁**：服务于多角色协作的材料继承和工作隔离。

**怎么设计**：用户放上桌面的文件和笔记属于共享层；角色自己打开的文件和写的笔记属于私有层。

**为什么这样**：被 @ 的 Agent 需要立刻看到共同材料，但不能踩乱另一个角色的临时工作台。

---

### 决策：Subconscious 权限受限

**为了谁**：服务于“后台认知层自动整理模式”的用户故事。

**怎么设计**：Subconscious 可以操作 Memory 和 Hints，不能改文件、执行命令或侵入主 Agent 工作状态。

**为什么这样**：后台整理有价值，但必须保证不会暗中改变项目文件或用户当前任务。

---

### 决策：图片是消息级快照

**为了谁**：服务于多模态输入需求。

**怎么设计**：图片随消息进入 image content block，不进入 open_files 文件追踪；view_image 作为按需工具。

**为什么这样**：图片表达的是某个时间点的视觉材料，不是应随 watcher 更新的活文档。

---

### 决策：浏览器操作优先文本，截图按需

**为了谁**：服务于浏览器和桌面操作能力。

**怎么设计**：优先通过 CDP 获取结构化信息；需要视觉判断时再截图；操作用户真实浏览器环境。

**为什么这样**：文本和结构化 DOM 更稳定、更省上下文，截图适合视觉验证而不是默认读取方式。

---

### 决策：March CLI 先只接显式 pi extension 路径

**为了谁**：服务于 [pi 能力补齐 Roadmap](../roadmaps/_resolved/2026-05/pi-capability-parity.md) 阶段 3 的扩展系统，同时保护 March 自己的上下文重建、群聊 runtime 和 sidecar 边界。

**怎么设计**：

- 第一切片支持 `--extension <path>` / `-e <path>`，由用户显式指定本地 extension 文件或目录。
- 项目级自动发现只扫描 `.march/extensions/*.js|*.ts|*.mjs|*.cjs` 和 `.march/extensions/*/index.js|ts|mjs|cjs`，不扫描全局目录。
- CLI 将发现路径和显式路径解析为绝对路径后传给 pi runtime host 的 `additionalExtensionPaths`，不复用 pi 的 `.pi/extensions` 自动发现心智。
- 扩展加载只在默认 pi runtime host 路径生效；旧 direct/legacy session 路径遇到 `--extension` 必须直接报错，避免用户误以为扩展 hook 已接入。
- March 暂不暴露 hot reload、extension UI、extension command routing 和 March 专属 lifecycle hook。后续实现必须遵守下方 lifecycle 边界：区分 pi 单 Agent turn、March 群消息、WorkingSummary、commit/diff 和 AgentSessionRef。

**为什么这样**：pi extension runtime 有完整能力，但它默认围绕单 Agent interactive session 设计。March 的长期产品事实源是群、Agent、workspace 和 ContextEngine sidecar。先接显式路径能验证 SDK extension runner 组装，不会把自动发现、权限模型和群聊生命周期一次性混进 CLI。

---

### 决策：March 专属 lifecycle hook 必须先绑定产品事实源，再桥接 pi extension runtime

**为了谁**：服务于阶段 3 的扩展系统，同时保护默认 pi session 切换后已经稳定下来的会话、sidecar 和群聊协作边界。

**怎么设计**：

- March 不把 pi extension event name 直接当作产品 API。pi extension runtime 仍负责单 Agent turn 内部的模型、工具、压缩和 session 事件。
- March 专属 hook 分三层定义：
  - `pi-agent-turn`：由 pi runtime host 承载，包括单 Agent prompt、tool call、tool result、retry、compaction 和 session lifecycle。March 只读取 diagnostics，不重命名这些底层事件。
  - `march-agent-runtime`：由 March runner 承载，包括 AgentSessionRef 绑定、model/thinking 状态同步、ContextEngine sidecar 读写、`/resume-pi`、`/clone-pi`、`/fork-pi` 等 March 命令造成的 session replacement。
  - `march-collaboration`：由未来群聊 orchestrator 承载，包括 group message send/commit、被 @ Agent 的 turn 调度、WorkingSummary 更新、workspace diff/commit 摘要和群内共享/私有上下文裁剪。
- 第一版 March hook adapter 只允许读取稳定事实：group id、agent id、workspace id、pi session id、sidecar metadata、summary hash、diff metadata 和 diagnostics。不能直接读取另一个 Agent 的私有 ContextEngine 层，除非该层已被显式提升到共享上下文。
- 后台认知类 hook 默认只能写 memory/hints/diagnostics，不能改项目文件、执行 shell、切换 session 或提交 commit。
- 阻塞式 hook 必须显式标注 hook kind。默认非阻塞 hook 报错只进入 extension diagnostics，并通过 `/extensions` 可见，不能阻断消息持久化、sidecar 写入或 session resume。
- 会改变 March 状态的 hook 必须走 March 命令/runner 的既有入口，而不是绕过 ContextEngine、SessionManager 或 AgentSessionRef 直接写文件。
- Hook 配置属于 March 产品数据，未来应放在 `.march/` 下；pi JSONL 只保存 pi runtime session 历史，不保存 March hook 注册状态。

**为什么这样**：pi extension runner 是能力执行层，不是 March 的第二个编排器。先把 hook 绑定到 group、agent、workspace、sidecar 这些产品事实源，可以防止扩展绕过群聊隔离、私有上下文和 session replacement 一致性。这样后续实现 hook adapter 时，失败模型、权限边界和 diagnostics 入口都已经有明确验收标准。

**当前实现状态**：2026-05-10 已提供只读 March lifecycle adapter。它暴露稳定 facts、hook 分层和默认 policy，并通过 `/extensions` 展示；它支持 permission-gated 的内部 hook 注册/执行路径，也支持 `.march/extensions` 下声明式 `*.march-hooks.json` / `march-hooks.json` 绑定，但不二次 import/执行外部 extension 模块，也不执行文件写入、shell、session switch 或 commit 等副作用。
