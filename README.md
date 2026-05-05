# Wiki Skill 体系

> 基于 Markdown + 反向链接的本地知识驱动开发工具链。  
> 将项目执行过程中的经验（踩坑、方案、模式、决策）结构化沉淀，并在规划、执行、重规划阶段自动检索复用。

---

## Skill 一览

| Skill | 定位 | 调用时机 |
|-------|------|---------|
| [`/wiki`](#wiki) | 底层知识库 CRUD | 手动创建/查询/更新/浏览 Wiki 页面 |
| [`/wiki-planner`](#wiki-planner) | 知识驱动规划 | 需求拆解为里程碑路线图（规划前先查 Wiki）|
| [`/wiki-execute`](#wiki-execute) | 知识驱动执行 | 执行路线图步骤（执行前查、执行后写）|
| [`/wiki-replan`](#wiki-replan) | 知识驱动重规划 | 步骤失败后局部重写路线图 |

---

## 协作流程

```
         需求输入
            │
            ▼
     ┌──────────────┐
     │ /wiki-planner │ ← 读取 wiki/pages/* (历史经验)
     │  知识驱动规划  │ → 生成 .claude/roadmaps/<name>.md
     └──────┬───────┘
            │
            ▼
     ┌──────────────┐
     │ /wiki-execute │ ← 读取 wiki/pages/* (知识辅助)
     │  迭代执行      │ → 写入 wiki/pages/* & logs/* (沉淀)
     └──────┬───────┘
            │
        成功? ──是──→ 下一步（循环）
            │
           否
            ▼
     ┌──────────────┐
     │ /wiki-replan  │ ← 读取 wiki/pages/* (已知方案)
     │  智能重规划    │ → 写入 wiki/pages/* (失败经验)
     └──────┬───────┘   → 更新 roadmaps/<name>.md
            │
            ▼
     回到 /wiki-execute
            │
     ┌──────────────┐
     │    /wiki      │ ← 底层 CRUD（被所有上层调用）
     └──────────────┘
```

---

## 目录结构

```
wiki/
  pages/    ← 结构化知识页面（长期沉淀，持续更新）
  logs/     ← 任务执行日志（归档，不再修改）
  index.md  ← 自动维护的索引 + 链接图
```

---

## Page Schema

所有 `wiki/pages/` 下的页面遵循统一 Schema，`type` 字段决定正文结构。

### Frontmatter

```yaml
---
# 必填
title: string
type: pitfall | solution | pattern | decision
created: YYYY-MM-DD HH:mm
updated: YYYY-MM-DD HH:mm
confidence: high | medium | low

# 推荐填
tags: [string]
tech_stack: [string]
source_task: string

# 自动填（由 skill 写入时加）
origin: wiki-execute | wiki-replan | manual
---
```

### 写入触发规则

| 场景 | type | 初始 confidence | 写入方 |
|------|------|----------------|--------|
| 步骤失败 | `pitfall` | `low` | wiki-execute |
| 找到有效修复 | `solution` | `medium` | wiki-execute |
| 发现可复用做法 | `pattern` | `medium` | wiki-execute |
| 做了技术选型 | `decision` | `high` | wiki-execute |
| 重规划触发失败 | `pitfall` | `low` | wiki-replan |
| 重规划找到新方案 | `solution` | `low` | wiki-replan |
| 常规步骤完成 | 不写 Page，仅追加 Log | — | — |

### Confidence 流转

```
low（失败/未验证）→ medium（单次验证）→ high（多次验证成熟）
```

由 `/wiki-execute` 在后续验证成功时自动升级。

---

## 各 Skill 详情

### /wiki

底层知识库，其他三个 skill 的依赖基础。

**操作模式：**

| 命令 | 功能 |
|------|------|
| `wiki create <name>` | 按 type 创建结构化 Page |
| `wiki search <keyword>` | 四层递进检索（标题→标签→全文→链接追踪）|
| `wiki read <name>` | 展示页面内容 + 反向链接 |
| `wiki update <name>` | 修改/追加内容，更新索引 |
| `wiki links <name>` | 查看出链/入链/2度关联 |
| `wiki index` | 展示/重建 `wiki/index.md` |

**检索接口（供其他 skill 调用）：**

```bash
# 按 type 筛选
grep -rl "^type: pitfall" wiki/pages/    # 踩坑记录
grep -rl "^type: solution" wiki/pages/   # 解决方案
grep -rl "^type: pattern" wiki/pages/    # 可复用模式
grep -rl "^type: decision" wiki/pages/   # 架构决策

# 按技术栈筛选
grep -rl "tech_stack:.*<tech>" wiki/pages/
```

---

### /wiki-planner

在 `milestone-plan` 基础上增加 Wiki 知识检索，用历史经验优化规划质量。

**核心差异：**
- 规划前检索 Wiki，按 `type` 定向提取：
  - `pitfall` → `## 规避方法` → 预置步骤风险提示（`⚠️ Wiki提示`）
  - `decision` → `## 最终决策` → 架构参考
  - `solution/pattern` → `## 方案/实现方式` → 步骤优化参考
- 路线图 frontmatter 增加 `wiki_refs` 字段，记录引用的页面

**用法：**
```
/wiki-planner 实现用户认证系统 --name user-auth
/wiki-planner 重构 API 网关 --no-wiki  # 跳过 Wiki 检索
```

---

### /wiki-execute

在 `milestone-execute` 基础上增加 Wiki 知识闭环：执行前查知识，执行后沉淀经验。

**核心差异：**

执行前（Phase 3）：
- 检索当前步骤相关的 Wiki 页面
- 优先读取 `pitfall.规避方法` 和 `solution.方案`
- 展示知识辅助摘要

执行后（Phase 6）：
- 判定是否产生新知识
- 按 type 写入结构化 Page
- 验证已有知识时自动升级 confidence

**用法：**
```
/wiki-execute                        # 执行下一步
/wiki-execute --status               # 只查看进度
/wiki-execute --retry                # 重试上一个失败步骤
/wiki-execute --skip                 # 跳过当前步骤
```

---

### /wiki-replan

当执行步骤失败且重试无效时介入，局部重写路线图，失败经验强制沉淀。

**三级应对策略：**

```
Level 1: wiki-execute --retry（简单重试）
Level 2: wiki-replan（查 Wiki + 局部重写步骤）
Level 3: wiki-replan --scope milestone（扩大范围重规划）
```

**核心差异：**
- 失败根因分类：实现错误 / 方案不可行 / 前序缺陷 / 需求变更 / 外部障碍
- 按 type 检索：`solution` 找直接方案，`pattern` 找替代路线，`pitfall` 找已知陷阱
- 写入两类 Page：`pitfall`（失败经验）+ `solution`（新方案，confidence: low 待验证）

**用法：**
```
/wiki-replan                         # 自动定位失败步骤，智能判断范围
/wiki-replan --scope step            # 只重写当前失败步骤
/wiki-replan --scope milestone       # 重写整个里程碑剩余步骤
/wiki-replan --from S2.3             # 从指定步骤开始重规划
```

---

## 反向链接语法

使用 Obsidian 兼容的 `[[page-name]]` 语法：

```markdown
[[jwt-issues]]                    → wiki/pages/jwt-issues.md
[[log:2026-04-06_user-auth]]      → wiki/logs/2026-04-06_user-auth.md
```

---

*Last updated: 2026-05-05*
