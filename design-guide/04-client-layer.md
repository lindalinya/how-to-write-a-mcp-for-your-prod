# 04 - 传输层（client）

> 这一层决定了整套工具"稳不稳"。业务命令可以生成，但传输层必须手写。

---

## 1. 职责边界

```
client 负责：
  · 认证头注入
  · 请求编码（JSON / multipart / form / 文件）
  · 超时与取消
  · 重试与退避
  · 缓存（条件请求）
  · 限流（自适应）
  · 重定向安全
  · 响应体安全
  · 错误分类与包装
  · 二进制流（下载/上传）

client 不负责：
  · 输出格式化（属于输出层）
  · 本地存储（属于 store）
  · 版本判断（属于 compat）
```

**规范**：命令包（`internal/cli/*`）**禁止**直接使用 `net/http`。

---

## 2. 认证

实测模型：

```json
{
  "mode": "bearer_token",
  "env_vars": [
    {"name": "MYCLI_TOKEN", "kind": "per_call", "required": true, "sensitive": true}
  ]
}
```

凭证来源优先级（建议）：

```
1. 显式参数（极少用）
2. 环境变量（MYCLI_TOKEN）
3. 配置文件（~/.instances.yml 的 users[].token）
4. 交互式提示（且只在 TTY 下）
```

**必须遵守**：

- 凭证**不得**出现在日志、错误信息、`--dry-run` 输出里。
- 凭证文件必须原子写 + 私有权限：

```go
// 实测存在：AtomicWritePrivateFile
func AtomicWritePrivateFile(path string, data []byte) error {
    dir := filepath.Dir(path)
    tmp, err := os.CreateTemp(dir, ".tmp-*")
    if err != nil { return err }
    defer os.Remove(tmp.Name())

    if err := tmp.Chmod(0o600); err != nil { return err }

    if _, err := tmp.Write(data); err != nil { return err }
    if err := tmp.Sync(); err != nil { return err }
    if err := tmp.Close(); err != nil { return err }
    return os.Rename(tmp.Name(), path)
}
```

- 启动时检测"占位符凭证"并给出可操作的错误（实测 `ErrPlaceholderCredential`）：

```
Set it with: echo "$TOKEN" | mycli auth set-token
or export MYCLI_TOKEN="your-token-here"
```

---

## 3. 请求构造

### 3.1 路径占位符

MCP 里有一个 `mycli_execute` 工具，说明是：

> Params are passed as a JSON object; path placeholders and query strings are resolved automatically.

实现要点（依赖 `uritemplate`）：

```go
func buildURL(base, pathTemplate string, params map[string]any) (string, error) {
    // /api/v2/dashboards/{id}
    u := base + pathTemplate
    for k, v := range params {
        if strings.Contains(u, "{"+k+"}") {
            u = strings.ReplaceAll(u, "{"+k+"}", url.PathEscape(fmt.Sprint(v)))
            delete(params, k)     // 已消费的参数不再进 query
        }
    }
    if len(params) > 0 {
        q := url.Values{}
        for k, v := range params { q.Set(k, fmt.Sprint(v)) }
        u += "?" + q.Encode()
    }
    return u, nil
}
```

**规范**：

- **必须**对路径参数做 `PathEscape`，防止 `/` 和 `..` 注入。
- **必须**把未消费的参数自动转成 query string，降低调用方心智负担。

### 3.2 请求体模式

实测支持四种：

| 模式 | 用途 | 命令里的表现 |
|---|---|---|
| `json` | 普通对象 | `--stdin` / `-f file.json` |
| `multipart` | 文件上传 | `-f ./sales.xlsx` |
| `form` | 表单 | 少量字段 |
| `raw/file` | 二进制流 | 下载/上传 |

**规范**：

- **必须**支持从 stdin 读 JSON（`--stdin`），避免 AI 写临时文件。
- **应该**支持从文件读（`-f`），避免超长命令行。

---

## 4. 重试

实测存在 `clientMaxRetries` 和 `Retry-After` 处理。

推荐策略：

