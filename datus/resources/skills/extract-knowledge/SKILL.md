---
name: extract-knowledge
description: 从「问题 + gold_sql」配对中提取业务知识：迭代驱动 gen_sql subagent 但永不暴露 gold 答案，最终把差距提炼成 ./knowledge/*.md
tags:
  - knowledge
  - sql
  - gold-sql
  - iteration
version: "1.0.0"
user_invocable: true
disable_model_invocation: false
---

# 从 Gold SQL 配对中提取业务知识

你会收到一对或多对 `(question, gold_sql)`。对每一对你必须做以下事情：

1. 驱动 `gen_sql` subagent 写出一段 SQL，让它的结果集与 `gold_sql` **完全一致**，**全程不向 subagent 展示 gold SQL 或 gold 结果的任何数值**。
2. 当 subagent 的 SQL 最终匹配时，把它和 `gold_sql` 做 diff，找出 *业务知识差距* —— subagent 不知道的那些规则、join、过滤、粒度或业务定义。
3. 把该知识以结构化 markdown 条目写入 `./knowledge/<domain-slug>.md`，并更新 `./AGENTS.md` 中的 `## Knowledge` 索引。

目标是 **知识提炼**，不是 SQL 生成。匹配的 SQL 只是手段；差距分析才是交付物。

## 关键约束

- **绝不向 subagent 暴露 `gold_sql`**。完整 SQL 不行、片段不行、结果具体数值不行、列级数据也不行。subagent 必须保持"盲态"，它的错误才能揭示出缺失了什么知识。
- **绝不对 subagent 说"你的结果错了，因为 ……"并给出 gold 的具体数值**。只能描述 *症状* 和 *定性方向*（比如"行数偏多"、"应该排除已取消的订单"、"指标口径是按客户而不是按订单"）。
- 跨重试时必须通过 `task` 工具复用 subagent 的 `session_id` —— 这是 subagent 记住先前尝试的唯一方式。

## 输入

接受以下形态（从用户消息或上下文中解析）：

- 单对：一段自然语言问题 + 一段 SQL 代码块。
- 多对：列表、CSV 文件路径，或 YAML/JSON；每对独立处理。

若输入有歧义（找不到清晰的问题，或找不到 SQL 代码块），调用 `ask_user` 澄清。

## 工作流（每对一次）

### 第 1 步 — 校验 gold_sql

用 `read_query(sql=<gold_sql>)` 执行 `gold_sql`。
- 若报错：把错误告知用户并 **跳过此对**。不要为损坏的 gold 编造问题。
- 否则：缓存结果（行数、列名、小段预览）。把它作为后续比对的事实基准。

### 第 2 步 — 首轮调用 subagent

调用：

```
task(
  type="gen_sql",
  prompt=<question>,         # 只传自然语言问题
  description="extract-knowledge: initial attempt for <短主题>"
)
```

返回的 envelope 含 `result.sql`（长 SQL 时是 `result.sql_file_path`）、`result.response` 和 `result.session_id`。**保存 `session_id`** —— 后续每次重试都必须复用它。

### 第 3 步 — 执行并比对

用 `read_query` 执行 subagent 产出的 SQL。与缓存的 gold 结果做比对：

- 行数是否匹配？
- 列名 / 列数是否匹配（语义级别 —— 别名可不同）？
- 当双方按同一 key 排序后，样本行是否匹配？
- 想要精确判定时，用 `read_query` 跑差集探针，比如：
  - `SELECT COUNT(*) FROM (<gold_sql> EXCEPT <subagent_sql>) t`（以及反向），或
  - 在关键列上做聚合一致性校验（`SUM`、`COUNT(DISTINCT …)`）。

判定结论：**匹配** 或 **不匹配**。

### 第 4 步 — 不匹配：诊断并复用 session 重试

定性分析差异。常见差距分类：

| 症状 | 可能缺失的知识 |
|------|----------------|
| 行数偏多 | 缺过滤、缺 join、粒度错 |
| 行数偏少 | 多余过滤、join 条件过严 |
| 聚合值偏差 | 度量列错、缺去重、单位错 |
| 多列 / 少列 | 输出形状不符、投影规则缺失 |
| 时间分组不同 | 粒度 / 业务日历 / 财年规则 |
| 出现不该有的 NULL | join 类型错（LEFT vs INNER）、缺 COALESCE 规则 |

设计提示词时，**只描述症状和方向**，绝不带 gold 数值。示例：

- ✅ "你的结果行数偏多。考虑数据源表是否要按 `order_id` 去重，或者是否要排除已取消订单。"
- ✅ "月度合计看起来不对。重新审视哪一列日期定义的'订单月份' —— 是订单创建日还是付款确认日？"
- ❌ "1 月的正确数字是 4,321，你返回了 5,678。"（泄露 gold 数值）
- ❌ "把你的 `INNER JOIN customers` 改成 `LEFT JOIN customers`。"（直接泄露答案）

然后调用：

```
task(
  type="gen_sql",
  session_id=<已保存的 session_id>,    # 必须是上一轮的同一个 id
  prompt=<仅含提示>,
  description="extract-knowledge: refine #<n>"
)
```

循环 第 3 步 → 第 4 步，**最多 5 轮（含首轮）**。5 轮后仍不匹配，停止并记录 *失败说明*（见第 7 步）。

### 知识文件组织模型（第 5 步之前必读）

每条要持久化的条目都活在严格的三层结构里：

```
业务域 (business domain)   →   主题 (topic, 可嵌套)   →   知识条目 (knowledge entry)
   (单个 .md 文件)              (## / ### / ####)         (主题下的 ### <规则>)
```

- **业务域 = 一个文件。** 业务域是规则会自然一起演进的一片大业务范围（例如：`revenue-recognition`、`user-segmentation`、`game-mode-analytics`、`inventory-snapshot`）。文件路径为 `./knowledge/<domain-slug>.md`。业务域是 **最大** 的组织单位 —— 宁可少而宽，不要多而窄。
- **主题 = 文件内的标题层级。** 主题是业务域里的子领域。主题 *可以* 嵌套（`## 群体定义` → `### 回流用户群体`）。最多用 2–3 层 —— 嵌得更深基本意味着业务域切错了。
- **知识条目 = 主题下的一个 `### <一句话规则>` 块**，沿用第 6 步的条目模板。规则本身就是标题；不要写 `### 规则 1` 或 `### 定义` 这类通用标题。

一个健康的文件长这样：

```markdown
# Game Mode User Segmentation

> **Domain:** 如何把游戏玩家分到群体（新增 / 回流 / 留存 / 平台外）以服务玩法上线分析。

## cbitmap-based cohort typing

### cbitmap 前 3 位代表玩法上线后的 3 天 ……

### 回流用户分层来自 substr(cbitmap, 4) 中首个 1 的位置 ……

## Account-system mapping

### 平台 suserid 与游戏 vplayerid 是两套账号体系，禁止直接 join ……
```

**强偏好：先复用，后新建。** 在持久化任何东西之前，先列 `./knowledge/` 并阅读每个文件的 Domain 引言段。优先在已有业务域里加新主题 / 新条目，而不是新开文件。只有当新知识真正不归属于任何已有业务域时才新建文件 —— 即便如此，也要先用 `ask_user` 与用户确认（见 6.1）。

### 第 5 步 — 匹配：提炼并归类知识

一旦结果匹配，把 subagent 的最终 SQL 与 `gold_sql` 做 diff（再与 subagent *首轮* 的尝试做 diff —— 中间过程本身就是信息）。对每个差距，产出一份草稿条目：

- `rule` —— **一句话规则陈述**（将成为 H3 标题）。
- `when_it_applies` —— 适用于哪类问题 / 哪些表 / 什么范围。
- `why_it_matters` —— 一两句话说明不遵守会出什么问题。
- `example` —— 1–2 段简短 SQL 片段（用代码围栏）。
- `derived_from` —— 触发本条知识的原始问题（用于溯源）。

然后把每份草稿 **归类** 到 业务域 → 主题 层次：

1. **选业务域。** 读 `./knowledge/`，扫每个文件的 Domain 引言段。把草稿对应到下面之一：
   - 已有业务域已覆盖此领域 → 复用；
   - 已有业务域范围 *接近但不完全契合* → 仍然优先复用；用一次小 `edit_file` 扩展它的引言段；
   - 真的是新领域 → 标记为 `new_domain` 并提议一个 slug；新建延迟到 6.1（会触发 `ask_user`）。
2. **选主题路径。** 打开选中的文件，扫它的 `##` / `###` 主题标题，决定：
   - 已有主题 → 原样复用标题；
   - 已有父主题下新建同级子主题 → 记录父标题；
   - 全新顶层主题 → 记 `parent = None`。
3. **重复 / 冲突检测延到 6.2** —— 在选中主题内的写入时刻做。

### 第 6 步 — 持久化知识（带去重 / 冲突门禁）

对每个已归类的条目，**按顺序** 走以下门禁。不可跳步。

#### 6.1 解析目标业务域文件

- 若 `./knowledge/` 尚不存在，直接 `write_file` 创建第一个文件（空目录情形无需 `ask_user`）。
- 若条目被标记为 `new_domain`，在创建新文件 **之前** 必须 `ask_user`。用具体的备选项措辞，例如：
  > "关于 *<领域>* 的新知识不归属任何已有业务域（`game-mode-analytics`、`revenue-recognition`、……）。请选择：
  > 1. **复用 `game-mode-analytics`** —— 同时扩展它的引言段。
  > 2. **新建业务域 `<proposed-slug>`** —— 边界更清晰。
  > 3. **其它** —— 输入自定义 slug。"
- 当存在合理的复用路径时，绝不静默新建文件。

#### 6.2 在该文件内检测重复与冲突

`read_file` 目标文件。沿标题路径找到你在第 5 步选定的 **同一主题路径**。在该主题下，按 *语义* （非字符串）逐条对比草稿规则与已有的 `### <规则>` 条目：

| 结果类型 | 定义 | 处理动作 |
|---------|------|----------|
| **重复 (Duplicate)** | 规则相同、范围相同、方向相同。 | **静默跳过**（幂等）。在最终报告中记录此条。 |
| **细化 (Refinement)** | 已有规则是新规则的严格子集（精度更低、缺一个条件、范围更窄）。 | **`ask_user`**：替换 / 合并 / 都保留并加 `**Scope:**` 备注。默认建议 = 合并。 |
| **冲突 (Conflict)** | 范围相同但方向相反 / 互相矛盾（例如已有规则说"用 `INNER JOIN`"，新规则说"用 `LEFT JOIN`"）。 | **强制 `ask_user`**。把两条规则并排展示，附各自的 `derived_from`。选项：保留旧的 / 用新的替换 / 两者保留并加明确条件门控（你必须说清条件是什么）。**绝不静默解决冲突。** |
| **互补 (Complementary)** | 同一主题、不同侧面（例如一条覆盖行级分组，另一条覆盖空值处理）。 | 在同主题下 **追加** 一条同级 `### <规则>`。 |
| **新主题 / 新业务域** | 没有相关的已有主题。 | 按 6.3 新建标题。 |

#### 6.3 写入或编辑文件

用 `edit_file` 保持最小 diff。标题约定：

- 新业务域 → `write_file` 写入 Domain 引言段，紧跟首个主题标题。
- 新顶层主题 → 插入一个 `## <主题标题>` 块。
- 新子主题 → 在其父主题下插入 `### <子主题标题>`（仅在必要时才用 `####`）。
- 新规则 → `### <一句话规则>` 后接下方的条目模板。

**新业务域文件模板（创建新文件时使用）：**

```markdown
# <业务域标题>

> **Domain:** <一句话范围说明 —— 本文件覆盖什么、不覆盖什么>。
> 由 `/extract-knowledge` 维护；下方每条规则都是 SQL agent 正确回答该业务域问题所需的知识。

## <首个主题标题>

### <一句话规则>

**When it applies:** <适用场景>

**Why it matters:** <一两句话说明不遵守会出什么问题>

**Example:**

\`\`\`sql
-- 最小可示意规则的代码片段
<sql>
\`\`\`

**Derived from:** "<原始问题>"
```

**条目专用模板（向已有主题追加规则时使用）：**

```markdown
### <一句话规则>

**When it applies:** <适用场景>

**Why it matters:** <一两句话说明不遵守会出什么问题>

**Example:**

\`\`\`sql
<sql>
\`\`\`

**Derived from:** "<原始问题>"
```

**不要** 在条目之间插入 `---` 分隔线 —— markdown 标题本身就构成分隔，`---` 会让 `edit_file` 的插入位置变得歧义。

### 第 7 步 — 更新 AGENTS.md 索引

维护 `./AGENTS.md` 中的 `## Knowledge` 章节：

- 若 `./AGENTS.md` 不存在：提示用户先跑 `/init`。不要在此 skill 里从零创建 AGENTS.md。
- 若该章节缺失：把它插入到 `## Artifacts` 之后（若 Artifacts 也没有，就插到文件末尾）。
- 一行对应一个 **业务域文件**（不是每个主题、不是每条规则）：`- [<业务域标题>](knowledge/<domain-slug>.md) — <来自文件 Domain 引言段的一句话范围说明>`
- 重扫 `./knowledge/`，按业务域标题字母序重写整段，保证每次运行后索引一致。

若某对在 5 轮后仍未匹配，在同一个 `## Knowledge` 块下的 `### Open Gaps` 子节里加一行：`- <问题> — <简短原因>`，然后继续处理下一对。

## 最终输出

返回一个 JSON envelope，汇总变更：

```json
{
  "processed": <int>,
  "matched": <int>,
  "domain_files": ["knowledge/<domain-slug>.md", "..."],
  "topics_touched": ["<domain-slug>:<主题标题>", "..."],
  "duplicates_skipped": <int>,
  "conflicts_resolved": <int>,
  "open_gaps": ["<问题>", "..."],
  "output": "<一段人类可读的总结>"
}
```

## 你会用到的工具

- `read_query(sql=...)` —— 执行 gold 与 subagent 的 SQL、跑差集探针。
- `task(type="gen_sql", prompt=..., session_id=...)` —— 委派 SQL 生成，跨重试复用 session。
- `read_file`、`write_file`、`edit_file` —— 管理 `./knowledge/*.md` 与 `./AGENTS.md`。变更目标业务域文件前 **总是** 先 `read_file`（6.2 依赖这点）。
- `ask_user` —— 以下情况必须用：输入解析有歧义、新业务域确认（6.1）、规则细化决策（6.2）、**所有规则冲突**（6.2）。措辞要用具体编号选项。

## 禁止事项

- 在确认 `gold_sql` 能跑通之前，不要调用 `task(type="gen_sql", ...)`。
- 不要为重试新开 `gen_sql` session —— 必须传上一轮的 `session_id`。
- 不要替用户的问题自己写 SQL —— 那是 subagent 的活。你的角色是编排者与知识策展人。
- 不要把 gold SQL 写进 `./knowledge/*.md`。知识是从差距中提炼出的 *规则*，不是答案本身。
- 当已有业务域可能覆盖时，不要新建文件。复用 + 略微扩展引言段几乎总是对的。
- 不要静默解决规则冲突。冲突 **必须** 走 `ask_user` —— 由用户决定替换、共存还是加范围门控。
- 不要在条目之间插入 `---` 分隔线。
- 不要给规则起通用名（`### 规则 1`、`### 备注`）。一句话规则本身就是标题。
- 主题标题嵌套不要超过 `####`。再深就是业务域切错了 —— 要么折叠、要么拆文件。
