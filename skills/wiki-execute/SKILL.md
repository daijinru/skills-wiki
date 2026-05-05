---
name: wiki-execute
description: "Wiki 增强的迭代执行器。当用户提到'wiki执行'、'wiki execute'、'wiki-execute'、'智能执行'、'带知识执行'、'知识驱动执行'、'wiki推进'、'wiki下一步'等关键词，或明确要求在 Wiki 知识辅助下执行路线图步骤时使用此技能。与 milestone-execute 的区别：执行前查 Wiki 获取辅助知识，完成后将经验写入 Wiki。"
argument-hint: [路线图文件路径] [--retry] [--skip] [--status]
---

# Wiki 增强的迭代执行器

在 `milestone-execute` 的基础上增加 **Wiki 知识闭环**：执行前检索 Wiki 获取辅助知识，执行后将新经验写入 Wiki，形成"学习 → 执行 → 沉淀"的正向循环。

用户输入：$ARGUMENTS

---

## 核心原则

1. **一次一步** — 每次调用只执行一个步骤，确保粒度可控
2. **知识辅助** — 执行前查阅 Wiki，用已有经验指导当前步骤
3. **经验沉淀** — 每次执行后判断是否有值得记录的新知识
4. **文档同步** — 路线图 + Wiki + 日志三方保持一致

---

## 执行步骤

### Phase 1：加载路线图

与 `milestone-execute` 的 Phase 1 完全一致：

**文件定位优先级**：
1. 如果 `$ARGUMENTS` 指定了文件路径 → 直接使用
2. 如果 `.claude/roadmaps/` 下只有一个 `.md` 文件 → 自动使用
3. 如果有多个文件 → 列出让用户选择

```bash
ls -lt .claude/roadmaps/*.md 2>/dev/null
```

**特殊参数处理**：
- `--status` → 只展示进度状态，不执行（跳到 Phase 7）
- `--retry` → 重试上一个失败的步骤
- `--skip` → 跳过当前步骤

读取目标路线图的完整内容。

### Phase 2：解析当前状态

与 `milestone-execute` 的 Phase 2 完全一致：

1. 解析 YAML frontmatter
2. 定位当前里程碑章节
3. 扫描步骤状态（`[x]`/`[ ]`/`[!]`）
4. 找到本次要执行的步骤

**状态判定逻辑**：

```
如果 status == "completed" → 项目完成，输出报告，结束
如果存在 [!] 步骤 → 提示重试或跳过（或建议 /wiki-replan）
找到第一个 [ ] 步骤 → 本次执行目标
如果当前里程碑全部 [x] → 里程碑完成，推进到下一个
```

### Phase 3：Wiki 知识检索（执行前）

**这是与 milestone-execute 的第一个核心区别。**

在执行当前步骤之前，从 Wiki 中检索相关知识。

#### 3.1 提取检索关键词

从当前步骤的描述、验收标准、所在里程碑名称中提取关键词。

#### 3.2 多层检索

```bash
# 检查 Wiki 是否可用
if [ -d "wiki/pages" ]; then
  # 标题匹配
  ls wiki/pages/ 2>/dev/null | grep -i "<step_keyword>"
  
  # 全文搜索
  grep -rl "<step_keyword>" wiki/pages/ 2>/dev/null
  
  # 检查路线图 frontmatter 中的 wiki_refs（planner 阶段标注的）
  grep "^wiki_refs:" .claude/roadmaps/<current>.md
fi
```

#### 3.3 读取并提取关键信息

对命中的 Wiki 页面：
- 读取与当前步骤相关的章节
- 特别关注：踩坑记录、解决方案、注意事项
- 如果步骤中有 `⚠️ Wiki提示` 标注（wiki-planner 留下的），优先读取对应页面

#### 3.4 知识辅助摘要

```
┌─────────────────────────────────────────┐
│  📚 Wiki 知识辅助                        │
├─────────────────────────────────────────┤
│                                         │
│  当前步骤: S2.3 — 实现 JWT Token 刷新    │
│                                         │
│  📄 [[jwt-issues]] 提示:                │
│    • 注意并发刷新竞态条件                │
│    • 建议使用 Token rotation 策略        │
│                                         │
│  📄 [[auth-system]] 参考:               │
│    • refresh 中间件已有模式可复用         │
│                                         │
│  ⚠️ 已知风险:                           │
│    • 上次实现时 CORS 预检请求导致问题    │
│                                         │
└─────────────────────────────────────────┘
```

### Phase 4：承前回顾（跨里程碑时）

与 `milestone-execute` 的 Phase 3 完全一致：

当进入新里程碑时，回顾上一个里程碑的产出：
1. 阅读上一里程碑的步骤和完成备注
2. 验证产出物是否存在
3. 如果产出物缺失 → 先补齐

### Phase 5：执行当前步骤

与 `milestone-execute` 的 Phase 4 类似，但增加知识辅助：

#### 5.1 读取步骤信息

提取当前步骤的描述、验收标准、产出物。

#### 5.2 知识增强执行

在执行实现时，**主动运用** Phase 3 检索到的 Wiki 知识：

