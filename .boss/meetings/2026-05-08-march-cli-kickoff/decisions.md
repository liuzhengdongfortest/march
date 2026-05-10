# 决策记录

日期：2026-05-08

## 决策 1：独立分支开发

**决定**：在 `march-cli` 分支上开发 CLI 版，与 `master` 的 Web 版并行。

**原因**：CLI 版是独立产品壳，代码结构不同，不需要和 Web 版在同一分支上迭代。

---

## 决策 2：完整移植 nocturne_memory 架构

**决定**：不简化，完整将 nocturne 的 4 实体图模型、版本链、循环检测、孤儿防护、Glossary、Changeset 移植到 JavaScript。

**原因**：老板判断简化版会丢失关键设计精髓。nocturne 的架构是一个经过仔细设计的完整系统，各部分互相配合。

---

## 决策 3：记忆系统嵌入 CLI，不做独立服务

**决定**：记忆系统直接嵌入 March CLI 进程，存 `.march/memory.db`，不拆成独立 MCP Server。

**原因**：CLI 是单用户终端工具，不需要多进程架构。项目绑定存储足以满足需求。

---

## 决策 4：直接在当前目录操作文件

**决定**：不做 git worktree 隔离，Agent 在用户当前项目目录直接读写文件。

**原因**：CLI 场景下用户在终端里看着，改完自己 `git diff`。worktree 隔离是 Web 版多 Agent 协作场景的需求，CLI 不需要。

---

## 决策 5：命令名 march

**决定**：命令名 `march`，不叫 `march-cli`。

**原因**：老板指定。

---

## 决策 6：turn 摘要嵌入 recent_chat

**决定**：`turn_summary` 不独立成层，直接嵌在 `[recent_chat]` 每轮记录里，格式为 [用户消息] + [摘要] + [March 回复]。

**原因**：AI 只需要一个故事线来源，摘要和对话自然关联。March 在 turn 结束时强制 AI 调用 `send_turn_summary`，Pi compaction 做兜底。

---

## 决策 7：open_files 完整内联

**决定**：`[open_files]` 使用完整文件内容 + 绝对路径 + 行号，不做截断。

**原因**：open_files 是 AI 主动声明要持续关注的，数量有限；行号是精确编辑的基础；watcher 保证内容最新。

---

## 决策 8：edit_file 用行号区间

**决定**：`edit_file(path, oldString, newString)` 支持行号区间（`"55-64"` / `"55"`），不需要 AI 复制原文。行号模式匹配不上时 fallback 精确文本匹配。

**原因**：省 token；参考 openslimedit 的验证结果（省 11-45%）。`[open_files]` 的 watcher 保证行号基于最新内容展开，天然准确。

---

## 决策 9：edit_file 要求先 open_file

**决定**：`edit_file` 的前置条件是 path 已在 `[open_files]` 中，否则直接拒绝。不需要先 `read_file`。

**原因**：Pi 的 edit 需要先 read 拿到旧文本，March 文件已在上下文中。`open_file` = 数据模式，`read_file` = 参考模式。

---

## 决策 10：技能独立层 [active_skills]

**决定**：技能作为 `[active_skills]` 层，放在 `[session_status]` 之下、`[open_files]` 之上。`.march/skills/` 为技能池，AI 通过 `activate_skill` / `deactivate_skill` 动态开关。

**原因**：技能是行为指令而非数据，和 `[open_files]` 混在一起会被淹没。变化频率"几轮一次"，放在稳定层之下、常变层之上，cache 损失被下层吸收。

---

## 决策 11：session_status 上移

**决定**：`[session_status]` 放在 `[tools]` 之下、`[active_skills]` 之上。

**原因**：session 启动后不变，比 `[active_skills]` 更稳定，上移强化 prefix cache 收益。

---

## 决策 12：open_files 不限路径

**决定**：`open_file` 接受任意绝对路径，包括项目外文件和 skills 文件。

**原因**：AI 需要跨项目工作、需要打开 SKILL 参考。项目内走目录级 watcher，外部文件逐条单独 watch。

---

## 决策 13：固定文件机制

**决定**：`--pin` 启动参数和 REPL 内 `/pin` / `/unpin` / `/pins` 命令管理固定文件。固定文件在 `[runtime_status]` 提示，`close_file` 对它们无效。

**原因**：用户需要确保某些文件始终在 AI 上下文中。

---

## 决策 14：Pi 原生工具保留纯读类

**决定**：Pi 原生的 `read_file`、`list_files`、`glob`、`grep` 保留，不做 March 重新实现。March 只接管有状态的操作（edit_file、write_file、sandbox_bash、open_file/close_file）和 March 独有工具。

**原因**：纯读操作不改状态，Pi 原生实现已足够。
