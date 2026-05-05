---
name: wiki
description: "本地 Wiki 记忆系统。当用户提到'wiki'、'记录经验'、'写wiki'、'查知识'、'知识库'、'记笔记'、'write wiki'、'search wiki'、'knowledge base'、'反向链接'、'backlinks'、'wiki index'、'更新索引'、'知识检索'、'查看wiki'、'wiki页面'、'添加知识'、'经验总结'等关键词，或明确要求创建/查询/更新/浏览 Wiki 页面时使用此技能。"
argument-hint: <操作> [页面名/关键词] [--tags tag1,tag2] [--related page1,page2]
---

# 本地 Wiki 记忆系统

基于 Markdown + `[[反向链接]]` 的轻量级本地知识库。所有数据存储为纯文本文件，无外部依赖，兼容 Obsidian 语法。

用户输入：$ARGUMENTS

---

## 核心概念

### 目录结构

```
<project-root>/wiki/
  /pages/          # 知识页面（长期沉淀，持续更新）
    auth-system.md
    jwt-issues.md
  /logs/           # 任务执行日志（归档，不再修改）
    2026-04-06_user-auth.md
  index.md         # 自动维护的索引 + 链接图
```

### 页面类型区分

| 类型 | 目录 | 内容 | 生命周期 |
|------|------|------|----------|
| **Page** | `wiki/pages/` | 提炼后的知识（经验、方案、模式、踩坑记录） | 长期，持续更新 |
| **Log** | `wiki/logs/` | 任务执行日志（做了什么、结果如何、时间线） | 归档，一般不修改 |

### 反向链接规则

- 使用 `[[Page Name]]` 语法（Obsidian 兼容）
- Page Name 即文件名（不含 `.md`），使用 kebab-case
- 例如：`[[jwt-issues]]` 指向 `wiki/pages/jwt-issues.md`
- Log 也可被链接：`[[log:2026-04-06_user-auth]]` 指向 `wiki/logs/2026-04-06_user-auth.md`

---

## 操作模式

根据 `$ARGUMENTS` 判断用户意图，执行对应操作：

### 模式 1：创建页面 — `wiki create <page-name>`

触发词：create、创建、新建、添加、write、写

#### Phase 1：初始化 Wiki 目录

```bash
mkdir -p wiki/pages wiki/logs
# 如果 index.md 不存在，创建初始索引
if [ ! -f wiki/index.md ]; then
  echo "初始化 wiki/index.md"
fi
```

#### Phase 2：生成页面内容

页面名从 `$ARGUMENTS` 提取，转为 kebab-case。

**Page Schema**

所有 Page 共享以下 frontmatter，`type` 字段必填，它决定正文的 Section 结构：

```yaml
---
# === 必填 ===
title: <string>
type: pitfall | solution | pattern | decision   # 经验类型
created: <YYYY-MM-DD HH:mm>
updated: <YYYY-MM-DD HH:mm>
confidence: high | medium | low

# === 推荐填 ===
tags: [<string>]
tech_stack: [<string>]          # 技术栈，供检索过滤
source_task: <string>           # 来源路线图项目代号

# === 自动填（由 wiki-execute / wiki-replan 写入时自动加）===
origin: wiki-execute | wiki-replan | manual
---
```

**`type` 对应的正文 Section（按类型必填）**：

`type: pitfall`（踩坑记录）：
```markdown
# <Page Title>

## 触发条件
在什么操作/场景下会触发这个问题

## 现象
具体报错、异常行为描述

## 根因
为什么会发生

## 规避方法
怎么避开或修复

## 相关链接
- [[related-page]]
```

`type: solution`（解决方案）：
```markdown
# <Page Title>

## 问题描述
解决的是什么问题

## 方案
具体实现步骤或代码

## 验证结果
怎么验证它有效

## 局限性
在哪些场景下不适用

## 相关链接
- [[related-page]]
```

`type: pattern`（可复用模式）：
```markdown
# <Page Title>

## 适用场景
什么情况下用这个模式

## 实现方式
核心做法

## 示例
具体例子

## 局限性
不适用的情况

## 相关链接
- [[related-page]]
```

