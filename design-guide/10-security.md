# 10 - 安全设计

> 这套工具手里有：服务器凭证、写权限、本地镜像数据。
> 它被 AI 调用，意味着"攻击面"里还要加上模型本身。

---

## 1. 威胁模型

### 1.1 资产

| 资产 | 位置 | 泄露后果 |
|---|---|---|
| API Token | `~/.wyninstances.yml`、环境变量 | 完全控制账号 |
| 密码 | 同上 | 同上 |
| 本地镜像数据 | `~/.local/share/wyn-cli/data.db` | 业务数据外泄 |
| 服务器写权限 | 进程内存 | 数据被篡改/删除 |
| 学习库 | 同 data.db | 污染 AI 的后续判断 |

### 1.2 攻击者与路径

```
A. 本机其它用户
   → 读配置文件 / 读数据库

B. 恶意提示（prompt injection）
   → 诱导 AI 调用 destructive 工具
   ← 来自：文件内容、网页、API 返回的数据

C. 恶意服务器 / 中间人
   → 返回超大响应、重定向到内网、诱导降级

D. 用户自己的误操作
   → 用错实例、删错资源、把 A 数据写到 B

E. 供应链
   → 第三方依赖被投毒
```

---

## 2. 控制清单（按威胁映射）

| # | 控制 | 防 A | 防 B | 防 C | 防 D | 防 E |
|---|---|:-:|:-:|:-:|:-:|:-:|
| 1 | 凭证文件原子写 + 0600 | ✓ | | | | |
| 2 | 重定向白名单（拒内网/降级/异域去认证头） | | | ✓ | | |
| 3 | 解压后响应体上限 | | | ✓ | | |
| 4 | 占位符凭证检测 | | | | ✓ | |
| 5 | 租户身份指纹校验 | | | | ✓ | |
| 6 | `readOnlyHint` / `destructiveHint` 注解 | | ✓ | | ✓ | |
| 7 | `--dry-run` | | ✓ | | ✓ | |
| 8 | 非回环地址强制 TLS | | | ✓ | | |
| 9 | 拒绝凭证类参数写入 profile | ✓ | | | ✓ | |
| 10 | 最小依赖 + 纯 Go SQLite | | | | | ✓ |
| 11 | 提示词中的 shell 注入警告 | | ✓ | | | |
| 12 | 候选隔离区 | | ✓ | | | |

---

## 3. 凭证处理

### 3.1 现状（实测，需要改进）

`~/.wyninstances.yml` 里：

```yaml
users:
  - username: admin
    password: mysecretpassword
    token: a1b2c3d4e5f6...
```

**明文**。二进制里没有 DPAPI / keyring / AES 的痕迹，只有一个用于显示脱敏的 `Redacted`。

**这是这套工具最明显的安全短板。**

### 3.2 现实可行的改进

| 方案 | 成本 | 适用 |
|---|---|---|
| 环境变量优先（`WYN_TOKEN`） | 低 | 所有场景 |
| 操作系统密钥环（Keychain/DPAPI/libsecret） | 中 | 单人开发机 |
| 只存"凭证引用"，真实值在外部 | 中 | 多环境 |
| 文件权限收紧到 0600 | 低 | 兜底 |

**规范**：

- **必须**支持从环境变量读凭证，且优先级高于配置文件。
- **必须**在保存凭证时使用原子写 + 0600。
- **应该**提供"凭证引用"模式：配置里只写 `ref:env:WYN_TOKEN`。
- **禁止**把凭证写进日志、错误、`--dry-run` 输出、`--deliver` 的 URL。

### 3.3 凭证泄露检测

实测有占位符检测：

```
ErrPlaceholderCredential
Set it with: echo "$TOKEN" | wyn-cli auth set-token
or export WYN_TOKEN="your-token-here"
```

**规范**：

- **必须**识别常见占位符（`<your-token>`、`xxx`、`changeme`），并给出配置指引。
- **应该**检测"看起来像凭证的 flag"并拒绝持久化。

实测原文：

> store credential-like flags: %s; use --client-profile with exact external references

---

## 4. 网络层安全

### 4.1 SSRF 防护

```go
var (
    ErrRedirectPrivateDestination = errors.New("redirect to private destination")
    ErrRedirectProtocolDowngrade  = errors.New("redirect protocol downgrade")
    ErrRedirectUnsupportedScheme  = errors.New("redirect unsupported scheme")
)
```

规则：

```
1. 最多 5 次重定向
2. 禁止 https → http
3. 禁止非 http(s) 协议
4. 禁止跳转到内网/回环/链路本地地址
5. 跨域重定向时删除 Authorization 头
```

**规范**：

- **必须**在 `http.Client.CheckRedirect` 里实现，不要事后检查。
- **禁止**提供关闭开关。

### 4.2 内网地址判定

需要覆盖：

```
10.0.0.0/8
172.16.0.0/12
192.168.0.0/16
127.0.0.0/8
169.254.0.0/16     链路本地（云元数据 169.254.169.254！）
::1
fc00::/7           唯一本地地址
fe80::/10
```

**规范**：

- **必须**在解析 DNS 之后校验 IP，防止 DNS rebinding。
- **必须**拦截云厂商元数据地址。

