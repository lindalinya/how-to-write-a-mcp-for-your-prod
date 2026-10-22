# 09 - 版本兼容层

> 一个 CLI 要对接多个服务器版本时，最容易出的问题是"命令存在但调不通"。
> 这一层要做的，是把"不支持"提前变成"不存在"。

---

## 1. 问题

```
CLI 支持 9.1 和 10.0
用户连的是 9.1
某个命令是 10.0 才有的
```

三种处理方式：

| 方式 | 用户看到 | 评价 |
|---|---|---|
| 直接调用 | 404 / 500 | 最差，用户不知道为什么 |
| 调用后报错 | "此接口需要 10.0" | 一般，用户已经浪费了一次操作 |
| **命令不存在** | 补全里看不到，`which` 搜不到 | 最好 |

实测选择第三种。

---

## 2. 实测的兼容描述

`context` 工具返回：

> Compatible with v9.1 (312/319 endpoints available; 7 v10+ endpoints gated)

信息量很大：

- 总端点 319
- 当前版本可用 312
- 被门控 7 个

**规范**：

- **必须**把"当前可用多少端点/总共多少"作为可查询的状态暴露出去。
- **应该**在 `doctor` 或 `context` 里展示，方便排障。

---

## 3. 版本探测：多信号融合

实测有三条正则线索：

```
assetVersionRegex   从静态资源文件名里找版本
bodyVersionRegex    从响应体里找版本
queryVersionRegex   从 HTML/查询串里找版本
```

以及一组函数：

```
DetectServerVersion   主流程
ResolveVersion        解析成可比较的结构
ParseVersion          版本字符串 → Version
RequiredVersion       某个端点要求的最低版本
GatedEndpoints        过滤当前不可用的端点
GatedPaths            按路径过滤
CacheDuration         探测结果的缓存时长
loadCachedVersion / saveCachedVersion
```

### 3.1 探测顺序（建议）

```
1. 显式参数 --server-version 9.1      ← 最高优先级，跳过探测
2. 配置文件里手写的 version           ← 用户已经确认过
3. 缓存（未超过 CacheDuration）        ← 避免每次请求都探
4. 主动探测：
   a. 调一个轻量的版本接口（如果有）
   b. 读静态资源文件名里的指纹
   c. 读响应头/响应体里的版本串
5. 全都失败 → 保守模式：只开放"通用端点"
```

**规范**：

- **必须**允许用户手动指定版本，跳过探测。
- **必须**缓存探测结果。
- **必须**在探测失败时进入**保守模式**，而不是乐观地全开。
- **应该**记录版本来源（`server_version_source`），方便解释行为差异。

实测 `context` 里确实有 `server_version_source` 字段。

### 3.2 版本比较

不要用字符串比较（`"10.0" < "9.1"` 是错的）。

```go
type Version struct {
    Major, Minor, Patch int
    Raw                 string
}

func ParseVersion(s string) (Version, error) {
    // 支持 "9.1" / "10.0.0" / "v9.1.3"
}

func (v Version) Compare(o Version) int {
    switch {
    case v.Major != o.Major:
        return cmp(v.Major, o.Major)
    case v.Minor != o.Minor:
        return cmp(v.Minor, o.Minor)
    default:
        return cmp(v.Patch, o.Patch)
    }
}
```

**规范**：

- **必须**按数字段比较。
- **必须**处理缺省段（`9.1` == `9.1.0`）。
- **应该**支持预发布后缀（`10.0.0-beta`），排序时低于正式版。

---

## 4. 端点门控的两种表达

### 方式 A：最小版本（推荐）

```yaml
endpoint: chat.analysis
compat:
  min_version: "10.0"
```

含义：低于 10.0 就隐藏。

### 方式 B：版本黑名单

```yaml
compat:
  gated_in: ["9.1", "9.2"]
```

含义：这些版本隐藏，其它都开放。

**规范**：

- **应该**默认用 `min_version`，语义更清晰。
- **可以**对"被移除的老接口"用 `max_version`。
- **禁止**在命令执行代码里写版本 if 分支。

---

## 5. 门控发生在什么阶段

```
规格加载
   │
   ▼
探测版本（缓存）
   │
   ▼
过滤端点集  ◄── GatedEndpoints / GatedPaths
   │
   ├──► 注册 CLI 命令（不支持的直接不注册）
   ├──► 注册 MCP 工具（不支持的直接不列）
   └──► 生成 agent-context（tool_surface 只含可用的）
```

