# 工作区与 Git 协议

最后更新：2026-05-07

## 决策：每个 Agent 使用独立 worktree

**为了谁**：服务于多 Agent 并发改代码。

**怎么设计**：群声明一个或多个本地 workspace。Agent 进入群后，为每个需要操作的 workspace 创建独立 git worktree branch。

**为什么这样**：共享工作目录会导致文件写入互踩；worktree 让每个 Agent 像一个独立开发者。

## 决策：工具必须被 March 包装

**为了谁**：服务于隔离和安全。

**怎么设计**：read/write/edit/bash 等工具解析路径时固定到 Agent worktree，工具事件写入该 Agent 日志。Pi 内置 bash 不直接开放给正式 Agent。

**为什么这样**：POC 已证明模型会主动 `cd` 到非预期目录；软提示不足以提供隔离。

## 决策：`send_group_message` 是交付边界

**为了谁**：服务于用户可追踪的协作汇报。

**怎么设计**：Agent 发群消息前，系统检查 git dirty 状态；有改动则要求 commit；随后生成 WorkingSummary，记录 commit before/after，消息落库并广播。

**为什么这样**：群消息是用户看到的承诺点，必须绑定代码状态和摘要，不能只是一段自然语言。

## 决策：diff UI 由 commit hash 驱动

**为了谁**：服务于代码审阅体验。

**怎么设计**：消息保存 `commit_hash_before` 和 `commit_hash_after`。二者不同则在气泡下显示可展开 diff，默认展示 stat。

**为什么这样**：用户能从群聊直接看到 Agent 这轮实际改了什么。
