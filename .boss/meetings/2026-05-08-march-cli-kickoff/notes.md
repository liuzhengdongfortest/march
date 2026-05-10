# 会议记录：March CLI 版启动

日期：2026-05-08

## 议题

开一条独立 git 分支，做 March 的 CLI 版——对标 Claude Code / Codex CLI 的终端原生编码 Agent。

## 讨论要点

### 产品定位

- 命令名 `march`，终端原生：`march "重构这个函数"` 单次执行，或 `march` 进入 REPL
- 基于 Pi SDK 做执行层（LLM loop、tool calling、streaming、compaction 全复用 Pi）
- 和 Web 版共享执行层，各自独立产品壳

### 核心差异化：上下文重建

来源文档 `.boss/source/doc/context.md` 的 9 层上下文分层结构：

```
[system_core]        ← 几乎不变
[injections]         ← MCP 等，session 启动时确定，不再变
[tools]              ← 工具定义，session 启动 1 次
[session_status]     ← March 写入，AI 只读；cwd、平台、shell、目录结构，session 启动后不变
[active_skills]      ← 当前装配的技能，AI 通过 activate_skill/deactivate_skill 开关，几轮一次
[open_files]         ← AI 通过 open_file/close_file 控制；文件内容带绝对路径+行号
[memory]             ← 完整移植 nocturne 的 4 实体图记忆系统
[runtime_status]     ← March 写入，AI 只读；当前时间、锁定文件、上下文压力，每轮变化
[hints]              ← 短命，每条独立 TTL
[recent_chat]        ← 最近 10 轮对话，每轮含 [用户消息] + [turn 摘要] + [March 回复]
```

每轮从项目当前事实重建上下文，不累积线性历史。

### 轮间稳定性：turn 摘要嵌入 recent_chat

担心每轮结束后轮内执行历史（读了什么文件、改了什么代码、跑了什么命令）全部丢弃，AI 下一轮变成黑箱。解决方案：**`turn_summary` 直接嵌在 `recent_chat` 的每轮记录里**。

```
[recent_chat]

Turn 3:
  [用户] 帮我给 auth 模块加测试
  [摘要] 读取 src/auth.ts、src/types.ts；新增 src/auth.test.ts (+45行)；npm test 3 passed
  [March] 已为 auth 模块添加单元测试，覆盖了 validateToken、refreshToken...

Turn 2:
  [用户] 重构一下认证逻辑
  [摘要] 读取 src/auth.ts；修改 src/auth.ts (+12/-3)；npm run build 通过
  [March] 已将认证逻辑抽成独立 middleware...
```

**生成机制**：March 在 turn 结束时强制 AI 调用 `send_turn_summary`（类似 Web 版的 `send_group_message`），把轮内事实（工具调用统计）拼成摘要 prompt，AI 确认后存入。不调这个工具 turn 不算结束。

**和 memory 的分工**：
- turn 摘要：March 强制生成，AI 确认，每轮覆盖，确保 AI 不会忘记刚才干了什么
- memory：AI 主动用 read_memory/create_memory/update_memory 管理的长期记忆，手动 remove 前一直存在

Pi compaction 机制做兜底：turn 太长导致 context 满时自动压缩轮内细节，外层摘要不受影响。

### 记忆系统：完整移植 nocturne_memory

老板要求完整拷贝 [nocturne_memory](https://github.com/Dataojitori/nocturne_memory) 的架构，不是简化学习。关键设计：

- **4 实体图**：Node（UUID 锚点）→ Memory（版本化内容）→ Edge（带 priority + disclosure）→ Path（URI 缓存）
- **版本链**：update 创建新 Memory → deprecate 旧版 → migrated_to 链支持 rollback
- **URI 路径体系**：`domain://path`，内容与路径分离，支持 alias
- **Disclosure 触发条件**：每条记忆绑定自然语言触发条件，非向量检索
- **Glossary**：Aho-Corasick 多模式匹配，关键词→节点自动超链接
- **启动协议**：`project://boot` 下的记忆 session 启动时自动加载
- **项目绑定**：记忆存 `.march/memory.db`，和项目目录绑定
- **7 个工具**：read_memory、create_memory、update_memory、delete_memory、add_alias、manage_triggers、search_memory

### 和上下文分层的融合

`[memory]` 层为完整移植 nocturne 的记忆系统：
- `project://boot` 子节点在 session 启动时注入 `[system_core]`
- disclosure 匹配当前对话关键词的记忆注入 `[memory]`
- `session://current/*` 全部注入 `[memory]`

### 当前目录直接干活

不做 worktree 隔离，Agent 在用户项目目录直接操作文件，更符合 CLI 心智。

### 技能系统：独立于 open_files

技能是行为指令，不是工作数据。放在 `[active_skills]` 层，和 `[open_files]` 分开：

- `.march/skills/` 目录下的 `.md` 文件为技能池，session 启动时扫描
- AI 通过 `activate_skill("code-review")` / `deactivate_skill("code-review")` 动态开关
- `--skill ~/my-review-skill.md` 手动注册额外技能
- 工具：`activate_skill`、`deactivate_skill`、`list_skills`
- AI 不能靠 `open_file` 把普通文件升级成技能

`[active_skills]` 放在 `[session_status]` 之下、`[open_files]` 之上——skill 开关几轮一次，不如 `session_status` 稳定，但比 `open_files`（每轮多次）稳定。

### 可固定文件

用户侧两个入口锁文件：
- `march --pin ~/.march/skills/review.md` 启动时固定
- REPL 内 `/pin src/auth.ts`、`/unpin`、`/pins` 运行时管理

固定文件在 `[runtime_status]` 提示，`close_file` 对它们无效。

### edit_file：行号区间参考 openslimedit

参考 [openslimedit](https://github.com/ASidorenkoCode/openslimedit) 的行号扩展方案，AI 不需要输出原文，直接指定行号：

```
edit_file(path, oldString, newString)

oldString 支持两种格式：
  "55-64"      → 替换 55-64 行
  "55"         → 替换第 55 行（单行）
  "import ..." → 精确文本匹配（现有行为，fallback）
```

工具内部用正则 `/^(\d+)(?:\\s*-\\s*(\\d+))?$/` 判断是否为行号模式，是则从文件读对应行展开为实际内容，再走精确替换。

**和 `[open_files]` watcher 的咬合**：AI 看到的文件内容是 watcher 实时追踪的最新版本，不存在"上次 read 之后文件被改过了"的竞态。AI 引用的行号基于唯一且最新的内容展开，天然准确。这个在 Claude Code / Codex CLI 中常见的失败场景，March 的上下文模型天然解决了。

**和 Pi 的差异**：Pi 的 edit 需要先 read 拿到原文。March 不需要——`[open_files]` 里的文件内容已在上下文中。如果 path 不在 `[open_files]`，直接拒绝并提示先 `open_file`。`read_file` 只用于看一眼就够的参考文件，不用于编辑前的准备。

## 技术选型

- 执行层：Pi SDK + DeepSeek
- 工具：复用现有 March 工具（list_files、read_file、write_file、edit_file、sandbox_bash）+ 新增记忆工具 + open_file/close_file
- 存储：node:sqlite，`.march/memory.db`
- 分支：`march-cli`（已创建并 checkout）