**规范**：

- **必须**在**注册阶段**过滤，而不是执行阶段。
- **必须**保证 CLI、MCP、context 三处看到的是**同一个过滤结果**。
- **应该**在 `doctor` 里显示"哪些端点因为版本被隐藏"。

---

## 6. 版本变化时的行为

| 场景 | 期望行为 |
|---|---|
| 服务器升级了 | 下次探测自动解锁新端点 |
| 服务器降级了 | 缓存要失效，避免继续调用新接口 |
| 网络中间设备伪装版本 | 以实际调用结果为准，失败后降级缓存 |
| 用户切了实例 | 立即重新探测，不能用上一个实例的缓存 |

**规范**：

- **必须**把版本缓存和**实例 + 用户**绑定，不能全局共享。
- **必须**在切换实例时失效缓存。
- **应该**在连续 N 次"版本不支持"错误后，主动重新探测。

---

## 7. 缓存设计

```go
type VersionCache struct {
    Instance   string    `json:"instance"`
    User       string    `json:"user"`
    Version    string    `json:"version"`
    Source     string    `json:"source"`      // probe | config | manual
    DetectedAt time.Time `json:"detected_at"`
}

const CacheDuration = 24 * time.Hour
```

**规范**：

- **必须**缓存键包含实例标识。
- **必须**有 TTL。
- **应该**把 `source` 也存下来，便于解释。
- **禁止**把版本缓存放进多租户共享的位置。

---

## 8. 兼容矩阵的维护

当版本变多时，手工维护会失控。建议：

```
规格里写 min_version
     ↓
生成兼容矩阵（CI）
     ↓
对比上一次发布，列出：
  · 新增端点
  · 移除端点
  · 最低版本变化的端点
```

**规范**：

- **必须**在发布前生成兼容性 diff。
- **应该**把"某个端点最低版本变化"视为破坏性变更，需要评审。
- **可以**对每个版本跑一遍 smoke test（只调只读端点）。

---

## 9. 降级策略

当版本探测不确定时：

```
保守模式：
  · 只开放 min_version <= 最低支持版本的端点
  · 明确告知用户当前是"保守模式"
  · 提供 --server-version 让用户手动解锁
```

**规范**：

- **必须**宁可不给，也不要给了调不通。
- **必须**让用户知道自己在保守模式。
- **应该**在保守模式下把探测失败的原因说清楚。

---

## 10. 最小实现

```go
type Compat struct {
    server   Version
    source   string
    gated    map[string]bool
}

func (c *Compat) Filter(eps []Endpoint) []Endpoint {
    out := make([]Endpoint, 0, len(eps))
    for _, e := range eps {
        if c.server.Compare(e.MinVersion) >= 0 {
            out = append(out, e)
        }
    }
    return out
}

func Resolve(ctx context.Context, cfg Config, opt Options) (*Compat, error) {
    if opt.ServerVersion != "" && opt.ServerVersion != "auto" {
        v, err := ParseVersion(opt.ServerVersion)
        return &Compat{server: v, source: "manual"}, err
    }
    if cfg.Version != "" {
        v, err := ParseVersion(cfg.Version)
        return &Compat{server: v, source: "config"}, err
    }
    if cached, ok := loadCache(cfg.Instance, cfg.User); ok && time.Since(cached.DetectedAt) < CacheDuration {
        return &Compat{server: cached.Version, source: "cache"}, nil
    }
    v, src, err := detect(ctx, cfg)
    if err != nil {
        return &Compat{server: MinSupported, source: "conservative"}, nil
    }
    saveCache(cfg.Instance, cfg.User, v, src)
    return &Compat{server: v, source: src}, nil
}
```

---

## 检查清单

- [ ] 不支持的端点是否在**注册阶段**就被移除？
- [ ] CLI / MCP / context 三处是否看到同一份过滤结果？
- [ ] 版本比较是否按数字段，而不是字符串？
- [ ] 是否支持手动 `--server-version` 覆盖？
- [ ] 版本缓存是否绑定到实例 + 用户？
- [ ] 探测失败时是否进入保守模式？
- [ ] 是否暴露"当前版本来源"和"可用端点数量"？
- [ ] 是否在发布前生成兼容性 diff？
