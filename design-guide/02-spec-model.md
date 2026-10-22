# 02 - 端点规格模型

> 这一章是整套设计的地基。看懂了它，后面所有分层都是在消费这份规格。

---

## 1. 原子单位是什么

**不是命令，是"端点 + 动作"**。

一个原子包含四样东西：

```
标识      dashboards.create
方法      POST
路径      /api/v2/dashboards
参数      path / query / body
```

实测证据：`agent-context` 输出的每个叶子节点都带：

```json
{
  "annotations": {
    "pp:endpoint": "dashboards.create",
    "pp:method": "POST",
    "pp:path": "/api/v2/dashboards"
  }
}
```

---

## 2. 规格的完整形态（建议）

如果从零实现，建议一份端点的 YAML 长这样：

```yaml
endpoint: dashboards.create
group: dashboards
action: create
method: POST
path: /api/v2/dashboards
summary: Create a new Wyn Dashboard (.dbd)
description: |
  Accepts a .dbd JSON document from a file or stdin.

params:
  # 位置参数：从 path 里抽取的占位符
  path:
    - name: id
      type: string
      required: true
      description: Dashboard ID
  # 查询参数
  query:
    - name: name
      type: string
      required: false
  # 请求体
  body:
    mode: json          # json | multipart | form | raw | file
    required: true
    schema_ref: "#/components/schemas/Dashboard"

responses:
  success: 201
  body_schema_ref: "#/components/schemas/Dashboard"

annotations:
  mcp:read-only: false
  pp:requires-input: true
  pp:happy-args:
    name: "Sales Overview"
  pp:typed-exit-codes: "0,2"

compat:
  min_version: "9.1"
  gated_in: ["10.0"]    # 仅 10.0 可用，9.1 隐藏
```

---

## 3. 标识命名规则

这是最容易做乱的地方。实测系统里有两套命名：

| 场合 | 规则 | 例子 |
|---|---|---|
| CLI 命令 | kebab-case，空格分隔层级 | `wyn-cli datasets native query-data` |
| 端点 ID | `资源.动作`，点号 | `datasets.native_query_data` |
| MCP 工具 | 下划线拼接 | `datasets_native_query_data` |
| Go 包名 | 下划线拼接 | `datasets_native_query_data` |

**规范**：

- **必须**保证四个名字之间可以**机械转换**，不能靠人记。
- **应该**用"资源 + 动作"两段式，不要三段以上。
- **禁止**在动作里混入版本号（用 `compat` 字段表达）。

转换示意：

```go
// 端点 ID → CLI 路径
func endpointToCLIPath(id string) []string {
    parts := strings.SplitN(id, ".", 2)   // ["datasets", "native_query_data"]
    resource := strings.Split(parts[0], "_")
    action := strings.ReplaceAll(parts[1], "_", "-")
    return append(resource, action)
}

// 端点 ID → MCP 工具名
func endpointToTool(id string) string {
    return strings.ReplaceAll(id, ".", "_")
}
```

---

## 4. 参数的三层分层

实测设计中，参数分成三层，顺序固定：

```
全局参数（所有命令共用）
  --instance --server-version --json --timeout --dry-run ...

资源参数（同一分组共用）
  --resource-type --venue ...

动作参数（这个端点独有）
  --path --name --stdin
```

**为什么重要**：MCP 工具的 `inputSchema` 就是这三层拼接的结果。实测 118 个工具里：

- 55 个工具 **零必填参数**
- 62 个工具 **只有 1 个必填参数**
- 只有 1 个工具需要 2 个必填参数

**规范**：

- **必须**把通用参数做成全局 flag，不要每个命令重复定义。
- **应该**让必填参数尽可能少。一个工具超过 3 个必填参数，AI 的调用成功率会明显下降。
- **应该**优先用默认值 + 可选参数，而不是强制必填。
- **可以**提供"从 stdin 读"或"从文件读"的替代入口，避免让 AI 拼大段 JSON。

---

## 5. 注解分类

实测出现的 11 种注解，可以分成四类：

### A. 绑定类（生成器必需）

| 注解 | 值 | 用途 |
|---|---|---|
| `pp:endpoint` | `dashboards.create` | 端点标识 |
| `pp:method` | `POST` | HTTP 方法 |
| `pp:path` | `/api/v2/...` | 路径模板 |

### B. 结构类（决定命令树形状）

| 注解 | 数量 | 含义 |
|---|---:|---|
| `pp:parent-group` | 60 | 这是分组节点，本身不可执行 |
| `pp:api-resource` | 19 | 顶层资源入口 |
| `mcp:hidden` | 2 | 不暴露给 MCP |

### C. 行为类（决定运行时）