```go
type RetryPolicy struct {
    Max       int           // 建议 3
    Base      time.Duration // 建议 200ms
    MaxDelay  time.Duration // 建议 5s
    RetryOn   []int         // 429, 500, 502, 503, 504
}

func (p RetryPolicy) Delay(attempt int, resp *http.Response) time.Duration {
    if resp != nil {
        if s := resp.Header.Get("Retry-After"); s != "" {
            if secs, err := strconv.Atoi(s); err == nil {
                return time.Duration(secs) * time.Second
            }
        }
    }
    d := p.Base << attempt
    if d > p.MaxDelay { d = p.MaxDelay }
    return d + jitter(d)   // ±20% 抖动
}
```

**规范**：

- **必须**尊重 `Retry-After`。
- **必须**只重试幂等方法（GET/PUT/DELETE）或明确带幂等键的 POST。
- **应该**加抖动，避免惊群。
- **禁止**重试 4xx（除 429）。

---

## 5. 缓存（条件请求）

实测可见 `If-None-Match`、`Last-Modified`、`Cache-Control`、`max-age`。

设计要点：

```
第一次请求 → 存下 ETag 和 Last-Modified
第二次请求 → 带 If-None-Match / If-Modified-Since
  304      → 用本地副本，不消耗带宽
  200      → 更新副本
```

**规范**：

- **必须**让缓存可以被 `--no-cache` 绕过。
- **必须**让用户能设置可接受的数据陈旧度：`--max-age 10m`。
- **应该**在拒绝使用过期缓存时给出可操作的提示：

```
local store data is 3h old, older than --max-age=10m. Run 'mycli sync' to refresh.
```

- **禁止**缓存认证失败、限流响应。

---

## 6. 限流：自适应而不是写死

实测有两个组件：

```
cliutil.AdaptiveLimiter.ObserveHeaders   → 从响应头学习
cliutil.AdaptiveLimiter.Wait             → 请求前等待
platform.NewEndpointLimiter / EndpointBudget → 按端点类别分配预算
```

设计思路：

```
初始 QPS 保守（比如 5）
每次收到 429 / Retry-After → 降速
连续成功 N 次 → 缓慢回升
每个端点类别单独记账，避免一个接口吃光全局额度
```

**规范**：

- **必须**按端点类别分桶，不要全局一个桶。
- **应该**从响应头学习，而不是让用户手算 QPS。
- **必须**提供 `--rate-limit` 让用户显式设上限。
- **应该**在耗尽时给出明确错误：

```
rate limited: endpoint class users exhausted after 5 attempts
```

---

## 7. 错误分类

这是这一层最值得抄的部分。实测错误类型：

```go
ErrRedirectPrivateDestination      // 重定向到内网 → SSRF 防护
ErrRedirectProtocolDowngrade       // https → http
ErrRedirectUnsupportedScheme       // file:// / gopher:// 等
ErrDecodedBodyTooLarge             // 解压后过大
ErrPlaceholderCredential           // 凭证还是占位符
```

建议的错误分层：

```go
type ErrorKind int

const (
    KindNetwork   ErrorKind = iota  // 可以重试
    KindAuth                        // 需要用户配凭证
    KindForbidden                   // 权限不足，不要重试
    KindNotFound                    // 资源不存在
    KindConflict                    // 并发冲突
    KindRateLimit                   // 等一会再试
    KindVersion                     // 服务器版本不支持
    KindValidation                  // 参数错误，不重试
    KindUnsafe                      // 安全策略拒绝
)

type Error struct {
    Kind     ErrorKind
    Status   int
    Endpoint string
    Message  string
    Hint     string   // 下一步怎么做
    Cause    error
}
```

**规范**：

- **必须**每种错误都带 `Hint`，告诉用户下一步。
- **必须**区分"可重试"和"不可重试"。
- **禁止**把原始响应体（可能含敏感信息）整段拼进错误。

---

## 8. 重定向安全

实测的三个拒绝条件，值得完整照抄：

```go
func checkRedirect(req *http.Request, via []*http.Request) error {
    if len(via) >= 5 {
        return errors.New("stopped after 5 redirects")
    }
    // 1. 协议只能 https（或显式允许的 http 本机）
    if req.URL.Scheme == "http" && via[0].URL.Scheme == "https" {
        return ErrRedirectProtocolDowngrade
    }
    // 2. 拒绝非 http(s) 协议
    if req.URL.Scheme != "https" && req.URL.Scheme != "http" {
        return ErrRedirectUnsupportedScheme
    }
    // 3. 拒绝跳转到内网/回环
    if isPrivateIP(req.URL.Hostname()) {
        return ErrRedirectPrivateDestination
    }
    // 4. 不要把认证头带到别的域名
    if req.URL.Host != via[0].URL.Host {
        req.Header.Del("Authorization")
    }
    return nil
}
```

