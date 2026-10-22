# 08 - 学习系统

> 让 AI "越用越熟"，同时不让它"越用越错"。

---

## 1. 要解决的问题

每次用户问一个问题，AI 都要重新做一遍：

```
理解问题 → 猜该查哪个资源 → 列资源 → 筛选 → 拿到答案
```

下次问类似的问题，还是从头来。

**学习系统要做的**：把"这个问题 → 这个资源"的映射记下来，下次直接复用。

**它必须避免的**：把自己猜错的映射也记下来，然后一直错下去。

这两个目标是对立的，所以整套设计围绕"隔离"展开。

---

## 2. 三层知识

```
第一层  search_learnings      具体映射
        "上季度销售" → dashboard/abc123

第二层  search_patterns       模板泛化
        "{指标} 的 {时间}" → "销售_{指标}看板"

第三层  learning_playbooks    完整步骤
        query_family → [命令1, 命令2, ...] + 注意事项
```

外加两个辅助表：

```
entity_lookups    实体字典：canonical ↔ value
learn_candidates  隔离区：还没被确认的候选
```

---

## 3. 表结构（实测）

### 3.1 search_learnings

```sql
CREATE TABLE search_learnings (
    id                INTEGER PRIMARY KEY AUTOINCREMENT,
    query_pattern     TEXT NOT NULL,
    query_entities    TEXT,
    venue             TEXT,
    resource_type     TEXT,
    resource_id       TEXT NOT NULL,
    action            TEXT NOT NULL,
    alias_target      TEXT,
    source            TEXT NOT NULL,
    confidence        INTEGER DEFAULT 1,
    created_at        DATETIME DEFAULT CURRENT_TIMESTAMP,
    last_observed_at  DATETIME,
    notes             TEXT
);
```

关键字段：

| 字段 | 作用 |
|---|---|
| `query_pattern` | 归一化后的问题 |
| `query_entities` | 提取出的实体（JSON） |
| `venue` | 场景/入口（用来区分不同用法） |
| `action` | 建议的动作（view / edit / export …） |
| `source` | 这条知识从哪来 |
| `confidence` | 可信度（1 起，验证后升到 2） |

### 3.2 source 的四种取值（实测）

```
taught            人/LLM 显式教的
inferred-followup 从后续追问推断
inferred-reach    从访问路径推断
inferred-pair     从成对出现推断
```

后三种是"推断出来"的，可信度天然低于 `taught`。

### 3.3 search_patterns

```sql
CREATE TABLE search_patterns (
    id                  INTEGER PRIMARY KEY AUTOINCREMENT,
    query_template      TEXT NOT NULL,
    resource_template   TEXT NOT NULL,
    resource_type       TEXT NOT NULL,
    venue               TEXT,
    strategy            TEXT NOT NULL,
    entity_kind         TEXT NOT NULL,
    confidence          INTEGER DEFAULT 2,
    source              TEXT NOT NULL,
    created_at          DATETIME DEFAULT CURRENT_TIMESTAMP,
    last_observed_at    DATETIME,
    example_query       TEXT,
    example_resource    TEXT
);
```

带 `example_query` / `example_resource` 是个好设计——模板太抽象，给一个具体例子，人和 AI 都好理解。

### 3.4 learning_playbooks

```sql
CREATE TABLE learning_playbooks (
    id                INTEGER PRIMARY KEY AUTOINCREMENT,
    query_family      TEXT NOT NULL UNIQUE,
    playbook_json     TEXT,
    notes_text        TEXT,
    source            TEXT NOT NULL DEFAULT 'taught',
    confidence        INTEGER DEFAULT 2,
    created_at        TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    last_observed_at  TIMESTAMP
);
```

`query_family` 唯一——同一类问题只保留一份 playbook。

### 3.5 entity_lookups

```sql
CREATE TABLE entity_lookups (
    kind       TEXT NOT NULL,
    canonical  TEXT NOT NULL,
    value      TEXT NOT NULL,
    source     TEXT NOT NULL DEFAULT 'seeded',
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (kind, canonical, value)
);
```

`source='seeded'` 说明系统带预置词表，不是从零学。

---

## 4. 召回算法（实测规则）

二进制里的原文：

> Returns prior learnings matching the supplied query by **token-set overlap (Jaccard >= 0.6)** plus entity-aware validation. If `found=true` AND `results[0].entity_match == "exact"` AND `confidence >= 2`, the LLM can skip discovery and go straight to live fetch.

翻译成伪代码：

```go
func Recall(query string) Result {
    q := normalize(query)                    // 小写、去标点、分词
    entities := extractEntities(q)           // 用 entity_lookups 匹配

    candidates := store.SearchLearnings(q)   // SQL: LIKE + 前缀匹配
    scored := []Scored{}
    for _, c := range candidates {
        j := jaccard(q.tokens, c.tokens)
        if j < 0.6 { continue }
        em := matchEntities(entities, c.query_entities)
        scored = append(scored, Scored{c, j, em})
    }

    sort(scored, by(confidence, j))

    if len(scored) == 0 {
        recordMiss(query)
        return Result{Found: false, Results: []}
    }

    top := scored[0]
    skipDiscovery := top.EntityMatch == "exact" && top.Confidence >= 2
    return Result{Found: true, Results: scored, CanSkipDiscovery: skipDiscovery}
}
```