- 如果 Wiki 中有相关方案 → 参考其实现思路
- 如果 Wiki 中有踩坑记录 → 提前规避已知问题
- 如果 Wiki 中有性能建议 → 在实现时考虑

**注意**：Wiki 知识是参考而非教条，当前项目的具体情况可能需要不同方案。

#### 5.3 验收检查

```bash
# 根据步骤验收标准验证
# 例如：test -f X, npm test, curl 等
```

#### 5.4 判定结果

- 验收通过 → `[x]`，写入完成备注
- 验收失败 → `[!]`，写入失败原因
  - **额外**：如果失败，建议用户使用 `/wiki-replan` 进行重规划

### Phase 6：经验沉淀到 Wiki（执行后）

**这是与 milestone-execute 的第二个核心区别。**

每次步骤执行完成后（无论成功或失败），评估是否有值得沉淀的知识。

#### 6.1 沉淀判定

以下情况**应该**写入 Wiki：

| 场景 | type | 初始 confidence | 示例 |
|------|------|----------------|------|
| 步骤失败 | `pitfall` | `low` | 发现某库有未文档化的行为 |
| 找到有效修复 | `solution` | `medium` | 找到了比 Wiki 现有方案更优的实现 |
| 发现可复用做法 | `pattern` | `medium` | 某种初始化顺序在多处有效 |
| 做了技术选型 | `decision` | `high` | 在两个库之间做了选择 |
| 验证已有知识 | 更新已有 Page（confidence → high） | — | Wiki 中的方案在新场景下也有效 |
| 常规步骤完成 | 不写 Page，仅追加 Log | — | 步骤 S2.3 完成，耗时 5 分钟 |

以下情况**不需要**写入 Wiki：

- 完全按预期完成的简单步骤（仅追加 Log）
- 与项目特定配置相关的一次性信息

#### 6.2 写入 Wiki Page

如果判定需要创建/更新 Page：

```bash
mkdir -p wiki/pages
```

**新建 Page**（使用 wiki skill 定义的 Page Schema，根据 `type` 选择对应 Section）：

`type: pitfall`（步骤失败时）：
```markdown
---
title: <知识标题>
type: pitfall
created: <YYYY-MM-DD HH:mm>
updated: <YYYY-MM-DD HH:mm>
tags: [<相关标签>]
tech_stack: [<技术栈>]
source_task: <当前路线图项目代号>
confidence: low
origin: wiki-execute
---

# <知识标题>

## 触发条件
执行 [[<路线图项目代号>]] S{M}.{N} 时：<触发场景>

## 现象
<具体报错或异常行为>

## 根因
<为什么会发生>

## 规避方法
<怎么避开或修复>

## 相关链接
- [[related-page]]
```

`type: solution`（找到有效修复时）：
```markdown
---
title: <知识标题>
type: solution
created: <YYYY-MM-DD HH:mm>
updated: <YYYY-MM-DD HH:mm>
tags: [<相关标签>]
tech_stack: [<技术栈>]
source_task: <当前路线图项目代号>
confidence: medium
origin: wiki-execute
---

# <知识标题>

## 问题描述
<解决的是什么问题>

## 方案
<具体实现步骤或代码>

## 验证结果
<怎么验证它有效>

## 局限性
<在哪些场景下不适用>

## 相关链接
- [[related-page]]
```

`type: pattern`（发现可复用做法时）：
```markdown
---
title: <知识标题>
type: pattern
created: <YYYY-MM-DD HH:mm>
updated: <YYYY-MM-DD HH:mm>
tags: [<相关标签>]
tech_stack: [<技术栈>]
source_task: <当前路线图项目代号>
confidence: medium
origin: wiki-execute
---

# <知识标题>

## 适用场景
<什么情况下用这个模式>

## 实现方式
<核心做法>

## 示例
<具体例子>

## 局限性
<不适用的情况>

## 相关链接
- [[related-page]]
```

`type: decision`（做了技术选型时）：
```markdown
---
title: <知识标题>
type: decision
created: <YYYY-MM-DD HH:mm>
updated: <YYYY-MM-DD HH:mm>
tags: [<相关标签>]
tech_stack: [<技术栈>]
source_task: <当前路线图项目代号>
confidence: high
origin: wiki-execute
---

# <知识标题>

## 背景
<为什么要做这个决策>

## 选项对比
| 选项 | 优点 | 缺点 |
|------|------|------|
| A    |      |      |
| B    |      |      |

## 最终决策
<选了什么>

## 理由
<为什么这样选>

## 相关链接
- [[related-page]]
```

**更新已有 Page**：
- 追加新的章节或更新 confidence 级别
- 更新 `updated` 字段

#### 6.3 写入/追加任务日志

```bash
mkdir -p wiki/logs
```

追加到 `wiki/logs/<date>_<task-name>.md`：

```markdown
| <时间> | S{M}.{N}: <步骤描述> | ✅ 成功 / ❌ 失败 | <备注> |
```

如果日志文件不存在，按 wiki skill 的 Log 模板创建。

#### 6.4 更新 Wiki 索引

如果创建或更新了 Wiki 页面，重建索引：