**规范**：

- **必须**限制重定向次数。
- **必须**在跨域重定向时删除认证头。
- **必须**拒绝跳转到内网地址。
- **禁止**提供"关闭重定向检查"的开关。

---

## 9. 响应体安全

```go
const maxDecodedBody = 64 << 20 // 64MB

func safeRead(resp *http.Response) ([]byte, error) {
    var r io.Reader = resp.Body
    switch resp.Header.Get("Content-Encoding") {
    case "gzip":
        gz, err := gzip.NewReader(resp.Body)
        if err != nil { return nil, err }
        defer gz.Close()
        r = gz
    case "deflate":
        // ...
    }
    lr := io.LimitReader(r, maxDecodedBody+1)
    b, err := io.ReadAll(lr)
    if err != nil { return nil, err }
    if len(b) > maxDecodedBody {
        return nil, ErrDecodedBodyTooLarge   // 压缩炸弹
    }
    return b, nil
}
```

**规范**：

- **必须**限制**解压后**的大小，不是压缩前的大小。
- **应该**对 HTML 错误页做折叠（实测 `collapseHTMLErrorBody`），只提取 `<title>`，避免把整页 HTML 塞进日志。

---

## 10. 上传与下载

实测能力：

- `UnwrapBinaryResponse`：从包装过的响应里取出二进制
- `StreamingHTTPClient`：流式下载，不整段进内存
- `binaryResponseHeaderValue`：从 header 取文件名/长度

**规范**：

- **必须**流式处理大文件（下载直接写磁盘，上传用 `io.Pipe`）。
- **必须**在下载前检查 `Content-Length` 是否超过上限。
- **应该**支持断点续传（`Range`）——不是必需，但导出大报表时很有用。

---

## 11. 一个最小可用的 client 骨架

```go
type Client struct {
    base     string
    hc       *http.Client
    token    string
    limiter  *AdaptiveLimiter
    retry    RetryPolicy
    cacheDir string
}

func (c *Client) Do(ctx context.Context, req *Request) (*Response, error) {
    for attempt := 0; ; attempt++ {
        if err := c.limiter.Wait(ctx, req.EndpointClass); err != nil {
            return nil, err
        }

        hr, err := c.build(req)
        if err != nil { return nil, err }

        if etag := c.loadETag(req); etag != "" {
            hr.Header.Set("If-None-Match", etag)
        }

        resp, err := c.hc.Do(hr.WithContext(ctx))
        if err != nil {
            if attempt < c.retry.Max && isRetryableNet(err) {
                time.Sleep(c.retry.Delay(attempt, nil)); continue
            }
            return nil, wrap(KindNetwork, err)
        }

        c.limiter.ObserveHeaders(resp.Header)

        switch {
        case resp.StatusCode == 304:
            return c.fromCache(req)
        case resp.StatusCode == 429 && attempt < c.retry.Max:
            time.Sleep(c.retry.Delay(attempt, resp)); resp.Body.Close(); continue
        case resp.StatusCode >= 500 && attempt < c.retry.Max:
            time.Sleep(c.retry.Delay(attempt, resp)); resp.Body.Close(); continue
        }

        body, err := safeRead(resp)
        if err != nil { resp.Body.Close(); return nil, err }
        defer resp.Body.Close()

        if resp.StatusCode >= 400 {
            return nil, classify(resp.StatusCode, body, req.Endpoint)
        }

        c.saveETag(req, resp.Header.Get("ETag"))
        return &Response{Status: resp.StatusCode, Body: body, Header: resp.Header}, nil
    }
}
```

---

## 检查清单

- [ ] 命令包里没有直接调用 `net/http`？
- [ ] 凭证是否可能出现在日志/错误里？
- [ ] 重试是否只针对幂等方法？是否尊重 `Retry-After`？
- [ ] 缓存是否可绕过？是否有陈旧度提示？
- [ ] 限流是否按端点分类？是否有明确错误？
- [ ] 重定向是否拒绝内网和协议降级？
- [ ] 解压后是否存在体积上限？
- [ ] 大文件是否流式处理？
- [ ] 每个错误是否带"下一步"？