对应的 SQL（实测）：

```sql
SELECT ... FROM search_learnings
WHERE query_pattern = ? OR ? LIKE '%' || query_pattern || '%'
ORDER BY confidence DESC, last_observed_at DESC
```

Jaccard 定义：

```
J(A, B) = |A ∩ B| / |A ∪ B|
```

### 4.1 三重条件为什么必要

| 条件 | 防什么 |
|---|---|
| Jaccard ≥ 0.6 | 语义相近（防止"销售额"匹配到"销售人员"） |
| entity_match == exact | 实体精确（防止"上季度"匹配成"本季度"） |
| confidence ≥ 2 | 至少被验证过（防止一次偶然命中被当成规律） |

三条全中才允许"跳过发现"，说明作者对误判非常谨慎。

### 4.2 实体匹配的三种结果

```
exact    ≥ 所有关键实体都精确匹配
partial  部分匹配
none     没有匹配
```

**规范**：

- **必须**把"跳过发现"的权限限制在最严格的条件上。
- **必须**在没命中时记录 miss（用于后续改进）。
- **应该**按 `confidence` 和 `last_observed_at` 排序，新且可信的优先。

---

## 5. 候选生命周期（核心设计）

### 5.1 表结构

```sql
CREATE TABLE learn_candidates (
    id                   INTEGER PRIMARY KEY,
    class                TEXT NOT NULL CHECK(class IN ('flag_alias','playbook_candidate')),
    payload              TEXT NOT NULL,
    derivation_signature TEXT NOT NULL UNIQUE,
    sightings            INTEGER NOT NULL DEFAULT 1,
    status               TEXT NOT NULL DEFAULT 'open'
                             CHECK(status IN ('open','confirmed','rejected','expired')),
    query_family         TEXT,
    command_path         TEXT,
    created_at           TEXT NOT NULL,
    updated_at           TEXT NOT NULL,
    last_seen_at         TEXT NOT NULL
);
```

### 5.2 状态机

```
        自动观察到一次
               │
               ▼
          ┌─────────┐
          │  open   │◄──────────────┐
          └────┬────┘               │ 再次出现
               │                    │ (sightings+1)
      ┌────────┼────────┐           │
      ▼        ▼        ▼           │
 confirm   reject   超时未出现   ┌───┴────┐
      │        │        │        │expired │
      ▼        ▼        └───────►└────────┘
  写入正式表  墓碑
```

关键 SQL（实测）：

```sql
-- 重复观察：sightings 自增，expired 的可以复活
INSERT INTO learn_candidates (...) VALUES (...)
ON CONFLICT(derivation_signature) DO UPDATE SET
    sightings  = sightings + 1,
    status     = CASE WHEN learn_candidates.status = 'expired'
                      THEN 'open' ELSE learn_candidates.status END,
    updated_at = excluded.updated_at,
    last_seen_at = excluded.last_seen_at
WHERE learn_candidates.status != 'rejected';
```

注意最后那个 `WHERE ... != 'rejected'`：**墓碑一旦立了，就不再复活**。

### 5.3 最关键的一句话（实测原文）

> Candidates are quarantined until an explicit `learnings confirm <id>` materializes them;
> they never alter command behavior or recall's verified results on their own.

翻译：**候选在被显式确认之前，不会改变任何行为，也不会污染召回的已验证结果。**

### 5.4 为什么这样设计

| 风险 | 隔离区如何化解 |
|---|---|
| AI 一次误判被当成规律 | 候选不会自动生效 |
| 同一条噪音反复出现 | `derivation_signature` 唯一约束，只加计数 |
| 被否掉的错误反复冒出来 | `rejected` 墓碑永久有效 |
| 环境变化后老候选过期 | `expired` 状态，再次出现可复活 |

---

## 6. 事件流水与效果指标

### 6.1 learn_events

```sql
CREATE TABLE learn_events (
    id                  INTEGER PRIMARY KEY,
    ts                  TEXT NOT NULL,
    event               TEXT NOT NULL CHECK(event IN (
                          'recall_hit','recall_miss','recall_playbook_hit',
                          'teach','teach_playbook','amend','forget',
                          'candidate_confirmed','candidate_rejected')),
    query_family_hash   TEXT,
    matched_row_id      INTEGER,
    entity_match        INTEGER,
    surface             TEXT CHECK(surface IN ('cli','mcp'))
);
```

`surface` 区分 CLI 和 MCP，可以分别看两个入口的效果。

### 6.2 指标（实测输出）

```json
{
  "recall_hit_rate": null,
  "teach_to_reuse": null,
  "playbook_resolution_rate": null,
  "reused_rows": 0,
  "taught_rows": 0,
  "candidates_confirmed": 0,
  "candidates_rejected": 0,
  "events_pruned": 0
}
```

