# 05 - 本地存储层（store）

## 1. 为什么需要本地库

三个理由，缺一个就不该做：

1. **离线**：断网时查询类功能还能用。
2. **全文搜索**：服务器接口通常只支持简单关键字，本地可以用 FTS5 做更好的匹配。
3. **分析**：跨资源做 join、聚合，服务器接口做不到。

**反面**：如果只是"缓存几个响应"，用文件缓存就够了，不需要 SQLite。

---

## 2. 实测表结构

### 2.1 资源镜像

```sql
CREATE TABLE resources (
    id            TEXT NOT NULL,
    resource_type TEXT NOT NULL,
    data          JSON NOT NULL,
    synced_at     DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at    DATETIME DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (resource_type, id)
);

CREATE INDEX idx_resources_type            ON resources(resource_type);
CREATE INDEX idx_resources_type_updated    ON resources(resource_type, updated_at);
CREATE INDEX idx_resources_synced          ON resources(synced_at);

CREATE VIRTUAL TABLE resources_fts USING fts5(
    id, resource_type, content,
    tokenize='trigram'
);
```

**关键决策**：

| 决策 | 原因 |
|---|---|
| 主键是 `(resource_type, id)` | 不同资源类型可能有相同 ID |
| 数据存 JSON 原文 | 不用为每个资源建表，扩展零成本 |
| 单独 FTS 表 | 避免把 JSON 全文塞进主索引 |
| `tokenize='trigram'` | 中文/英文混排都能搜，不依赖分词词典 |

### 2.2 同步状态

```sql
CREATE TABLE sync_state (
    resource_type         TEXT PRIMARY KEY,
    last_cursor           TEXT,
    last_synced_at        DATETIME,
    total_count           INTEGER DEFAULT 0,
    last_attempt_complete INTEGER NOT NULL DEFAULT 0
);
```

四个字段各有用途：

- `last_cursor`：增量同步的游标
- `total_count`：进度展示
- `last_attempt_complete`：区分"同步完成"和"同步到一半崩了"

**规范**：

- **必须**记录"这次同步是否完整结束"。没有这个字段，你就不知道本地数据能不能信。
- **必须**让游标可空——首次同步没有游标。

---

## 3. 同步状态机

```
        ┌──────────┐
        │  no data │
        └────┬─────┘
             │ 首次 sync
             ▼
        ┌──────────┐
        │ syncing  │◄──────────┐
        └────┬─────┘           │ 增量 sync
             │                 │
      ┌──────┴──────┐          │
      ▼             ▼          │
 ┌─────────┐   ┌─────────┐     │
 │complete │   │ partial │─────┘
 └────┬────┘   └─────────┘
      │              │
      │              └─► 下次 sync 从 last_cursor 继续
      └─► 可安全读取
```

**规范**：

- **必须**让读取方知道当前是 complete 还是 partial。
- **应该**在 partial 状态下继续提供旧数据，而不是清空。
- **必须**支持 `--max-pages` 防止一次拉爆内存；触顶时明确告知：

```
{"reason":"max_pages_cap_hit","message":"reached --max-pages cap of 50; data may be truncated."}
```

---

## 4. 迁移

实测存在三个版本标记：

```
store_schema_version
user_version        （SQLite 内置）
schema_version      （agent-context 里是 4）
```

迁移的可靠写法：

```go
func migrate(db *sql.DB) error {
    var v int
    db.QueryRow(`PRAGMA user_version`).Scan(&v)

    for v < len(migrations) {
        tx, err := db.Begin()
        if err != nil { return err }

        if err := migrations[v](tx); err != nil {
            tx.Rollback()
            return fmt.Errorf("migration %d: %w", v, err)
        }
        if _, err := tx.Exec(fmt.Sprintf(`PRAGMA user_version=%d`, v+1)); err != nil {
            tx.Rollback(); return err
        }
        if err := tx.Commit(); err != nil { return err }
        v++
    }
    return nil
}
```

**规范**：

- **必须**每个迁移单独事务。
- **必须**迁移前备份数据库（实测有 `BackupLegacyArtifacts`）。
- **禁止**在迁移里依赖网络或用户交互。
- **应该**迁移是幂等的：重复跑不出错。

---

## 5. 打开方式：读写与只读分离

实测有两套打开函数：

```
store.Open(ctx)              读写
store.OpenReadOnlyContext()  只读
```

只读连接参数（从二进制里读到的）：

```
mode=ro&_pragma=busy_timeout(1000)&_pragma=mmap_size(0)
```

**为什么重要**：

