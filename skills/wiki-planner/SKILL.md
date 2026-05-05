---
name: wiki-planner
description: "Wiki 增强的智能规划器。当用户提到'wiki规划'、'wiki plan'、'智能拆解'、'wiki-planner'、'知识驱动规划'、'wiki拆解'、'带知识的规划'等关键词，或明确要求在检索已有知识的基础上进行需求拆解和路线图生成时使用此技能。与 milestone-plan 的区别：本技能会先检索 Wiki 已有知识来优化规划质量。"
argument-hint: <需求描述> [--name 项目代号] [--no-wiki 跳过Wiki检索]
---

# Wiki 增强的智能规划器

在 `milestone-plan` 的基础上增加 **Wiki 知识检索**能力：规划前先查阅 Wiki 中的已有经验、踩坑记录、最佳实践，用已有知识优化任务拆解质量。

用户输入：$ARGUMENTS

---

## 核心原则

1. **知识复用优先** — 规划前必须检索 Wiki，避免重复踩坑
2. **经验驱动拆解** — 如果 Wiki 中有相关经验，据此调整步骤粒度和顺序
3. **关联透明** — 路线图中标注引用了哪些 Wiki 页面，方便执行时追溯
4. **规划即知识** — 规划过程本身的决策逻辑也值得沉淀

---

## 执行步骤

### Phase 1：理解需求

与 `milestone-plan` 的 Phase 1 完全一致：

1. 解析 `$ARGUMENTS` 中的需求描述
2. 提取核心功能、约束条件、隐含需求
3. 分析项目结构获取技术上下文

```bash
ls -la
cat package.json 2>/dev/null | head -30
find . -maxdepth 2 -type d -not -path '*/node_modules/*' -not -path '*/.git/*' | head -30
```

提取或生成项目代号（kebab-case）。

### Phase 2：Wiki 知识检索

**这是与 milestone-plan 的核心区别。**

如果 `$ARGUMENTS` 包含 `--no-wiki`，跳过此阶段。

#### 2.1 检查 Wiki 是否存在

```bash
if [ -d "wiki/pages" ] && [ "$(ls wiki/pages/*.md 2>/dev/null | wc -l)" -gt 0 ]; then
  echo "📚 Wiki 可用，共 $(ls wiki/pages/*.md | wc -l) 个知识页面"
else
  echo "📭 Wiki 为空或不存在，跳过知识检索"
fi
```

#### 2.2 多层检索

从需求描述中提取关键词，按四层策略检索：

**Layer 1 — 标题匹配**：

```bash
# 从需求中提取关键词，匹配 Wiki 页面名
for keyword in <extracted_keywords>; do
  ls wiki/pages/ 2>/dev/null | grep -i "$keyword"
done
```

**Layer 2 — 标签匹配**：

```bash
# 在 frontmatter 的 tags 中搜索
for keyword in <extracted_keywords>; do
  grep -rl "tags:.*$keyword" wiki/pages/ 2>/dev/null
done
```

**Layer 3 — 全文搜索**：

```bash
# 搜索页面正文中的关键词
for keyword in <extracted_keywords>; do
  grep -rl "$keyword" wiki/pages/ 2>/dev/null
done
```

**Layer 4 — 链接追踪**：

```bash
# 从已匹配的页面出发，通过 index.md 追踪关联页面
for page in <matched_pages>; do
  grep "$page" wiki/index.md 2>/dev/null
done
```

#### 2.3 读取相关页面

读取 Layer 1-4 命中的页面内容（去重后），按 `type` 字段定向提取：

- `type: pitfall` → 读取 `## 规避方法`，作为步骤风险提示
- `type: decision` → 读取 `## 最终决策` + `## 理由`，作为架构参考
- `type: solution` → 读取 `## 方案` + `## 局限性`，作为实现参考
- `type: pattern` → 读取 `## 适用场景` + `## 实现方式`，评估是否可直接复用
- 所有类型：优先关注 `confidence: high` 的成熟经验

#### 2.4 知识摘要

将检索到的相关知识整理为结构化摘要：

```
┌─────────────────────────────────────────┐
│  📚 Wiki 知识检索结果                     │
├─────────────────────────────────────────┤
│                                         │
│  命中 <N> 个相关页面:                     │
│                                         │
│  ⚠️  [[jwt-issues]]                     │
│     type: pitfall | confidence: high    │
│     规避方法: 注意并发刷新竞态条件        │
│                                         │
│  🏛️  [[auth-system]]                    │
│     type: decision | confidence: medium │
│     最终决策: 使用中间件架构处理认证      │
│                                         │
│  🔧  [[token-refresh-pattern]]          │
│     type: pattern | confidence: medium  │
│     适用场景: 需要无感知刷新的 SPA 项目   │
│                                         │
│  📄 [[log:2026-04-01_api-gateway]]      │
│     → 上次 JWT 实现遇到 CORS 问题        │
│                                         │
│  💡 这些知识将用于优化规划                │
└─────────────────────────────────────────┘
```