`type: decision`（架构决策）：
```markdown
# <Page Title>

## 背景
为什么要做这个决策

## 选项对比
| 选项 | 优点 | 缺点 |
|------|------|------|
| A    |      |      |
| B    |      |      |

## 最终决策
选了什么

## 理由
为什么这样选

## 相关链接
- [[related-page]]
```

**Log 模板**（写入 `wiki/logs/<date>_<task-name>.md`）：

```markdown
---
title: "任务日志: <task-name>"
date: <YYYY-MM-DD>
task: <关联的路线图/任务名>
status: <in_progress/completed/failed>
tags: [log, <其他>]
---

# 任务日志: <task-name>

## 执行时间线

| 时间 | 步骤 | 结果 | 备注 |
|------|------|------|------|
| | | | |

## 关键发现

- 

## 关联知识

- [[related-page]]
```

如果 `$ARGUMENTS` 包含 `--tags`，使用指定标签。
如果 `$ARGUMENTS` 包含 `--related`，在页面底部添加相关链接。

#### Phase 3：更新索引

创建/更新页面后，**必须**执行索引更新（见"索引维护"章节）。

#### Phase 4：输出确认

```
╔══════════════════════════════════════════╗
║  📝 Wiki 页面已创建                       ║
╠══════════════════════════════════════════╣
║                                          ║
║  页面: wiki/pages/<page-name>.md         ║
║  标签: <tags>                            ║
║  链接: <outgoing links count> 个出链     ║
║                                          ║
║  💡 提示:                                ║
║  - /wiki search <关键词> 搜索知识         ║
║  - /wiki links <page> 查看链接关系        ║
╚══════════════════════════════════════════╝
```

---

### 模式 2：搜索/检索 — `wiki search <关键词>`

触发词：search、搜索、查找、find、查、检索、retrieve

#### 检索策略（四层递进）

**Layer 1 — 标题精确匹配**：

```bash
# 关键词匹配文件名
ls wiki/pages/ | grep -i "<keyword>"
```

**Layer 2 — 标签匹配**：

```bash
# 在 frontmatter 的 tags 中搜索
grep -rl "tags:.*<keyword>" wiki/pages/
```

**Layer 3 — 全文搜索**：

```bash
# 搜索页面正文
grep -rl "<keyword>" wiki/pages/ wiki/logs/
```

**Layer 4 — 链接追踪**：

找到匹配的页面后，通过 `index.md` 中的 Backlinks 部分，扩展到关联页面。

```bash
# 从 index.md 中提取目标页面的反向链接
grep "<matched-page>" wiki/index.md
```

#### 输出格式

```
╔══════════════════════════════════════════╗
║  🔍 Wiki 搜索结果: "<keyword>"           ║
╠══════════════════════════════════════════╣
║                                          ║
║  📄 精确匹配:                             ║
║    1. jwt-issues (tags: auth, jwt)       ║
║       → JWT Token 过期处理和刷新策略      ║
║                                          ║
║  🔗 关联页面 (via backlinks):             ║
║    2. auth-system ← [[jwt-issues]]       ║
║    3. security-best-practices            ║
║                                          ║
║  📝 日志匹配:                             ║
║    4. 2026-04-06_user-auth (提及3次)      ║
║                                          ║
║  💡 /wiki read jwt-issues 查看详情        ║
╚══════════════════════════════════════════╝
```

---

### 模式 3：查看页面 — `wiki read <page-name>`

触发词：read、查看、打开、show、open、看

1. 读取 `wiki/pages/<page-name>.md` 的完整内容
2. 从 `index.md` 提取该页面的反向链接
3. 展示内容 + 反向链接列表

```
╔══════════════════════════════════════════╗
║  📖 <Page Title>                         ║
╠══════════════════════════════════════════╣
║                                          ║
║  <页面正文内容>                            ║
║                                          ║
║  ─── 反向链接 ───                         ║
║  ← [[auth-system]] (提及此页)             ║
║  ← [[log:2026-04-06_user-auth]]          ║
║                                          ║
╚══════════════════════════════════════════╝
```

---

### 模式 4：更新页面 — `wiki update <page-name>`