- 多个进程（CLI + MCP）同时访问同一个库
- 只读查询不应该拿到写锁
- `busy_timeout` 避免瞬间冲突就报错

**规范**：

- **必须**让查询类命令用只读连接。
- **必须**设置 `busy_timeout`。
- **应该**启用 WAL 模式（`PRAGMA journal_mode=WAL`），提高并发读性能。

---

## 6. 多租户隔离（重要）

实测存在一张元数据表（从 SQL 查询里读到字段）：

```sql
SELECT schema_version, client_profile, source_slug,
       expected_identity, observed_identity,
       credential_fingerprint, last_gate_at, migration_provenance
FROM pp_tenant_meta
```

核心思路：

```
expected_identity  期望连的是谁（来自 client-profile）
observed_identity  实际连到了谁（来自服务器响应）
credential_fingerprint  凭证指纹
```

三者不匹配时，**拒绝使用本地库**。

实测错误文案：

```
database tenant metadata does not match the verified client profile
tenant artifact conflict: --%s must use the selected client profile path
```

**为什么必须做**：本地库里混着 A 客户的数据，如果连到 B 客户还继续用，就会把 A 的数据写到 B。这是多租户工具的**头号事故**。

**规范**：

- **必须**在本地库里记录"这份数据属于哪个租户/环境"。
- **必须**在每次使用本地数据前校验身份指纹。
- **禁止**在身份不匹配时"尽力而为"地继续——直接拒绝。
- **应该**把数据库按 profile 分目录，物理隔离比逻辑校验更可靠。

---

## 7. 一致性策略

镜像必然有不一致的可能。选一种明确的策略：

| 策略 | 说明 | 适用 |
|---|---|---|
| 读时校验 | 读之前比 `synced_at` 和 `--max-age` | 默认推荐 |
| 写后失效 | 写操作成功后删除相关缓存 | 必须做 |
| 定时重建 | 定期全量 sync | 兜底 |

**规范**：

- **必须**在写操作成功后失效或更新对应缓存。
- **必须**在数据过旧时提示用户，而不是静默返回旧数据。
- **应该**把"数据新不新"做成可查询的状态，而不是靠用户猜：

```
wyn-cli workflow status     # 显示每个资源最后同步时间
```

---

## 8. 只读 SQL 能力

实测有 `sql` 工具：

> Run read-only SQL against local database. Use for ad-hoc analysis, aggregations, and joins across synced resources. Requires sync first.

**规范**：

- **必须**只允许 `SELECT`（拒绝 INSERT/UPDATE/DELETE/DDL）。
- **必须**限制返回行数和超时。
- **应该**用 SQLite 的 `PRAGMA query_only=ON` 做硬约束，而不是靠字符串匹配。
- **禁止**允许 `ATTACH` 到外部数据库文件。

```go
db.Exec("PRAGMA query_only = ON")
ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
defer cancel()
rows, err := db.QueryContext(ctx, userSQL)
```

---

## 9. 数据库体积控制

实测目录里 `data.db` 初始 147KB。膨胀来自 `resources.data`。

**规范**：

- **应该**对超大字段做截断或跳过（比如图片 base64）
- **应该**定期 `VACUUM`（但要避开高峰期）
- **必须**让用户能一键清空：`wyn-cli sync --purge` 或删库重建
- **可以**按资源类型分库，避免单文件过大

---

## 10. 最小骨架

```go
func Open(ctx context.Context, path string, readonly bool) (*Store, error) {
    dsn := path
    if readonly {
        dsn = "file:" + path + "?mode=ro&_pragma=busy_timeout(1000)"
    } else {
        dsn = "file:" + path + "?_pragma=busy_timeout(1000)&_pragma=journal_mode(WAL)"
    }
    db, err := sql.Open("sqlite", dsn)
    if err != nil { return nil, err }
    db.SetMaxOpenConns(1)   // SQLite 写入串行

    if !readonly {
        if err := migrate(db); err != nil { return nil, err }
        if err := verifyTenant(db); err != nil { return nil, err }
    }
    return &Store{db: db}, nil
}
```

---

## 检查清单

- [ ] 主键是否包含资源类型，避免跨类型 ID 冲突？
- [ ] FTS 分词是否适合目标语言（中文需要 trigram）？
- [ ] 同步状态是否记录"是否完整结束"？
- [ ] 是否有 `--max-pages` 之类的保护？
- [ ] 迁移是否单事务、可重入、会备份？
- [ ] 查询是否走只读连接？
- [ ] 本地库是否绑定租户身份，不匹配时拒绝？
- [ ] 写操作后是否失效缓存？
- [ ] 用户 SQL 是否用 `query_only` 硬限制？