```bash
# 扫描所有页面的 [[links]]，重建 index.md
# （使用 wiki skill 中描述的索引重建流程）
```

#### 6.5 沉淀报告

如果本次写入了 Wiki：

```
┌─────────────────────────────────────────┐
│  📝 Wiki 经验沉淀                        │
├─────────────────────────────────────────┤
│                                         │
│  新建页面:                               │
│    📄 wiki/pages/cors-preflight.md      │
│       tags: [cors, api, networking]     │
│       confidence: medium                │
│                                         │
│  更新页面:                               │
│    📄 wiki/pages/jwt-issues.md          │
│       → confidence: high (已验证)       │
│                                         │
│  日志追加:                               │
│    📄 wiki/logs/2026-04-07_user-auth.md │
│       → S2.3 完成记录                    │
│                                         │
└─────────────────────────────────────────┘
```

### Phase 7：更新路线图文档

与 `milestone-execute` 的 Phase 5 完全一致：

1. 更新步骤状态（`[ ]` → `[x]` 或 `[!]`）
2. 写入完成备注
3. 更新里程碑总览表
4. 如果里程碑完成 → 写入里程碑回顾
5. 更新 frontmatter（`last_updated` 等）
6. 追加变更日志

**额外**：如果本次写入了 Wiki，在完成备注中标注：

```markdown
- [x] **S2.3**: 实现 JWT Token 刷新
  - 验收: Token 刷新测试通过
  - 产出: src/auth/refresh.ts
  - 完成备注: 实现了 Token rotation 策略，发现 CORS 预检问题并解决。经验已写入 [[cors-preflight]]。 — 2026-04-07 15:30
```

### Phase 8：进度通报

与 `milestone-execute` 的 Phase 6 格式一致，额外增加 Wiki 信息：

**步骤完成时**：

```
╔══════════════════════════════════════════════════╗
║           ✅ 步骤完成                              ║
╠══════════════════════════════════════════════════╣
║                                                  ║
║  完成: S{M}.{N} — <步骤描述>                      ║
║  备注: <简要完成说明>                              ║
║  Wiki: <写入了 N 个页面 / 无新知识>                ║
║                                                  ║
║  当前里程碑: M{X} <名称>                           ║
║  里程碑进度: [████░░░░] {完成}/{总数}              ║
║  总体进度:   [██░░░░░░] {完成}/{总数}              ║
║                                                  ║
║  📎 下一步: S{M}.{N+1} — <下一步描述>             ║
║  💡 继续执行: /wiki-execute                        ║
║  🔧 需要调整: /wiki-replan                        ║
╚══════════════════════════════════════════════════╝
```

**步骤失败时**：

```
╔══════════════════════════════════════════════════╗
║           ❌ 步骤失败                              ║
╠══════════════════════════════════════════════════╣
║                                                  ║
║  失败: S{M}.{N} — <步骤描述>                      ║
║  原因: <失败原因>                                  ║
║  Wiki: 失败经验已写入 [[<page-name>]]              ║
║                                                  ║
║  🔧 可选操作:                                     ║
║  1. /wiki-execute --retry   重试此步骤             ║
║  2. /wiki-execute --skip    跳过此步骤             ║
║  3. /wiki-replan            智能重规划             ║
║  4. 手动修复后再次执行                              ║
╚══════════════════════════════════════════════════╝
```

**里程碑完成时** / **项目完成时**：格式同 `milestone-execute`，额外附上本里程碑/项目期间写入的 Wiki 页面列表。

---

## 边界情况处理

### Wiki 不存在

```
如果 wiki/ 目录不存在：
  → 自动初始化：mkdir -p wiki/pages wiki/logs
  → 创建空 index.md
  → 正常执行（跳过知识检索，仍进行经验沉淀）
```

### 路线图文件不存在

```
如果 .claude/roadmaps/ 为空：
  → 提示用户先执行 /wiki-planner 或 /milestone-plan
```

### Wiki 知识与当前情况矛盾

```
如果 Wiki 建议的方案与当前项目技术栈不兼容：
  → 在执行备注中记录："Wiki [[X]] 的方案不适用于当前场景，原因：..."
  → 完成后将新发现写入 Wiki（更新或新建页面）
```

### 步骤失败且 Wiki 中有相关方案

```
步骤失败时：
  1. 查询 Wiki 是否有相关的解决方案
  2. 如果有 → 在失败报告中附上 Wiki 建议
  3. 如果没有 → 建议用户使用 /wiki-replan
```

---

## 与其他 Skill 的协作

```
/wiki-execute (本技能)
    │
    ├── 读取 → .claude/roadmaps/<name>.md (路线图)
    ├── 读取 → wiki/pages/* (知识检索)
    ├── 写入 → wiki/pages/* (经验沉淀)
    ├── 写入 → wiki/logs/* (任务日志)
    ├── 更新 → wiki/index.md (索引维护)
    ├── 更新 → .claude/roadmaps/<name>.md (步骤状态)
    │
    ├── 上游 → /wiki-planner 生成路线图
    ├── 失败 → /wiki-replan 智能重规划
    └── 底层 → /wiki 知识CRUD
```
