# 06 - 输出管线

> 同一条数据，人要看表格，脚本要 JSON，AI 要精简字段，导出要文件。
> 输出层做不好，前面所有设计都白费。

---

## 1. 格式矩阵（实测）

| 参数 | 输出 | 用途 |
|---|---|---|
| 默认 | 彩色表格（TTY） | 人看 |
| `--json` | JSON | 脚本、AI |
| `--csv` | CSV | Excel、分析 |
| `--plain` | 制表符分隔 | `awk`/`cut` |
| `--quiet` | 每行一个值 | shell 变量 |
| `--human-friendly` | 彩色 + 富格式 | 显式要求时 |
| `--no-color` | 无颜色 | CI、日志 |
| `--compact` | 只保留关键字段 | 省 token |
| `--select a,b,c` | 指定字段 | 精确取数 |
| `--deliver` | 送到文件/webhook | 大批量 |

**规范**：

- **必须**保证同一份数据在不同格式下的**内容一致**，只是表现形式不同。
- **必须**让格式选择是全局参数，命令实现不感知。
- **禁止**某个命令只支持一种格式。

---

## 2. 输出管线的三段式

```
业务结果（结构体）
      │
      ▼
  投影（projection）     --select / --compact
      │
      ▼
  渲染（render）         table / json / csv / plain / quiet
      │
      ▼
  投递（deliver）        stdout / file / webhook
```

**为什么这么分**：

- 业务命令只返回结构体，不知道用户要什么格式
- 投影和渲染可以独立测试
- 投递是 IO，可以单独处理错误

---

## 3. 字段投影

### 3.1 `--compact`

实测语义："Return only key fields (id, name, status, timestamps) for minimal token usage."

建议的紧凑字段集合：

```
id, name, status, created_at, updated_at
```

**规范**：

- **必须**在规格里标记哪些字段是"关键字段"，而不是运行时猜。
- **应该**让 `--compact` 和 `--select` 可以组合使用（select 优先）。

### 3.2 `--select`

```
--select id,name,status
--select id,users.email     # 支持嵌套路径
```

**规范**：

- **必须**在字段不存在时报错，而不是静默返回空。
- **应该**支持点号路径取嵌套字段。
- **可以**支持重命名（`id:ident`）。

---

## 4. 渲染细节（表格最容易被忽略）

一个合格的表格渲染器需要处理：

| 问题 | 处理方式 |
|---|---|
| 中文宽度 | 用 `go-runewidth` 计算显示宽度，不是字节数 |
| 超长单元格 | 截断 + 省略号，或换行 |
| 终端宽度 | 读 `COLUMNS`，动态分配列宽 |
| 空值 | 显示 `-` 或空，不要显示 `<nil>` |
| 布尔 | 显示 `true/false` 或 `yes/no`，不要 `1/0` |
| 时间 | 统一 ISO8601 或本地时间，二选一 |
| 大数字 | 用 `go-humanize` 显示 `1.2 MB` |

**规范**：

- **必须**按**显示宽度**对齐，不是字符数。
- **必须**在非 TTY 时禁用颜色（或尊重 `--no-color`）。
- **应该**在管道输出时自动降级为无颜色、无分页。

---

## 5. stdout 与 stderr 的严格分离

这是脚本可用性的底线：

```
stdout  → 只有数据
stderr  → 日志、进度、警告、错误
```

**规范**：

- **必须**保证 stdout 可以直接 `| jq .`。
- **禁止**把 "Loading..." 之类的进度信息写到 stdout。
- **必须**在错误时仍然输出结构化的空结果（如果用户要 JSON）：

```json
{"matches": []}
```

（实测 `which` 在无匹配时正是这么做的，同时退出码为 2。）

---

## 6. 退出码规范

实测语义：

```
0  成功
2  参数错误 / 无匹配结果
```

建议完整约定：

