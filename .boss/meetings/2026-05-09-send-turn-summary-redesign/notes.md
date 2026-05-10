# 会议记录

日期：2026-05-09

主题：send_turn_summary 工具重新设计

## 讨论要点

### 现状问题

- 模型需要主动记得调用 `send_turn_summary` 工具，增加认知负担
- 忘记调用时触发 enforcement 轮次，期间 UI 完全静默，用户体验差
- 工具定义、system_core 指令、enforcement 三道防线，说明前两道不够可靠
- enforcement fallback 取 draft 前 300 字符，摘要质量可能很低

### 老板意图

- 模型不应该承担"记得调工具"的责任——这是 March 自己的节奏控制
- 流程应该是：模型自然结束回复 → March 注入总结提示 → 模型纯文本输出一段总结
- 总结轮不给模型任何工具（或极受限）
- 用户不感知总结内容，只看到轻量的 UI 状态标记（"summarizing..." → "done"）
- 总结 token 上限 1k