含义：

| 指标 | 怎么算 | 说明什么 |
|---|---|---|
| `recall_hit_rate` | hits / (hits + misses) | 经验库有没有用 |
| `teach_to_reuse` | reused_rows / taught_rows | 教的东西有没有被再次用到 |
| `playbook_resolution_rate` | 按 playbook 成功解决的比例 | 步骤是否可靠 |

**规范**：

- **必须**记录 hit 和 miss，否则算不出命中率。
- **必须**给事件表设保留期（实测有 `DELETE FROM learn_events WHERE ts < ?`）。
- **应该**把指标做成一条命令，而不是让用户自己写 SQL。

### 6.3 召回失败表

```sql
CREATE TABLE learn_recall_misses (
    entity        TEXT PRIMARY KEY,
    miss_count    INTEGER NOT NULL DEFAULT 1,
    first_seen_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    last_seen_at  DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

用途：找出"老是召回不到的实体"，作为下一步教什么的依据。

**注意**：实测这个表在建表语句里存在，但实际库里**没有**——说明它是按需创建的。

---

## 7. 冷启动问题

新装用户：

```
search_learnings 空
patterns 空
playbooks 空
→ recall 全部返回 found:false
```

实测的解决方案（`learn_protocol` 原文）：

> Empty-store short-circuit: if the store has no learnings, playbooks, or candidates yet,
> skip recall for the rest of the session.

**规范**：

- **必须**对空库短路，不要每次都白跑一次召回。
- **应该**带预置实体词表（`source='seeded'`），至少让实体识别从第一天就有效。
- **可以**提供初始 playbook 模板。

---

## 8. 反模式

| 反模式 | 后果 | 正确做法 |
|---|---|---|
| 自动学到的直接生效 | 越用越错 | 候选隔离 + 显式确认 |
| 只记录成功，不记录失败 | 算不出命中率，无法判断有没有用 | hit 和 miss 都记 |
| 用完全相同的字符串匹配 | 换个说法就失效 | 归一化 + Jaccard + 实体校验 |
| 置信度只有一个值 | 无法区分"教过的"和"猜的" | 分级：1 推断 / 2 验证 |
| 拒绝后直接删除记录 | 同样的错误反复出现 | 墓碑机制 |
| 无限增长的事件表 | 数据库膨胀 | TTL 清理 |
| 把用户输入拼进 shell | 命令注入 | 用参数传递或文件 |

最后一条尤其重要，实测原文专门警告：

> never interpolate user-controlled text into a shell command line, quoted string, or heredoc:
> an apostrophe breaks quotes, and a body line equal to the delimiter breaks a heredoc.

---

## 9. 最小实现骨架

```go
// 召回
func Recall(q string) (Result, error) {
    tokens := tokenize(normalize(q))
    ents := lookupEntities(q)

    if store.IsEmpty() {
        return Result{Found: false, Skipped: true}, nil   // 空库短路
    }

    cands, err := store.SearchLearnings(q)
    if err != nil { return Result{}, err }

    best := pickBest(tokens, ents, cands)   // Jaccard >= 0.6 + 实体校验

    if best == nil {
        store.RecordMiss(q)
        store.AppendEvent("recall_miss", q, "", 0)
        return Result{Found: false, Results: []Result{}}, nil
    }

    store.AppendEvent("recall_hit", q, best.ID, best.Confidence)
    return Result{
        Found:            true,
        Results:          cands,
        CanSkipDiscovery: best.EntityMatch == "exact" && best.Confidence >= 2,
    }, nil
}

// 自动观察 → 候选
func Observe(class, payload, signature, family string) error {
    return store.UpsertCandidate(class, payload, signature, family)
    // 内部：ON CONFLICT(signature) DO UPDATE sightings = sightings + 1
    //       WHERE status != 'rejected'
}

// 人工确认
func Confirm(id int64) error {
    c, err := store.GetCandidate(id)
    if err != nil { return err }

    switch c.Class {
    case "flag_alias":
        err = store.InsertLearning(c.Payload)
    case "playbook_candidate":
        err = store.InsertPlaybook(c.Payload)
    }
    if err != nil { return err }

    store.SetCandidateStatus(id, "confirmed")
    store.AppendEvent("candidate_confirmed", "", id, 0)
    return nil
}
```

---

## 检查清单

- [ ] 自动学到的知识是否需要人工确认才生效？
- [ ] 是否有唯一签名防止重复候选？
- [ ] 拒绝后是否留墓碑，防止反复冒出来？
- [ ] 召回是否同时用"相似度 + 实体精确度 + 置信度"？
- [ ] hit 和 miss 是否都记录？
- [ ] 事件表是否有保留期？
- [ ] 空库时是否短路？
- [ ] 是否有预置实体词表？
- [ ] 是否提供"效果统计"命令？
- [ ] 用户输入是否绝对不会拼进 shell？