| 码 | 含义 |
|---:|---|
| 0 | 成功 |
| 1 | 运行时错误（网络、服务器） |
| 2 | 用法错误（参数不合法、无匹配） |
| 3 | 认证失败 |
| 4 | 权限不足 |
| 5 | 资源不存在 |
| 6 | 版本不兼容 |
| 7 | 限流 |
| 8 | 安全策略拒绝 |

**规范**：

- **必须**区分"用法错误"和"运行错误"。
- **必须**让 `--strict` 改变部分失败的处理（实测：默认非关键失败退出 0，`--strict` 恢复为非零）。
- **禁止**用同一个退出码表达多种含义。

---

## 7. `--deliver`：输出不经过 stdout

实测支持：

```
--deliver stdout              默认
--deliver file:./out.json
--deliver webhook:https://...
```

**规范**：

- **必须**在投递成功时 stdout 保持安静（或只输出路径）。
- **必须**在投递失败时退回到 stdout，并给出明确错误。
- **应该**对 webhook 做重试和超时。
- **禁止**把凭证放进 `--deliver` 的 URL。

---

## 8. AI 友好输出

### 8.1 `--agent` 是一个组合开关

实测定义：

```
--agent  =  --json --compact --no-input --no-color
```

这一条设计非常值得抄：AI 不需要记住四个参数。

**规范**：

- **必须**提供一个"一键 AI 模式"。
- **必须**在 AI 模式下关闭所有交互提示。
- **应该**在 AI 模式下默认开启 `--compact`。

### 8.2 错误也要是结构化的

```json
{
  "error": {
    "kind": "version",
    "message": "endpoint requires Wyn 10.0",
    "hint": "re-run with --server-version 10.0 or upgrade the server",
    "endpoint": "chat.analysis"
  }
}
```

**规范**：

- **必须**让错误在 `--json` 下也是合法 JSON。
- **应该**把 `hint` 单独成字段，方便 AI 直接采取动作。

### 8.3 分页信息要显式

```json
{
  "results": [...],
  "has_more": true,
  "next_cursor": "abc123"
}
```

**规范**：

- **必须**告诉调用方还有没有下一页。
- **禁止**让调用方靠"结果数量 < page_size"来推断。

---

## 9. 流式与大结果

实测有 `tail` 命令（轮询式流）和 `--max-pages`。

**规范**：

- **必须**对可能很大的结果设置上限（`--limit`、`--max-pages`）。
- **应该**支持 NDJSON（每行一个 JSON）用于流式处理。
- **必须**在触顶时明确提示数据被截断。

```
{"reason":"max_pages_cap_hit","message":"reached --max-pages cap of 50;
 data may be truncated. Re-run with --max-pages 0 (unlimited)."}
```

---

## 10. 最小实现

```go
type Format string

const (
    FormatTable Format = "table"
    FormatJSON  Format = "json"
    FormatCSV   Format = "csv"
    FormatPlain Format = "plain"
    FormatQuiet Format = "quiet"
)

type Renderer interface {
    Render(w io.Writer, rows []map[string]any) error
}

func Run(rows []map[string]any, opt Options) error {
    rows = project(rows, opt.Select, opt.Compact)

    var w io.Writer = os.Stdout
    var closer io.Closer
    if opt.Deliver != "" && opt.Deliver != "stdout" {
        var err error
        w, closer, err = openSink(opt.Deliver)
        if err != nil { return err }
        defer closer.Close()
    }

    r := pickRenderer(opt.Format, opt.IsTTY && !opt.NoColor)
    return r.Render(w, rows)
}
```

---

## 检查清单

- [ ] 所有命令都支持同一套格式参数？
- [ ] stdout 只有数据，日志走 stderr？
- [ ] 中文表格对齐正确（按显示宽度）？
- [ ] 非 TTY 时自动去色？
- [ ] 退出码是否区分用法错误和运行错误？
- [ ] `--agent` 是否一键开启 AI 模式？
- [ ] 错误在 JSON 模式下是否也是合法 JSON？
- [ ] 分页是否有 `has_more` / `next_cursor`？
- [ ] 大结果是否有上限和截断提示？
