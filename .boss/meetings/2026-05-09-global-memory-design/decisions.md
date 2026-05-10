# 全局记忆设计 — 决策

日期：2026-05-09

## 决策 1：统一全局 DB

**决定**：记忆数据库从 `cwd/.march/memory.db` 迁移到 `~/.march/memory.db`，所有项目共享一个 DB 文件。

**理由**：一个用户的所有记忆集中管理，避免每个项目各自维护 DB 的碎片化。

## 决策 2：project-id 做 namespace 隔离

**决定**：`.march/project-id` 文件存储一个 UUID，作为该项目的 namespace。DB 中所有查询通过 namespace 过滤实现项目隔离。

**理由**：UUID 跟随 `.march/` 目录，git clone 和目录迁移都能保持关联。比绝对路径更稳定。

## 决策 3：两层 namespace

**决定**：
- 项目记忆使用 `namespace = <project-id>`，只在本项目可见
- 全局记忆使用 `namespace = "global"`，所有项目可见
- 查找时同时检索 project-id 和 "global" 两个 namespace，合并结果

**理由**：既隔离项目专属发现，又保留跨项目可复用的知识。

## 决策 4：工具层暴露 scope 参数

**决定**：`create_memory` / `update_memory` 增加 `scope` 参数（`"project"` | `"global"`），默认 `"project"`。`read_memory` / `search_memory` 自动合并两层。

**理由**：默认安全（写入隔离到项目），读取自然合并。

## 决策 5：需修的实现泄漏点

以下模块需加 namespace 过滤：`GraphService.getChildren`、`GlossaryService`、`SystemViews.boot/index/diagnostic`。