触发词：update、更新、修改、编辑、追加、append

1. 读取现有页面内容
2. 根据用户指示修改/追加内容
3. 更新 frontmatter 的 `updated` 字段
4. 如果新增了 `[[链接]]`，重建索引
5. 输出更新确认

---

### 模式 5：查看链接关系 — `wiki links <page-name>`

触发词：links、链接、关系、graph、关系图、backlinks

```bash
# 提取出链（当前页面链接了谁）
grep -oP '\[\[([^\]]+)\]\]' "wiki/pages/<page-name>.md" | sort -u

# 提取入链（谁链接了当前页面）— 从 index.md 读取
grep "^- <page-name> ←" wiki/index.md
```

输出：

```
╔══════════════════════════════════════════╗
║  🕸️ 链接关系: <page-name>               ║
╠══════════════════════════════════════════╣
║                                          ║
║  → 出链 (此页链接了):                     ║
║    → [[jwt-issues]]                      ║
║    → [[session-management]]              ║
║                                          ║
║  ← 入链 (链接了此页):                     ║
║    ← [[api-gateway]]                     ║
║    ← [[log:2026-04-06_user-auth]]        ║
║                                          ║
║  🔗 链接深度 2 (间接关联):                 ║
║    jwt-issues → [[token-refresh]]        ║
║    session-management → [[redis-config]] ║
║                                          ║
╚══════════════════════════════════════════╝
```

---

### 模式 6：浏览索引 — `wiki index` 或 `wiki list`

触发词：index、索引、列表、list、目录、overview、总览

读取并展示 `wiki/index.md` 的内容。如果索引不存在或过期，先执行重建。

---

## 索引维护

这是 Wiki 系统的核心机制。每次创建/更新/删除页面后必须执行。

### 索引重建流程

```bash
# 1. 扫描所有页面，提取 title 和 tags
for f in wiki/pages/*.md; do
  TITLE=$(grep '^title:' "$f" | head -1 | sed 's/title: *//' | tr -d '"')
  TAGS=$(grep '^tags:' "$f" | head -1 | sed 's/tags: *//')
  UPDATED=$(grep '^updated:' "$f" | head -1 | sed 's/updated: *//')
  NAME=$(basename "$f" .md)
  echo "- [[$NAME]] — $TITLE ($TAGS) — $UPDATED"
done

# 2. 扫描所有页面中的 [[links]]，构建链接图
for f in wiki/pages/*.md wiki/logs/*.md; do
  FROM=$(basename "$f" .md)
  LINKS=$(grep -oP '\[\[([^\]]+)\]\]' "$f" 2>/dev/null | sed 's/\[\[//;s/\]\]//' | sort -u)
  if [ -n "$LINKS" ]; then
    echo "$FROM → $(echo $LINKS | tr '\n' ', ')"
  fi
done

# 3. 反转链接图生成 backlinks
# （在上一步的基础上，反转 A→B 为 B←A）
```

### index.md 格式

```markdown
---
last_rebuilt: <YYYY-MM-DD HH:mm>
total_pages: <N>
total_logs: <N>
total_links: <N>
---

# 📚 Wiki Index

## Pages

- [[auth-system]] — 认证系统设计 [decision, auth, security] — 2026-04-06
- [[jwt-issues]] — JWT 常见问题 [pitfall, auth, jwt] — 2026-04-06
- [[session-management]] — 会话管理策略 [pattern, auth, session] — 2026-04-07

## Logs

- [[log:2026-04-06_user-auth]] — 用户认证任务日志 [log, auth]

## Link Graph

```text
auth-system → jwt-issues, session-management
jwt-issues → auth-system, token-refresh
session-management → auth-system, redis-config
```

## Backlinks

```text
auth-system ← jwt-issues, session-management
jwt-issues ← auth-system
token-refresh ← jwt-issues
redis-config ← session-management
session-management ← auth-system
```
```

---

## 供其他 Skill 调用的接口约定

其他 skill（wiki-planner、wiki-execute、wiki-replan）可通过以下方式与 Wiki 交互：

### 检索知识（读取）