### Phase 3：知识增强的拆解

在 `milestone-plan` 的 Phase 2 拆解逻辑基础上，增加以下知识增强：

#### 3.1 风险预判

根据 Wiki 中 `type: pitfall` 的页面（读取 `## 规避方法`），在路线图中预置风险提示：

```markdown
- [ ] **S2.3**: 实现 JWT Token 刷新
  - 验收: Token 刷新测试通过
  - 产出: src/auth/refresh.ts
  - ⚠️ Wiki提示: [[jwt-issues]] 记录了 Token rotation 的坑，注意并发刷新场景
```

#### 3.2 步骤优化

根据 Wiki 中的成功经验，优化步骤顺序或合并/拆分步骤：

- 如果 Wiki 记录了某个方案比较复杂 → 拆分为更细粒度的步骤
- 如果 Wiki 记录了某个快速方案 → 合并简单步骤
- 如果 Wiki 记录了依赖顺序的坑 → 调整里程碑顺序

#### 3.3 引用标注

在路线图中明确标注引用了哪些 Wiki 知识：

```markdown
## 知识引用

本路线图参考了以下 Wiki 知识：

- [[jwt-issues]] — Token 过期和刷新策略
- [[auth-system]] — 中间件架构参考
- [[log:2026-04-01_api-gateway]] — 上次 JWT 实现经验
```

### Phase 4：生成路线图文档

确保路线图目录存在：

```bash
mkdir -p .claude/roadmaps
```

文档格式与 `milestone-plan` 的 Phase 3 完全一致（YAML frontmatter + 里程碑总览 + 步骤清单），但增加以下内容：

1. **frontmatter 增加字段**：

```yaml
---
project: <项目代号>
created: <YYYY-MM-DD HH:mm>
last_updated: <YYYY-MM-DD HH:mm>
status: active
current_milestone: 1
total_milestones: <N>
completed_milestones: 0
wiki_refs: [jwt-issues, auth-system]  # 新增：引用的 Wiki 页面
---
```

2. **步骤中的 Wiki 提示**（见 3.1）

3. **文档末尾的知识引用章节**（见 3.3）

4. **变更日志首条**：

```markdown
| <创建时间> | 创建 | 初始路线图生成（参考 Wiki: <N> 个页面） |
```

### Phase 5：输出摘要报告

```
╔══════════════════════════════════════════════════╗
║           📋 Wiki 增强规划完成                     ║
╠══════════════════════════════════════════════════╣
║                                                  ║
║  项目: <项目代号>                                  ║
║  文档: .claude/roadmaps/<项目代号>.md              ║
║                                                  ║
║  里程碑: <N> 个                                   ║
║  总步骤: <M> 个                                   ║
║  Wiki 引用: <K> 个知识页面                         ║
║  风险预警: <J> 个（来自 Wiki 踩坑记录）             ║
║                                                  ║
║  M1: <名称>  (<N>步)                              ║
║  M2: <名称>  (<N>步) ⚠️ 含Wiki风险提示            ║
║  ...                                             ║
║                                                  ║
║  💡 下一步: /wiki-execute 开始实施                  ║
║     重规划: /wiki-replan（如需调整）                ║
╚══════════════════════════════════════════════════╝
```

---

## 质量检查清单

继承 `milestone-plan` 的全部质量检查，额外增加：

- [ ] 如果 Wiki 中有相关知识，路线图中是否已引用
- [ ] Wiki 中记录的已知风险，是否已在对应步骤标注 ⚠️
- [ ] frontmatter 中 `wiki_refs` 列表是否完整
- [ ] 知识引用章节是否列出了所有参考的 Wiki 页面

## 特殊处理

- 如果 Wiki 为空 → 行为退化为普通的 `milestone-plan`，并提示用户开始积累知识
- 如果需求与 Wiki 中的知识完全无关 → 明确告知"未找到相关知识"，正常规划
- 如果 Wiki 中有**矛盾信息**（不同页面给出不同建议）→ 在规划中列出矛盾点，让用户决策
- 如果 `.claude/roadmaps/` 下已存在同名文件 → 提示用户是覆盖还是另存为新版本

---

## 与其他 Skill 的协作

```
/wiki-planner (本技能)
    │
    ├── 读取 → wiki/pages/* (通过 /wiki 接口约定)
    ├── 写入 → .claude/roadmaps/<name>.md
    │
    ├── 后续 → /wiki-execute 执行路线图
    ├── 调整 → /wiki-replan 局部重规划
    └── 底层 → /wiki 知识CRUD
```
