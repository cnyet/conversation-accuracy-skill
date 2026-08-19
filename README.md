# Conversation Accuracy Skill

管理项目级会话记忆（`.agent/memory/`），在长会话与跨会话续接时保持上下文准确率。

**核心原则**：摘要优先（存提炼结论，不存对话原文）；最小加载（先必要，再扩展）；非阻塞持久化（保存后继续，不等上下文压缩）。

> 平台中立：遵循 Agent Skills 开放标准（frontmatter: `name`/`description`），目录约定 `.agent/memory/` 跨平台一致，可在支持 SKILL.md 的 agent（Claude Code、Codex、OpenClaw、Hermes 等）加载。
>
> **本 skill 本体（SKILL.md、`.agent/memory/` 约定、LOAD/PERSIST 流程、触发词）跨平台通用，不绑定任何特定 agent**。
>
> **范围标记**（全文通用）：
> - 🟢 **跨平台通用** — 所有支持 SKILL.md 的 agent 均生效
> - 🔴 **Claude Code 提供**（可选增强）— 仅 Claude Code 有该机制；**非 skill 必需**，不配置不影响核心功能，其他平台按各自等价机制处理

---

## 目录

1. [安装方式](#安装方式)
2. [使用方式（完整流程）](#使用方式完整流程)
3. [触发词列表](#触发词列表)
4. [平台能力对照](#平台能力对照)
5. [文件结构](#文件结构)

---

## 安装方式

### 🟢 方式一：SKILL.md 直接安装（跨平台通用）

本技能是一个标准 Agent Skill 目录，核心文件为 `SKILL.md`。任何遵循 Agent Skills 开放标准的 agent，将本目录放入其 skills 目录即可：

```bash
# 克隆到本地
git clone https://github.com/cnyet/conversation-accuracy-skill.git

# 放入对应平台的 skills 目录（按你的 agent 任选其一）
# Claude Code — 用户级（所有项目生效）
cp -r conversation-accuracy-skill ~/.claude/skills/
# Claude Code — 项目级（仅本项目生效）
cp -r conversation-accuracy-skill <项目>/.claude/skills/
# Codex
cp -r conversation-accuracy-skill ~/.codex/skills/
# OpenClaw
cp -r conversation-accuracy-skill ~/.openclaw/skills/
# Hermes
cp -r conversation-accuracy-skill ~/.hermes/skills/
```

> 💡 **单一事实源**：若同一台机器上多个 agent 都要用，建议采用 **symlink**——源目录放一处（如 `~/.agents/skills/`），各平台 skills 目录用软链接指向源，改源即同步到所有消费者，避免多份拷贝漂移。

### 🔴 方式二：Plugin marketplace 安装（Claude Code 专用渠道，可选）

> 本方式仅面向 Claude Code 用户，是 Claude Code 的**额外安装渠道**；skill 本体不依赖它。不用 plugin、直接走方式一，功能完全一致。

Claude Code 支持 plugin 机制：一个 marketplace 是一个 git 仓库，内含 `.claude-plugin/marketplace.json`（插件清单）+ 各插件目录（`plugin.json` + `skills/`）。

若本技能被打包进 marketplace（或将本仓库包装为 plugin 格式），安装方式为：

```bash
# 添加 marketplace（若尚未添加）
claude plugin marketplace add <marketplace-url>

# 安装插件（启用其中的 skill）
claude plugin install <marketplace>@conversation-accuracy-skill
```

> ⚠️ 本仓库当前为**纯 skill 仓库**（未含 `.claude-plugin/` 结构）。若要走 marketplace 路径，需先按 [Claude Code Plugin 规范](https://docs.anthropic.com/en/docs/claude-code/plugins) 包装。日常直接使用请用方式一（跨平台）。

### 🔴 可选增强：hook 自动注入（Claude Code 自动化，非必需）

Claude Code 可配置 hook 实现「会话开始自动注入 ACTIVE_CONTEXT.md」与「编辑后自动校验」，无需手动加载。**这是纯自动化增强——不配置 hook 也能在 Claude Code 正常使用本 skill（手动触发 LOAD/PERSIST），其他平台同理：**

```jsonc
// settings.local.json（机器级，建议放 local 层避免被 provider 切换重写）
{
  "hooks": {
    "SessionStart": [
      { "matcher": "", "hooks": [{ "type": "command", "command": "<path>/session-start.mjs" }] }
    ]
  }
}
```

> 🔴 hook 机制仅 Claude Code 提供，且仅为可选自动化。其他平台按其各自的启动加载机制处理（如 Codex 的启动钩子、OpenClaw 的插件生命周期），`SKILL.md` 核心流程不变。

---

## 使用方式（完整流程）

以「从零创建一个新项目」为例，走一遍记忆全流程。

### 第 1 步：初始化记忆目录（新项目首次）

新项目没有记忆目录时，对 Agent 说：

> 这个新项目还没建过记忆目录，你先帮我把上下文管理规范建立起来，以后我们就按这个来。

技能会执行：
1. 确认在项目根（`pwd`，找不到则向上搜索）
2. `mkdir -p .agent/memory/archive`
3. 按 `memory-templates.md` 模板创建 4 个初始文件：`ACTIVE_CONTEXT.md` / `session-summary.md` / `todo-tracker.md` / `user-preferences.md`

> 🟢 目录约定 `.agent/memory/` 跨平台一致，4 个文件结构通用。
> ⚪ 可选：仅当项目安装了 OpenViking 插件时，`user-preferences.md` 才包含「Session History Storage」双记忆系统章节；否则省略（与 agent 平台无关）。
> 🟢 兼容旧项目：目录缺失时回退读取遗留的 `.claude/memory/`（已废弃目录，仅历史包袱项目存在）——**读取支持回退，写入一律 `.agent/memory/`**，读到旧目录会提示迁移。

### 第 2 步：启动 / 续接会话（LOAD 加载记忆）

新会话开始或续接任务时，Agent 按需加载**最小记忆集**：

1. 检查 `.agent/memory/` 是否存在（不存在则回退找遗留 `.claude/memory/`）
2. 读最小集：`ACTIVE_CONTEXT.md` + `todo-tracker.md`
3. 仅当任务依赖旧决策 → 读 `session-summary.md`
4. 仅当用户偏好影响当前实现 → 读 `user-preferences.md`
5. **核对**用户声称的历史事实与记忆是否一致；不一致先指出差异再继续

> 🔴 Claude Code 若配置了 SessionStart hook（可选增强），`ACTIVE_CONTEXT.md` 会**自动注入**，无需手动要求恢复，也不再询问「是否恢复」；未配置则与其他平台一致，手动触发即可。
> 🟢 所有平台（含 Claude Code）：向 Agent 说「继续上次的工作」等触发语即可，Agent 按上述步骤加载。

### 第 3 步：工作中保存（PERSIST 保存上下文）

当对话变长、做出关键决策、或要结束会话时，触发保存。PERSIST 模式共 7 步：

1. 重读 `ACTIVE_CONTEXT.md` 与 `session-summary.md` 当前内容（**写前必 Read**）
2. diff 出本轮新增的决策/事实/坑点/待办，**不重写已存在内容**
3. 按模板追加到 `session-summary.md`（当日节不存在则新建 `## YYYY-MM-DD`）
4. 更新 `ACTIVE_CONTEXT.md` 为当前态（任务/状态/约束/阻塞/下一步），删除过期项
5. 更新 `todo-tracker.md` 状态
6. 检查归档触发（`## 日期节` > 50 → 最早的节移入 `archive/`）
7. 告知用户「上下文已保存」，**继续任务，不等上下文压缩**

**上下文占用临界值**（决定何时必须保存）：

| 上下文占用 | 动作 |
|-----------|------|
| < 60% | 正常运行，定期持久化 |
| 60–80% | PERSIST → 继续工作，建议压缩上下文 |
| ≥ 80% | 已持久化则继续；未持久化立即 PERSIST |
| ≥ 100% | 上下文压缩触发（🟢 各平台均有，机制不同），从项目记忆自动恢复 |

### 第 4 步：跨会话恢复（再次启动）

新会话开始后回到第 2 步。最小加载 → 核对历史 → 继续任务，整个过程不依赖人类手工翻记录。

### 全流程示意图

```
新项目 ──初始化──► .agent/memory/ 4 文件（archive/）
                      │
  新会话 ──LOAD──►    │ 读最小集（ACTIVE_CONTEXT + todo-tracker）
                      │  ├─ 需要旧决策 → +session-summary.md
                      │  └─ 偏好影响实现 → +user-preferences.md
                      ▼
  工作中 ──PERSIST──► 追加 session-summary.md + 更新 ACTIVE_CONTEXT/todo-tracker
                      │  └─ 节 >50 → 归档到 archive/
                      ▼
  会话结束 ──PERSIST─► 保存最终态（阻塞 + 下一步）
                      ▼
  下次启动 ──LOAD──►  自动恢复，继续任务
```

---

## 触发词列表

以下人类语言触发语会激活本技能（按 SKILL.md frontmatter 关键词 + 评测集整理）：

| 触发词 / 触发语 | 模式 | 动作 |
|:---|:---|:---|
| 新会话开始 / 续接任务 / 继续上次的工作 | **LOAD** | 恢复上下文 |
| 咱们继续昨天讨论的那个方案… | **LOAD** | 跨会话续接，先读记忆再动手 |
| 我们已经聊了 20 多轮了，重点有点漂移… | **PERSIST** | 长会话中期压缩，先整理关键决策 |
| 上下文占用高了 / 该保存一下进度了 | **PERSIST** | 立即保存上下文 |
| 会话即将结束 / 今天先到这 | **PERSIST** | 保存最终态后结束 |
| 这个新项目还没建过记忆目录… | **初始化** | 建 4 个初始文件 |
| 上次我们定的不是 B 方向吗？ | **LOAD（核对）** | 与记忆矛盾 → 先指出差异请用户确认，再继续 |
| 别管以前那些记录，直接开始… | **不适用** | 用户明确不要历史 → 跳过加载 |
| 这个老项目没建过 .agent/memory/，之前应该有历史… | **LOAD（回退）** | 回退读遗留 `.claude/memory/`，提示迁移 |
| 之前说过 / 下次继续 / 记忆恢复 | **触发词** | frontmatter 关键词，激活本技能 |
| session continuity / restore context | **触发词** | 英文等价关键词 |

---

## 平台能力对照

> skill 本体全部能力（`SKILL.md` 核心 + `.agent/memory/` 约定 + 流程 + 触发词）**跨平台一致**。以下仅列出因平台而异的**可选增强/渠道**——均非 skill 必需。

| 能力 | 范围 | 说明 |
|:---|:---|:---|
| `.agent/memory/` 目录约定 | 🟢 跨平台 | 所有平台目录一致 |
| SKILL.md frontmatter（name/description） | 🟢 跨平台 | Agent Skills 开放标准 |
| LOAD / PERSIST 核心流程 | 🟢 跨平台 | 平台无关，按需加载 + 摘要保存 |
| 触发词（人类语言） | 🟢 跨平台 | 与平台无关 |
| 遗留 `.claude/memory/` 回退兼容 | 🟢 跨平台 | 旧项目仅回退读取，写入一律新目录 |
| 上下文占用临界值（60/80/100%） | 🟢 跨平台 | 通用经验值，各平台按其机制处理 |
| SessionStart 自动注入 ACTIVE_CONTEXT | 🔴 Claude Code 提供 | 可选增强；其他平台按各自启动加载机制 |
| auto-compact 自动上下文压缩 | 🔴 Claude Code 提供 | 可选增强；其他平台各自的等价压缩机制 |
| Plugin marketplace 安装 | 🔴 Claude Code 提供 | 可选安装渠道；`claude plugin install` |
| hook（session-start / post-edit-tsc） | 🔴 Claude Code 提供 | 可选增强；settings.local.json 注册，机器级 |
| Session History Storage（OpenViking 集成） | 可选 | 仅项目安装 openviking 插件时启用 |

---

## 文件结构

```text
conversation-accuracy-skill/
├── SKILL.md               # 技能主文件（模式判定 / LOAD / PERSIST / 决策表 / 异常 fallback）
├── memory-templates.md    # 四文件模板 + 质量自查
├── test-prompts.json      # 评测集（LOAD / PERSIST / 初始化 / 回退 6 场景）
├── README.md              # 本文档
└── LICENSE                # MIT License（Copyright 2026 cnyet）
```

技能运行时的记忆目录（在**使用技能的项目**里，非本仓库）：

```text
<项目>/.agent/memory/
├── ACTIVE_CONTEXT.md      # 当前态（任务/状态/约束/阻塞/下一步）
├── session-summary.md     # 高价值历史（决策/事实/坑点/待办）
├── todo-tracker.md        # 任务状态（In Progress/Pending/Blocked/Done）
├── user-preferences.md    # 长期稳定偏好
└── archive/               # 归档的旧日期节（保留最近 10 个文件）
```
