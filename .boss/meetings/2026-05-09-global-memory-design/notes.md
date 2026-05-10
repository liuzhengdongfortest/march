# 全局记忆设计 — 会议记录

日期：2026-05-09

## 现状

- 记忆系统是项目级 SQLite（`cwd/.march/memory.db`），4 实体图模型（nodes/edges/paths/memories）
- 没有全局记忆，跨项目知识无法复用
- Skills 系统已有全局 + 项目的双层结构（`~/.march/skills/` + `.march/skills/`），记忆系统缺这一层

## 讨论要点

1. 全局记忆应该存在哪里：决定统一到一个全局 DB（`~/.march/memory.db`），不再每个项目各自建 DB
2. 项目隔离方式：`.march/project-id` 文件存 UUID，随项目迁移/clone。比路径做 key 更稳定，比 commit hash 更持久
3. namespace 模型：`project-id` = 项目记忆，`"global"` = 全局记忆。两层同时存在，查找时合并结果
4. 对 Agent 工具暴露 scope 参数（project/global），写入时指定，读取时合并
5. 现有 schema 已有 `namespace` 字段（paths、glossary_keywords、search_documents、memory_access_log），不需要改表结构
