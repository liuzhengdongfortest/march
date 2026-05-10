# 上下文核心设计

最后更新：2026-05-07

---

## 适用状态

这是旧 Task / Session 阶段的上下文架构资料。原则继续有效，当前实现应迁移到 [群聊上下文引擎](context-engine.md)。

## 设计背景

这一设计服务于 [上下文工作台需求](../requirements/context-workbench.md)。March 的核心卖点是“上下文不会腐烂”，所以架构必须把当前磁盘事实、AI 主动记忆、运行时状态和最近对话拆开管理。

## 设计决策

### 决策：Task 持久化，Session 运行时化

**为了谁**：服务于“一个主题就是一个任务窗口”的用户故事。

**怎么设计**：Task 存 working_directory、model 参数、recent_chat、notes、open_files；Session 是 App 加载 Task 后的运行实例。

**为什么这样**：用户关心的是主题能被恢复和继续，不需要理解内部 session 生命周期。Subconscious 计数、session_start、session_end 等运行概念可以独立处理。

---

### 决策：上下文按变化频率排序

**为了谁**：服务于“上下文按稳定性分层”的用户故事。

**怎么设计**：从稳定到高频依次是 system_core、injections、tools、open_files、notes、session_status、runtime_status、hints、recent_chat。

**为什么这样**：稳定前缀更容易命中 cache；高频状态靠后，不拖累前面层级。

---

### 决策：open_files 是材料清单，不是历史读取记录

**为了谁**：服务于“AI 上下文反映当前磁盘事实”的用户故事。

**怎么设计**：open_files 只持久化路径、顺序和锁定状态，文件内容由 watcher 实时刷新；删除、移动、重命名通过 FileSnapshot 和 Hint 暴露给 AI。

**为什么这样**：文件系统变化是事实，AI 需要知道变化而不是静默丢失文件。

---

### 决策：Notes 用稳定 id 覆盖更新

**为了谁**：服务于“AI 有可管理的工作记忆”的用户故事。

**怎么设计**：write_note(id, content) 是 upsert；target、plan、build_output 等信息复用稳定 id。

**为什么这样**：避免上下文里出现多条语义接近但相互矛盾的旧笔记。

---

### 决策：Hints 是短命外部注入

**为了谁**：服务于“外部事件能短暂提醒 AI”的用户故事。

**怎么设计**：Hints 同时支持 ttl_secs 和 ttl_turns，过期前进入上下文，过期后清理。

**为什么这样**：外部通知应该能影响近期判断，但不该变成长期记忆。

---

### 决策：Memory 和 Notes 分层

**为了谁**：服务于 [扩展与长期记忆需求](../requirements/extensions-and-memory.md)。

**怎么设计**：Notes 是 task 内工作记忆；Memory 是跨 session、跨 task 的长期知识层，项目级文件存储，全局级数据库存储。

**为什么这样**：短期计划和长期偏好生命周期不同，混在一起会同时伤害上下文清晰度和用户管理体验。