| 注解 | 数量 | 含义 |
|---|---:|---|
| `mcp:read-only` | 293 | 不改变服务器状态 |
| `mcp:local-write` | 7 | 只写本地库 |
| `pp:requires-input` | 51 | 必须提供输入 |
| `pp:typed-exit-codes` | 67 | 区分失败类型 |
| `pp:no-error-path-probe` | 1 | 跳过错误路径探测 |

### D. 辅助类（给人/AI 用）

| 注解 | 数量 | 含义 |
|---|---:|---|
| `pp:happy-args` | 43 | 一组推荐的可用参数 |
| `pp:deprecated` | 2 | 已废弃 |

**规范**：

- **必须**把"只读/写"变成注解，不要靠人工维护文档。
- **应该**给每个常用命令配 `happy-args`，这是 AI 最省事的参考。
- **应该**用 `hidden` 隐藏内部调试命令，而不是删掉。

---

## 6. "只读"判断的坑

实测 `mcp:read-only: "true"` 出现在 293 个节点上，其中包含**父分组节点**。

所以不能简单地说"293 个只读操作"。真正要看的是**叶子节点**。

**规范**：

- **必须**在叶子节点上判定只读性，父节点继承或忽略。
- **应该**对 POST 但实际只读的接口（比如"查询"类 POST）显式标注，否则 AI 会因为看到 POST 而不敢调。
- **禁止**把 DELETE 标成只读。

统计参考（实测）：

```
GET    130   → 通常只读
POST   117   → 混合：创建 / 查询 / 动作
PUT     48   → 修改
DELETE  26   → 删除
```

---

## 7. 版本门控是规格字段，不是代码分支

实测兼容描述：

> Compatible with Wyn Enterprise 9.1 (312/319 endpoints available; 7 Wyn 10+ endpoints gated)

**规范**：

- **必须**把版本要求写在规格里（`min_version` / `gated_in`），不要在命令代码里写 `if version >= "10.0"`。
- **应该**在生成阶段就过滤掉不支持的端点，让命令**不存在**，而不是存在但报错。
- **必须**给用户一个参数手动覆盖探测结果：`--server-version 9.1|10.0|auto`。

理由：命令不存在，AI 就不会尝试；命令存在但报错，AI 会反复重试。

---

## 8. 从规格生成什么

一份规格应该至少能生成五样东西：

| 生成物 | 消费者 |
|---|---|
| cobra 命令定义（含 flags、help） | 人/脚本 |
| MCP 工具定义（name/description/inputSchema） | AI |
| 参数校验逻辑（必填、类型、枚举） | 运行时 |
| API 文档 / 帮助文本 | 人 |
| `agent-context` 的命令树 JSON | AI 首次接入 |

**规范**：

- **必须**保证五者来自同一份数据，不允许任何一份手工维护。
- **应该**在 CI 里做一致性检查：规格变了但生成物没更新就报错。
- **可以**把生成物提交进仓库，方便 code review 时看到变化。

---

## 9. 规格的校验规则

建议在生成前做这些检查（都是这套系统实际踩过的点）：

```
[ ] endpoint 唯一
[ ] method 合法（GET/POST/PUT/PATCH/DELETE）
[ ] path 里的 {placeholder} 都在 params.path 里有定义
[ ] 每个必填参数有描述
[ ] read-only 与 method 不矛盾（DELETE 不得为 read-only）
[ ] 有 happy-args 的命令，其参数确实存在
[ ] deprecated 的命令仍然可解析（不能直接删）
[ ] MCP 工具名不与已有工具冲突
[ ] 工具名长度合理（建议 <= 48 字符）
```

---

## 10. 一个最小可用的规格示例

如果只想先做 10 个命令验证可行性：

```yaml
endpoints:
  - endpoint: dashboards.list
    method: GET
    path: /api/v2/dashboards
    summary: List or search Dashboards
    params:
      query:
        - {name: keyword, type: string}
        - {name: page, type: int, default: 1}
    annotations:
      mcp:read-only: true
      pp:happy-args: {keyword: "sales"}

  - endpoint: dashboards.delete
    method: DELETE
    path: /api/v2/dashboards/{id}
    summary: Delete a Dashboard by ID
    params:
      path:
        - {name: id, type: string, required: true}
    annotations:
      mcp:read-only: false
```

生成结果应该是：

```
wyn-cli dashboards list --keyword sales
wyn-cli dashboards delete <id>
MCP: dashboards_list / dashboards_delete
```

---

## 检查清单

- [ ] 新增接口只改规格，不改执行代码？
- [ ] 四个名字（端点/CLI/MCP/Go 包）能机械转换？
- [ ] 必填参数是否控制在 2 个以内？
- [ ] 只读注解是否只标在叶子上？
- [ ] 版本限制是规格字段而不是 if 分支？
- [ ] 五类生成物是否同源？
