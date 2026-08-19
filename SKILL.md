---
name: conversation-accuracy-skill
description: "管理项目级会话记忆（.agent/memory/）：PERSIST 保存关键决策 + LOAD 按需恢复历史，保持长会话与跨会话上下文准确率。触发词：长会话, 记忆恢复, 跨会话续接, 上下文压缩, 之前说过, 下次继续, session continuity, restore context."
---

# Conversation Accuracy Skill

管理项目级会话记忆（`.agent/memory/`），在长会话与跨会话续接时保持上下文准确率。

**核心原则**：摘要优先（存提炼结论，不存对话原文）；最小加载（先必要，再扩展）；非阻塞持久化（保存后继续，不等上下文压缩）。

> **平台中立**：遵循 Agent Skills 开放标准（frontmatter: name/description)，目录约定 `.agent/memory/` 跨平台一致，可在支持 SKILL.md 的 agent（Claude Code、Gemini、Codex、OpenClaw 等）加载。Claude 专属机制（SessionStart hook、auto-compact）在其他平台按各自等价机制处理。

> **历史兼容**：目录解析顺序为 `.agent/memory/` **优先**，不存在时**回退**读取遗留的 `.claude/memory/`（已废弃目录，仅历史包袱项目存在）。读取方向支持回退；**写入方向一律 `.agent/memory/`**——读到遗留目录时提示用户顺手迁移到新目录，绝不继续写入或新建 `.claude/memory/`。

## 模式判定（进入先选，二选一）

| 触发场景 | 模式 |
|---------|------|
| 新会话开始 / 续接任务 / 需追溯历史决策 | **LOAD** — 恢复上下文 |
| 上下文占用 ≥ 60% / 对话 ≥ 8 轮 / 完成关键决策 / 会话结束 | **PERSIST** — 保存上下文 |

两者同时满足（长会话中续接）：先 PERSIST 再 LOAD。

**不适用**：一次性短任务与历史无关；项目无 `.agent/memory/` 且用户未要求初始化；加载历史只会引入噪音时。

## LOAD 模式（恢复）

1. 检查 `.agent/memory/` 是否存在；不存在则回退找遗留 `.claude/memory/`（废弃目录，仅兼容旧项目）；不在项目根则 `pwd` 向上搜到含该目录的根
2. 读最小集：`ACTIVE_CONTEXT.md` + `todo-tracker.md`
3. 仅当任务依赖旧决策 → 读 `session-summary.md`
4. 仅当用户偏好影响当前实现 → 读 `user-preferences.md`
5. 核对用户声称的历史事实与记忆是否一致；不一致先指出差异再继续

> **平台注**：Claude Code 的 SessionStart hook 会自动注入 `ACTIVE_CONTEXT.md`；已注入则不重复加载、不再询问「是否恢复」。其他平台按其各自的启动加载机制处理。

## PERSIST 模式（保存）— 7 步清单

1. 重读 `ACTIVE_CONTEXT.md` 与 `session-summary.md` 当前内容（**写前必 Read**）
2. diff 出本轮新增的决策/事实/坑点/待办，不重写已存在内容
3. 按 `memory-templates.md` 模板追加到 `session-summary.md`；当日节不存在则新建 `## YYYY-MM-DD`
4. 更新 `ACTIVE_CONTEXT.md` 为当前态（任务/状态/约束/阻塞/下一步），删除过期项
5. 更新 `todo-tracker.md` 状态
6. 检查归档触发（`## 日期节` > 50 → 最早的节移入 `archive/`）
7. 告知用户「上下文已保存」，继续任务（不等待上下文压缩）

### 上下文占用临界值矩阵

| 上下文占用 | 动作 |
|-----------|------|
| < 60% | 正常运行，定期持久化 |
| 60-80% | PERSIST → 继续工作，建议压缩上下文 |
| ≥ 80% | 已持久化则继续；未持久化立即 PERSIST |
| ≥ 100% | 上下文压缩触发（各平台机制不同），从项目记忆自动恢复 |

## 决策表（场景 → 动作）

| 场景 | 动作 |
|------|------|
| 新的短任务（一次性） | 不加载历史 |
| 延续已有任务 | LOAD 步骤 1-2 |
| 需要旧决策 | LOAD 步骤 1-3 |
| 用户偏好影响方案 | LOAD 步骤 1-4 |
| 上下文开始漂移 | PERSIST → 继续 |
| 会话即将结束 | PERSIST |
| 新项目首次初始化 | 见「初始化」 |

## 初始化与回退

### Phase 0 预检
1. `.agent/memory/` 不存在：先查遗留 `.claude/memory/`——存在则回退读取并提示迁移；用户要求初始化 → `mkdir -p .agent/memory/archive` 并按模板建 4 个文件（**只建新目录，不建废弃目录**）；未要求 → 仅提醒，不创建
2. 确认在项目根（`pwd`），否则向上搜
3. 4 个核心文件缺失 → 按 `memory-templates.md` 补空壳

### 异常 fallback

| 场景 | 处理 |
|------|------|
| 目录不存在 & 用户未要求 | 不创建，仅提醒 |
| 找到遗留 `.claude/memory/`（废弃目录） | 回退读取历史；提示迁移到 `.agent/memory/`，不继续写入旧目录 |
| `ACTIVE_CONTEXT.md` 为空 | 开新任务，不强制加载其他文件 |
| 模板文件损坏（JSON/表格乱码） | 备份 `<file>.bak.YYYYMMDD-HHMM`，重建后告知 |
| 跨项目引用错误记忆目录 | 先 `pwd` 确认项目根，不写到家目录全局记忆（`~/.claude/`、`~/.agents/`） |
| 多 Agent 并行写同一文件 | 写前必 Read（RMW）；同一文件禁止并行写 |

## 记忆目录结构

```text
.agent/memory/
├── ACTIVE_CONTEXT.md      # 当前态（任务/状态/约束/阻塞/下一步）
├── session-summary.md     # 高价值历史（决策/事实/坑点/待办）
├── todo-tracker.md        # 任务状态（In Progress/Pending/Blocked/Done）
├── user-preferences.md    # 长期稳定偏好
└── archive/               # 归档的旧日期节
```

> 兼容：旧项目遗留 `.claude/memory/`（迁移前约定）仅在 LOAD 时回退读取；新写入一律 `.agent/memory/`。

## 文件规范

- **时间戳**：节标题用日期 `2026-08-19`；`Last updated` 用 UTC `2026-08-19T14:30:00Z`
- **归档**：`session-summary.md` 的 `## YYYY-MM-DD` 节 > 50 时，最早的节移至 `archive/`；archive 保留最近 10 个文件
- **安全**：禁止把 API key / 密码 / token 写入 memory 文件；必要时只存位置引用
- **模板**：结构与章节见 `memory-templates.md`（含「Session History Storage」声明；其中 OpenViking 章节**仅在项目安装 openviking 插件时启用**）

## 核心模式

**推荐**：先读最小集再继续；决策即时 PERSIST；变长时先摘要再扩展；只追加不覆盖。
**不推荐**：每次加载全部记忆；反复读同一文件；用原文长对话替代摘要；只追加不归档。

## 支撑文件

- `memory-templates.md` — 四文件模板与质量自查
- `test-prompts.json` — 评测集（LOAD / PERSIST / 初始化三场景）
