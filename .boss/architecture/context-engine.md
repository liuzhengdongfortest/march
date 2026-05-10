# 群聊上下文引擎

最后更新：2026-05-07

## 决策：上下文按稳定性排序

**为了谁**：服务于“上下文不会腐烂”和 provider prefix cache。

**怎么设计**：构建顺序从低频变化到高频变化：system core、工具、群公告、群笔记、open files、记忆检索、最近群消息、工作记忆、draft。

**为什么这样**：稳定前缀更容易命中缓存，高频内容放在后面也更符合 Agent 对现场的理解顺序。

## 决策：群公共层和 Agent 私有层分离

**为了谁**：服务于多 Agent 协作。

**怎么设计**：群公告、群消息、群笔记属于公共层；Agent open files、draft、工作日志、私有记忆、自己的 WorkingSummary 属于私有层。

**为什么这样**：Agent 需要共享现场，但不能互相泄露全部内部工作脉络。

## 决策：自己的历史消息带 WorkingSummary

**为了谁**：服务于 Agent 自我恢复。

**怎么设计**：上下文中的群消息按原顺序渲染。自己的 Agent 消息前插入自己的 WorkingSummary；别人的 Agent 消息只显示 content。

**为什么这样**：Agent 需要知道“我上次为什么这么说”，但其他 Agent 的内部摘要既耗 token，也可能破坏角色边界。

## 决策：工作记忆存 jsonl，搜索走 ripgrep

**为了谁**：服务于可恢复和可排障。

**怎么设计**：工具调用、文件修改和中断事件写入 `.march/{username}/agent_logs/{agent_id}/{turn_id}.jsonl`。Agent 需要回顾时用搜索工具查日志。

**为什么这样**：日志是事实证据，不适合全部塞进 DB 或 prompt；文件系统搜索简单、透明、可调试。
