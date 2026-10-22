# 11 - 可靠性与可运维性

> 这一章讲的是"让自动化放心跑"需要的开关。
> 单独看每个参数都不起眼，合起来就是一套运维契约。

---

## 1. 完整全局参数表（实测 27 个）

| 参数 | 类型 | 作用 |
|---|---|---|
| `--agent` | bool | `--json --compact --no-input --no-color` 的简写 |
| `--compact` | bool | 只回关键字段 |
| `--select` | string | 选字段 |
| `--json` | bool | JSON 输出 |
| `--csv` | bool | CSV 输出 |
| `--plain` | bool | 制表符输出 |
| `--quiet` | bool | 每行一个值 |
| `--human-friendly` | bool | 彩色富文本 |
| `--no-color` | bool | 去色 |
| `--deliver` | string | `stdout` / `file:` / `webhook:` |
| `--dry-run` | bool | 只构造不发送 |
| `--yes` | bool | 跳过确认 |
| `--no-input` | bool | 禁用所有交互 |
| `--timeout` | duration | 请求超时（默认 1m） |
| `--server-version` | string | `auto` / `9.1` / `10.0` |
| `--instance` | string | 目标实例名 |
| `--instance-user` | string | 目标用户 |
| `--profile` | string | 应用一组保存的参数 |
| `--client-profile` | string | 租户/客户端隔离 |
| `--data-source` | enum | `auto` / `live` / `local` |
| `--max-age` | duration | 本地数据可接受陈旧度 |
| `--no-cache` | bool | 绕过缓存 |
| `--rate-limit` | string | 主动限速 |
| `--idempotent` | bool | 使用幂等键 |
| `--ignore-missing` | bool | 目标不存在不算失败 |
| `--allow-partial-failure` | bool | 批量中部分失败继续 |
| `--no-learn` | bool | 本次不记录经验 |
| `--receipt` | string | 生成请求/响应回执 |
| `--help-all` | bool | 显示高级/内部参数 |

**规范**：

- **必须**让每个参数在所有命令上可用（全局 flag）。
- **必须**保证 `--no-input` 下程序绝不阻塞等待输入。
- **应该**给每个"危险/消耗型"能力配一个关闭开关。

---

## 2. `doctor`：一条命令回答"能不能用"

实测存在 `mycli doctor --agent`。

建议自检内容：

```
[配置]
  ✓ 找到 ~/.instances.yml
  ✓ 默认实例: local-dev (http://localhost:51980)
  ✗ 用户 admin 的 token 是占位符 → 怎么修
[网络]
  ✓ 服务器可达 (120ms)
  ✓ 版本探测: 9.1 (来源: probe)
  ✗ 7 个端点在当前版本不可用
[本地]
  ✓ data.db 存在 (2.3 MB)
  ✗ 最后同步: 3 天前，超过 --max-age
[权限]
  ✓ 缓存目录可写
  ✗ 配置文件权限过宽（Windows 上为建议性）
```

**规范**：

- **必须**每条检查都给出"通过/失败 + 怎么修"。
- **必须**支持 `--agent` 输出 JSON。
- **应该**在失败时退出非零。
- **禁止**在 `doctor` 里做写操作。

---

## 3. `--dry-run`

语义：**完整构造请求，但不发送**。

实测有 `dry-run preview: %w` 和 "dry-run: would %s"。

**规范**：

- **必须**走到"序列化完成、准备发送"这一步，让用户看到真实请求。
- **必须**脱敏凭证和敏感头。
- **必须**在 dry-run 下不写本地库、不写学习库。
- **应该**支持 `--json` 输出结构化预览。

示例输出：

```json
{
  "dry_run": true,
  "request": {
    "method": "POST",
    "url": "https://api.example.com/api/v2/dashboards",
    "headers": {"Authorization": "Bearer ***", "Content-Type": "application/json"},
    "body_bytes": 4821
  },
  "would": "create dashboard 'Sales Overview'"
}
```

---

## 4. 幂等

实测有 `idempotent` 参数和 `idempotency key` 字符串。

**规范**：

- **必须**为 POST 创建类操作提供幂等键。
- **应该**让幂等键可指定（重试时复用），也可自动生成（首次调用）。
- **必须**在服务端不支持幂等时明确告知，不要假装安全。

```go
if opts.Idempotent {
    key := opts.IdempotencyKey
    if key == "" {
        key = uuid.NewString()
    }
    req.Header.Set("Idempotency-Key", key)
}
```

---

## 5. 部分失败

实测文案：

> %d resource(s) failed but exit code is 0 because the new default treats non-critical failures as warnings.
> Pass `--strict` to restore the old behavior.

这里有一个很实际的产品决策：