```bash
# 按关键词搜索相关页面
grep -rl "<keyword>" wiki/pages/ 2>/dev/null

# 按经验类型筛选（type 字段）
grep -rl "^type: pitfall" wiki/pages/ 2>/dev/null    # 踩坑记录
grep -rl "^type: solution" wiki/pages/ 2>/dev/null   # 解决方案
grep -rl "^type: pattern" wiki/pages/ 2>/dev/null    # 可复用模式
grep -rl "^type: decision" wiki/pages/ 2>/dev/null   # 架构决策

# 按技术栈筛选
grep -rl "tech_stack:.*<tech>" wiki/pages/ 2>/dev/null

# 按标签筛选
grep -rl "tags:.*<tag>" wiki/pages/ 2>/dev/null

# 读取特定页面
cat wiki/pages/<page-name>.md

# 读取反向链接
grep "<page-name> ←" wiki/index.md
```

**消费方按 type 读取的规范**：

| 消费方 | 优先读取的 type | 目标 Section |
|--------|----------------|-------------|
| `wiki-planner` | `pitfall` | `## 规避方法` |
| `wiki-planner` | `decision` | `## 最终决策` + `## 理由` |
| `wiki-execute` | `pitfall` | `## 规避方法` |
| `wiki-execute` | `solution` | `## 方案` + `## 局限性` |
| `wiki-replan`  | `solution` | `## 方案` + `## 验证结果` |
| `wiki-replan`  | `pattern`  | `## 实现方式` |

### 写入知识（写入后必须更新索引）

**写入触发规则**（由 wiki-execute / wiki-replan 执行时判断）：

| 场景 | 写入 type | 初始 confidence | origin |
|------|-----------|----------------|--------|
| 步骤失败 | `pitfall` | `low` | `wiki-execute` |
| 找到有效修复 | `solution` | `medium` | `wiki-execute` |
| 发现可复用做法 | `pattern` | `medium` | `wiki-execute` |
| 做了技术选型 | `decision` | `high` | `wiki-execute` |
| 重规划触发 | `pitfall` | `low` | `wiki-replan` |
| 用户手动创建 | 任意 | 任意 | `manual` |
| 常规步骤完成 | 不写 Page，仅追加 Log | — | — |

1. 创建/更新 `wiki/pages/<name>.md`（含完整 frontmatter + 对应 type 的 Section）
2. 创建/追加 `wiki/logs/<date>_<task>.md`
3. 重建 `wiki/index.md`

### Confidence 级别

| 级别 | 含义 | 典型场景 |
|------|------|----------|
| `high` | 经过多次验证的成熟知识 | 多次成功使用的方案、已确认的架构决策 |
| `medium` | 单次验证或推断所得 | 一次成功的解决方案、合理的模式推断 |
| `low` | 未验证假设或失败经验 | 失败记录、待验证想法、重规划中的新方案 |

---

## 边界情况处理

### Wiki 目录不存在

```
如果 wiki/ 目录不存在：
  → 询问用户是否初始化
  → 如果是 → mkdir -p wiki/pages wiki/logs && 创建空 index.md
  → 如果否 → 结束
```

### 页面名冲突

```
如果 wiki/pages/<name>.md 已存在：
  → 操作为 create 时 → 提示用户选择：覆盖 / 追加 / 改名
  → 操作为 update 时 → 正常更新
```

### 索引损坏或过期

```
如果 index.md 内容与实际页面不一致：
  → 完全重建索引（全量扫描 wiki/pages/ 和 wiki/logs/）
```

### 孤立页面

```
如果某个页面没有任何入链和出链：
  → 在索引中标记为 "🏝️ 孤立页面"
  → 提示用户考虑添加关联
```

---

## 注意事项

- Wiki 数据存储在项目根目录的 `wiki/` 下，建议加入 `.gitignore`（或根据团队决定是否提交）
- 索引重建是幂等操作，可以随时安全执行
- 页面文件名必须是 kebab-case，不包含空格和特殊字符
- `[[链接]]` 中的名称必须与文件名完全一致（大小写敏感）
- 本 skill 是 wiki-planner、wiki-execute、wiki-replan 的底层依赖
