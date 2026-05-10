# 工具与 Provider 设计

最后更新：2026-05-07

---

## 设计背景

这一设计服务于 [编码 Agent 能力需求](../requirements/coding-agent-capabilities.md)。March 要让 AI 像工程助手一样完成真实任务，但每个动作都要有清楚边界、结果生命周期和用户可见归因。

## 设计决策

### 决策：run_command 显式选择 shell

**为了谁**：服务于“AI 能调用本机开发环境”的用户故事。

**怎么设计**：run_command 输入包含 shell、command、timeout_secs；会话启动时扫描真实可用 shell 并注入当前工作目录。

**为什么这样**：PowerShell、cmd、bash 语法不同，显式选择能减少平台错误，也方便审计和重放。

---

### 决策：基础文件操作是一等工具

**为了谁**：服务于“AI 能稳定读写文件”的用户故事。

**怎么设计**：提供 open_file、close_file、write_file、replace_lines、insert_lines、delete_lines；修改工具返回 diff。

**为什么这样**：高频文件操作不应依赖 shell 拼接；直接接入 watcher 和 ModifiedBy 归因更稳定。

---

### 决策：行号级编辑优先于文本匹配

**为了谁**：服务于可审查、可定位的代码修改流程。

**怎么设计**：AI 看到带行号的文件内容，通过 replace_lines、insert_lines、delete_lines 定位。

**为什么这样**：减少文本匹配失败和跨平台转义问题；用户也更容易审查修改范围。

---

### 决策：LSP 是语义查询层

**为了谁**：服务于“AI 能使用代码语义能力”的用户故事。

**怎么设计**：提供 hover、goto_definition、find_references、code_action、rename、diagnostics。

**为什么这样**：文本上下文解决不了类型和引用关系；LSP 结果只在轮内可见，避免长期污染上下文。

---

### 决策：Provider 层不接管上下文

**为了谁**：服务于多模型供应商和 reasoning 能力。

**怎么设计**：March 构建统一上下文；WireAdapter 负责把它映射到不同 provider 的请求格式，并归一化响应。

**为什么这样**：上下文构建是 March 产品差异化，Provider 只应处理通信和能力差异。

---

### 决策：Turn 快照支持回退

**为了谁**：服务于“每轮修改可以回看和撤销”的用户故事。

**怎么设计**：每轮开始前保存快照；有 git 时用 git stash create，无 git 时保存 open_files 内容。

**为什么这样**：AI 修改需要低成本撤销，尤其在多文件变更时。