```
默认：非关键失败 = 警告，退出 0
--strict：任何失败都退出非零
```

**规范**：

- **必须**在输出里明确"成功几个、失败几个、失败的是哪些"。
- **必须**提供 `--strict` 让 CI 能严格判断。
- **应该**把"失败原因"分类（网络 / 权限 / 不存在）。
- **禁止**静默吞掉失败。

```json
{
  "results": [...],
  "summary": {
    "total": 50,
    "succeeded": 48,
    "failed": 2,
    "failures": [
      {"id": "abc", "reason": "not_found"},
      {"id": "def", "reason": "forbidden"}
    ]
  },
  "strict": false
}
```

---

## 6. 超时与取消

**规范**：

- **必须**有全局超时（实测默认 1 分钟）。
- **必须**允许 `--timeout 0` 表示无限（用于长同步）。
- **必须**响应 `SIGINT` 并做清理。
- **应该**把超时错误和网络错误区分开。

```go
ctx, cancel := context.WithTimeout(context.Background(), opts.Timeout)
defer cancel()

sigCtx, stop := signal.NotifyContext(ctx, os.Interrupt, syscall.SIGTERM)
defer stop()
```

---

## 7. 日志与回执

### 7.1 日志

```
默认：只输出错误到 stderr
--verbose：输出请求摘要
--receipt：输出完整请求/响应记录
```

**规范**：

- **必须**日志只写 stderr。
- **必须**默认安静（成功时什么都不输出）。
- **应该**支持 `MYCLI_DEBUG=1` 之类的环境变量开关。

### 7.2 回执（receipt）

实测有 `NewReceiptWriter`、`receipt-file`、`receipt CLI name is required`。

用途：把每次调用记录下来，用于审计或复现。

```json
{
  "cli": "mycli",
  "version": "0.2.32",
  "timestamp": "2026-10-22T08:00:00Z",
  "command": ["dashboards", "delete", "abc123"],
  "instance": "local-dev",
  "user": "admin",
  "request": {"method": "DELETE", "path": "/api/v2/dashboards/abc123"},
  "response": {"status": 204},
  "duration_ms": 143
}
```

**规范**：

- **必须**脱敏凭证。
- **必须**记录足够的上下文以便复现（命令、实例、用户、版本）。
- **应该**支持追加写入，避免覆盖。

---

## 8. 可观测性

| 手段 | 内容 |
|---|---|
| `learnings stats` | 学习系统效果 |
| `workflow status` | 本地镜像状态 |
| `doctor` | 环境健康 |
| `--receipt` | 单次调用审计 |
| `learn_events` | 行为流水 |

**规范**：

- **应该**每条"状态类"命令都支持 `--json`。
- **应该**在 CI 里定期跑 `doctor`。
- **可以**把 receipt 喂进日志平台。

---

## 9. 故障排查手册

### 现象：命令报 404
```
1. mycli doctor            → 版本对不对
2. mycli --server-version 9.1 <cmd>  → 手动指定试试
3. 检查端点是否被门控
```

### 现象：认证失败
```
1. mycli auth status
2. 检查 token 是否是占位符
3. 检查实例/用户是否选对
```

### 现象：数据不是最新的
```
1. mycli workflow status
2. 加 --data-source live 试试
3. mycli sync --full
```

### 现象：被限流
```
1. 加 --rate-limit 5/s
2. 减少并发
3. 检查是否命中某个端点类别的预算
```

### 现象：连接超时
```
1. --timeout 2m
2. 检查代理环境变量
3. mycli doctor
```

**规范**：

- **必须**让每条错误信息直接对应一个排查步骤。
- **应该**在错误里带上"建议执行的下一条命令"。

---

## 10. 自动化契约

如果你要在 CI / Agent 里用它，应该满足：

```
1. --no-input       永远不阻塞
2. --yes            不弹确认
3. --json           可解析
4. --timeout        有上限
5. 退出码           可判断
6. --dry-run        可预演
7. --idempotent     可重试
8. --receipt        可审计
9. --strict         可严格
10. stderr          只有日志
```

**规范**：

- **必须**保证这 10 条同时成立，才算"可自动化"。
- **应该**在文档里明确承诺这些行为。

---

## 检查清单

- [ ] `--no-input` 下是否绝对不会阻塞？
- [ ] `doctor` 是否覆盖配置、网络、本地、权限四类？
- [ ] `--dry-run` 是否真的不产生副作用？
- [ ] POST 创建类操作是否支持幂等键？
- [ ] 批量操作是否报告成功/失败明细？是否有 `--strict`？
- [ ] 是否有全局超时和信号处理？
- [ ] 日志是否只走 stderr？成功时是否安静？
- [ ] 是否有可审计的回执？
- [ ] 每条错误是否对应一个排查步骤？
