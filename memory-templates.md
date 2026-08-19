# 记忆模板

保持记忆精简、有效，并便于重新加载。

## ACTIVE_CONTEXT.md

```markdown
# Active Context

> Last updated: 2026-03-06T10:00:00Z

## Current Task
- In progress: {任务名称}
- Goal: {成功标准}
- Next step: {下一步具体动作}
- Blockers: {无 | 阻塞说明}

## Current Constraints
- {当前有效规则或边界}
- {当前技术限制}
- {当前范围约束}

## Current Decisions
- {决策} | Reason: {简短原因}
- {决策} | Reason: {简短原因}

## Current References
- `{重要文件或文档}`
- `{重要文件或文档}`
```

规则：
- 只保留当前状态
- 保持简短
- 删除过期项，不要不断追加历史

## session-summary.md

```markdown
# Session Summary

## 2026-03-06

### Completed
- [x] {已完成结果}
- [x] {已完成结果}

### Key Decisions
- {决策} | Reason: {选择原因}
- {决策} | Reason: {选择原因}

### Important Findings
- {事实、行为或实现细节}
- {事实、行为或实现细节}

### Pitfalls
- {问题} | Fix: {解决方式}
- {问题} | Fix: {解决方式}

### Carry-over
- [ ] {后续任务}
- [ ] {后续任务}
```

规则：
- 事实和决策优先于叙述
- 只追加高价值信息
- 不记录冗长对话原文

## todo-tracker.md

```markdown
# Todo Tracker

## In Progress
- [ ] {任务} | Priority: P0 | Started: 2026-03-06
- [ ] {任务} | Priority: P1 | Started: 2026-03-06

## Pending
- [ ] {任务} | Priority: P1 | Created: 2026-03-06
- [ ] {任务} | Priority: P2 | Created: 2026-03-06

## Blocked
- [ ] {任务} | Blocker: {原因}

## Done
- [x] {任务} | Completed: 2026-03-06
- [x] {任务} | Completed: 2026-03-06
```

规则：
- 保持状态最新
- 只用于任务状态，不记录详细推理
- 及时删除过期或已取消事项

## user-preferences.md

> 注：「Session History Storage」章节为**可选集成**——仅在项目安装 OpenViking 插件时启用；否则删除该章节，直接从 `## Communication` 开始。

```markdown
# User Preferences

## Session History Storage

本项目采用 **双记忆系统**，两层独立共存、各司其职：

| 存储层 | 触发方式 | 存储内容 | 检索方式 |
|--------|----------|----------|----------|
| **OpenViking 向量数据库** | AI 自动保存 | 执行过程中的原始对话历史、发现和决策轨迹 | 向量语义搜索（跨 Agent 共享：Claude Code、OpenClaw、Hermes、Codex） |
| **.agent/memory/**（项目根目录） | 用户手动触发 `conversation-accuracy-skill` | 提炼后的精要记忆文件（ACTIVE_CONTEXT.md、todo-tracker.md、session-summary.md、user-preferences.md） | 文件系统直接读取 |

**职责分工**：
- OpenViking 存 **原始对话** — 全量、语义可检索、跨 Agent 共享
- .agent/memory/ 存 **提炼事实** — 精简、结构化、快速恢复上下文

**不冗余原则**：不应将 OpenViking 中已有的原始对话全文复制到 .agent/memory/；memory 文件只保存提炼后的结论、决策和偏好。

## Communication
- {偏好}
- {偏好}

## Coding Style
- {偏好}
- {偏好}

## Workflow
- {偏好}
- {偏好}

## Project Conventions
- {偏好}
- {偏好}
```

规则：
- 只保存长期稳定的偏好
- 不记录一次性请求
- 表述保持具体且可复用
- 若安装 openviking 插件，则在开头包含 "Session History Storage" 章节声明双记忆系统职责分工；否则省略

## 质量检查

- 这些内容在下次会话里是否仍然相关？
- 这是不是当前最短但仍有用的版本？
- 这属于事实、决策、阻塞还是下一步？
- 它应该放在 active context、summary、todo 还是 preferences？
- 旧内容能否删除或归档，而不是继续追加？
