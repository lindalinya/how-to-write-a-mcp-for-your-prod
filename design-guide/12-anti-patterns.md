# 12 - 反模式与代价

> 前面都在讲优点。这一章讲这套设计的软肋，以及做类似系统时容易掉进去的坑。

---

## 一、这套系统本身的代价

### 1. 二进制体积

```
wyn-cli.exe   28 MB
wyn-mcp.exe   33 MB
wynlauncher    9 MB
```

原因：Go 静态编译 + 321 个端点全量打包。

**代价**：下载慢、CI 缓存浪费、容器镜像变大。

**如果重做**：可以把端点定义外置成数据文件（JSON/YAML），运行时加载。代价是启动慢一点、需要管理文件版本。

---

### 2. 生成代码的可读性

524 个命令节点、几百个包目录。

**代价**：

- IDE 索引变慢
- 出问题时，要看规格层，而不是命令包
- code review 时，生成物的 diff 噪音大

**缓解**：

- 生成物单独目录、单独 review
- 保留"从命令反查规格"的能力（实测有 `which`）

---

### 3. 隐式行为太多

自动缓存、自动学习、自动限流、自动重试、自动探测版本。

**代价**：出问题时定位困难。"为什么这条命令没发请求？"可能是缓存。"为什么变慢了？"可能是自适应限流降速了。

**缓解**（实测都有）：

```
--no-cache
--data-source live
--no-learn
--dry-run
doctor
receipt
```

**规范**：**每个隐式行为必须有一个显式关闭开关。**

---

### 4. 版本矩阵

9.1 和 10.0 两套门控，每加一个版本，测试组合就乘一倍。

**缓解**：

- 只在规格里声明 `min_version`
- CI 里跑兼容性 diff
- 只对只读端点做 smoke test

---

### 5. 明文凭证

`~/.wyninstances.yml` 里 password 和 token 是明文。

**代价**：本机其它用户、备份同步、误提交都能泄露。

**这是最该改的一条。**

---

### 6. MCP 精选可能漏

118 个工具之外的需求，只能靠 `wyn-cli_search` + `wyn-cli_execute` 兜底。

**代价**：AI 需要多两步才能完成冷门操作。

**缓解**：兜底工具要好用（实测的 `wyn-cli_execute` 支持自动解析路径占位符）。

---

### 7. 学习系统冷启动

新用户的前几十次调用完全没有收益，`recall` 全部 miss。

**缓解**：预置实体词表（实测 `source='seeded'`）+ 空库短路。

---

## 二、通用反模式（做类似系统时）

### 反模式 1：把 API 端点直接暴露成工具

```
✗ get_user_by_id_and_org_with_options
✓ users_get
```

**判断标准**：工具名里出现 `and`、`with`、`options`，说明你在暴露实现细节。

---

### 反模式 2：每个命令自己实现重试

```
✗ internal/cli/reports_list/retry.go
  internal/cli/dashboards_list/retry.go
✓ internal/client/retry.go
```

**后果**：行为不一致，修一处漏九处。

---

### 反模式 3：在命令代码里判断版本

```go
// ✗
if serverVersion >= "10.0" {
    callNewEndpoint()
} else {
    callOldEndpoint()
}

// ✓ 在规格里声明，注册阶段过滤
```

---

### 反模式 4：把日志写到 stdout

```go
// ✗
fmt.Println("Loading...")
fmt.Println(jsonData)

// ✓
fmt.Fprintln(os.Stderr, "Loading...")
fmt.Println(jsonData)
```

**后果**：`| jq .` 直接失败。

---

### 反模式 5：静默回退

```
用户: --data-source local
程序: 本地没数据，那我偷偷走网络吧
```

**后果**：用户以为离线，实际发了一堆请求。

**正确**：明确报错，或返回空并提示。

---

### 反模式 6：自动学习直接生效

```
AI 观察到一个映射 → 写入正式表 → 下次直接使用
```

**后果**：一次误判永久污染。

**正确**：候选隔离 + 显式确认 + 墓碑。

---

### 反模式 7：用字符串比较版本

```go
if version < "9.10" { }   // "9.10" < "9.9"? 字符串比较会错
```

**正确**：解析成数字段。

---

### 反模式 8：错误信息只说"失败"

```
✗ Error: request failed
✓ Error: 403 forbidden for POST /api/v2/dashboards
       hint: token may lack 'dashboard:write' permission
```

---

### 反模式 9：所有参数必填

```
✗ 12 个必填参数
✓ 1 个必填 + 11 个带默认值
```

**实测参考**：118 个工具里，55 个零必填，62 个只有一个必填。

---

### 反模式 10：把凭证放进 URL

```
✗ --deliver webhook:https://hooks.example.com/abc?token=xxx
✓ 用 header 或环境变量
```

---

### 反模式 11：没有 `--no-input`

**后果**：CI 里卡住直到超时。

---

### 反模式 12：dry-run 只打印"会执行"

```
✗ dry-run: would create dashboard
✓ dry-run: 完整请求（方法/URL/头/体大小）+ 会被影响的对象
```

---

### 反模式 13：把"父分组"当命令统计

```
524 个命令节点 ≠ 524 个功能
其中 321 个是真实端点，60 个是分组
```

**后果**：文档里数字虚高，用户找不到东西。

---

### 反模式 14：只测 happy path

**正确**：至少覆盖

```
· 401 / 403
· 404
· 429（限流）
· 500
· 超时
· 版本不支持
· 部分失败
```

---

### 反模式 15：把用户输入拼进 shell

实测原文专门警告过：

> never interpolate user-controlled text into a shell command line, quoted string, or heredoc:
> an apostrophe breaks quotes, and a body line equal to the delimiter breaks a heredoc.

**正确**：

```go
// ✗
exec.Command("sh", "-c", "recall '"+query+"'")

// ✓
exec.Command("wyn-cli", "recall", query)
```

---

## 三、什么时候【不该】用这套设计

| 场景 | 更合适的做法 |
|---|---|
| 只有 5-20 个接口 | 手写命令，别上生成器 |
| 接口经常剧烈变化 | 生成器维护成本高于收益 |
| 只需要给 AI 用 | 直接做 MCP，不用做 CLI |
| 只需要给人用 | 直接做 CLI，别做 MCP |
| 没有离线需求 | 不用本地 SQLite |
| 团队 < 3 人 | 学习系统、候选机制都是负担 |
| 单版本、不升级 | 不需要兼容层 |

**判断方法**：

```
端点数 × 变化频率 × 消费方数量
```

这个乘积 > 100 时，规格化才划算。

---

## 四、如果只能改三件事

按投入产出排序：

1. **凭证加密或改用外部引用** —— 安全收益最大，改动局部。
2. **给所有隐式行为加显式开关** —— 可运维性提升最明显。
3. **把"父分组"和"叶子"在文档里分开统计** —— 直接影响用户能不能找到东西。

---

## 检查清单

- [ ] 你是否在把端点名翻译成工具名，而不是直接照搬？
- [ ] 重试/限流/超时是否只有一份实现？
- [ ] 版本判断是否在规格里，而不是 if 分支里？
- [ ] stdout 是否只有数据？
- [ ] 是否避免了静默回退？
- [ ] 自动学到的知识是否需要人工确认？
- [ ] 版本比较是否按数字段？
- [ ] 错误是否带"下一步"？
- [ ] 必填参数是否压到 2 个以内？
- [ ] `--no-input` 是否真的不阻塞？
