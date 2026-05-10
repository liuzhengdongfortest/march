`.boss/` 是项目唯一权威文档层。
`.boss/source/` 保存迁入的原始资料快照（含旧 pi-go March 的架构文档），只做溯源，不作为当前决策入口。
项目文档默认使用中文。
从资料反推出来但未经老板确认的内容，必须标注为“推测”。
`.boss/requirements/VISION.md` 是当前设计总入口。
当前项目基于 OpenCode fork，技术栈为 Bun + TypeScript + Effect-TS。`.boss/source/` 里的旧 Node.js/pi-go 架构文档不代表当前实现。
当前优先级：愿景和产品范式 > 领域模型 > 差异化注入 > 上游追版。
需求文档使用用户故事格式：为什么、需要什么、怎么做。
架构文档使用设计决策格式：为了谁、怎么设计、为什么这样。
原始资料里的缺失链接或待补资料，进入 `.boss/tasks/` 追踪。
所有日期使用 Asia/Shanghai 的绝对日期。
会议目录模板为 `notes.md` + `decisions.md` 两个文件，不设 `action-items.md`。行动项走两条路径：大方向关联到 `.boss/roadmaps/`，具体任务由用户触发创建 `.boss/tasks/`。
临时调试产物、临时 state 目录、一次性验证脚手架不能留在项目根目录；验证后必须清理或纳入明确的 ignore/隔离目录。
MVP 可以先用轻量实现穿刺，但不能把调试用 HTML/单文件脚本长期当作正式架构；进入产品验证后必须及时升级到可维护的前后端工程结构。
API KEY在.env里
注意代码质量：单个源码文件超过 500 行或承担多类职责时，优先拆成独立内聚模块；不要为了最小 diff 继续往大文件里堆功能。
Task 体量：task 是有规划、有思想、有深度的工作单元，不是原子编辑的包装纸。禁止把单个文件拆分、单个函数提取、单个测试新增当作独立 task；同类小改动必须合并。一个 task 至少包含多个关联步骤，交付可感知的价值。如果 task 文档比代码改动还长，说明太碎。
20	本机 HTTP/HTTPS 代理地址为 `http://127.0.0.1:7890`。git clone 等需要外网访问的命令遇网络不通时，自动加上 `-c http.proxy=http://127.0.0.1:7890 -c https.proxy=http://127.0.0.1:7890`。
21	执行遇阻时不原地自旋。先尝试兼容/替代方案（如无法自建搜索后端则用 tavily/brave API，无法自建沙箱则用 Docker），方案确定后在 `.boss/` 留下汇报文件，说明原方案为什么不可行、替代方案是什么、取舍是什么。汇报文件放在对应 task 目录或 `.boss/wiki/`。
22	本项目是 `anomalyco/opencode` 的 fork。上游 remote: `git@github.com:anomalyco/opencode.git`。向上游提交 PR 是首选贡献路径；March 差异化代码优先通过 plugin/extension 机制隔离，避免直接 patch 核心模块。
23	运行时：Bun（`bun@1.3.13`），包管理：bun，语言：TypeScript（strict）。不引入 Node.js-only 依赖；如需 Node API，通过 OpenCode 已有的 `#pty` / `#db` 条件导入机制。
24	`.boss/source/` 下保存了旧 pi-go March 的架构设计文档（context-engine.md、shell-runtime.md 等），这些描述 March 三大差异化的设计思想，在实现 fork 改造时作为参考。