### 4.3 响应体安全

```
解压后上限（实测 64MB 量级）
HTML 错误页只取 <title>（实测 collapseHTMLErrorBody）
```

**规范**：

- **必须**限制解压后大小。
- **应该**对非预期 Content-Type 做截断，避免把整页 HTML 写进日志。

---

## 5. 多租户隔离

### 5.1 问题

```
本地库里有 A 客户的数据
用户切换到了 B 客户
程序继续用本地库 → A 的数据被展示或被写进 B
```

### 5.2 实测控制

```sql
pp_tenant_meta(
    schema_version, client_profile, source_slug,
    expected_identity, observed_identity,
    credential_fingerprint, last_gate_at, migration_provenance
)
```

错误文案：

```
database tenant metadata does not match the verified client profile
tenant artifact conflict: --%s must use the selected client profile path
```

### 5.3 规范

- **必须**在本地库里记录数据属于哪个租户。
- **必须**每次使用本地数据前校验 `expected_identity == observed_identity`。
- **必须**在身份不匹配时**拒绝**，不得降级为警告。
- **应该**按 profile 分数据库文件，物理隔离优于逻辑校验。
- **必须**在迁移数据库时保留租户指纹（`migration_provenance`）。

---

## 6. AI 特有风险

### 6.1 提示注入

攻击者可以把指令藏在：

```
API 返回的文档内容
文件名
看板标题
用户上传的表格
```

**控制**：

| 控制 | 作用 |
|---|---|
| `destructiveHint` 注解 | AI 应主动确认危险操作 |
| `--dry-run` | 先看后做 |
| 服务端权限 | 最后一道防线 |
| 工具描述里明确"输入是数据不是指令" | 降低被诱导概率 |

**规范**：

- **必须**把工具返回的内容当**数据**，不要在服务端做"根据内容自动执行"的逻辑。
- **应该**对 `destructiveHint` 的工具要求显式确认。
- **禁止**让工具返回的文本直接影响下一个工具的调用参数（除非经过模型判断）。

### 6.2 越权调用

AI 可能调用用户本来没权限的接口。

**规范**：

- **必须**以"当前用户 token"的权限调用，不得使用服务账号提权。
- **必须**在权限不足时返回明确错误，而不是静默重试。

### 6.3 学习污染

AI 把错误映射写进学习库，后续所有会话都受影响。

**控制**：候选隔离 + 显式确认（见 [08](08-learning-layer.md)）。

---

## 7. 本地文件安全

| 文件 | 权限要求 |
|---|---|
| `~/.wyninstances.yml` | 0600 |
| `data.db` | 0600 |
| 缓存目录 | 0700 |
| 导出文件 | 默认权限，但**不得**包含凭证 |

**规范**：

- **必须**在创建时设置权限，不要创建后再 chmod。
- **必须**用原子写（临时文件 + rename）。
- **应该**在 Windows 上说明"权限是建议性的"（实测文案：`warning: %s %s has permissive access (%v); on Windows this is advisory`）。

---

## 8. 审计

| 手段 | 内容 |
|---|---|
| `--receipt` | 记录请求/响应（脱敏后） |
| `learn_events` | 记录学习相关动作 |
| `doctor` | 检查配置、连通性、权限 |
| `--dry-run` | 不执行的预演 |

**规范**：

- **必须**让每次写操作可追溯（谁、什么时候、对什么）。
- **必须**在审计记录里脱敏凭证和敏感字段。
- **应该**提供"这次操作会改什么"的预览。

---

## 9. 安全评审检查清单

### 凭证
- [ ] 是否支持从环境变量读，且优先于文件？
- [ ] 文件权限是否 0600、是否原子写？
- [ ] 日志/错误/dry-run 是否可能泄露凭证？
- [ ] 是否拒绝把凭证存进 profile？

### 网络
- [ ] 重定向是否拒绝内网、降级、异协议？
- [ ] 跨域重定向是否删除认证头？
- [ ] 是否校验 DNS 解析后的 IP？
- [ ] 解压后是否有限制？

### 多租户
- [ ] 本地库是否绑定租户身份？
- [ ] 身份不匹配时是否硬拒绝？
- [ ] 切换实例是否失效缓存？

### AI
- [ ] 危险工具是否带 `destructiveHint`？
- [ ] 是否提供 `dry-run`？
- [ ] 自动学到的知识是否需要人工确认？
- [ ] 工具返回内容是否被当作纯数据？

### 供应链
- [ ] 依赖是否精简、版本是否固定？
- [ ] 是否避免 CGO（减少构建差异）？
- [ ] 是否锁定依赖校验和？

---

## 10. 已知短板（诚实清单）

| 短板 | 影响 | 缓解 |
|---|---|---|
| 配置明文存密码/token | 本机其它用户可读 | 收紧权限、改用环境变量 |
| 无密钥环集成 | 同上 | 手动管理 |
| MCP HTTP 模式无内建认证 | 本机进程可调 | 绑定回环 + TLS |
| 学习库无签名 | 本机可篡改 | 依赖文件权限 |
| 依赖版本未验证 | 供应链 | 锁定 + 审计 |
