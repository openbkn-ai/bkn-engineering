# Context Loader 命令参考

MCP JSON-RPC 协议的分层检索。

## KN 选择

运行时子命令接受 `<kn-id>` 作为**第一个位置参数**（与 `openbkn bkn …` 风格一致），MCP endpoint 自动从当前平台派生为 `<base-url>/api/agent-retrieval/v1/mcp`，无需任何持久化配置。也支持全局 `--kn-id <id>` / `-k <id>` flag。

```bash
openbkn context-loader tools <kn-id>
openbkn context-loader search-schema <kn-id> "Pod"
# 或者
openbkn context-loader tools --kn-id <kn-id>
```

## 配置（已废弃）

> **Deprecated**: `context-loader config` 子命令仍保留向后兼容，但每次调用打印 deprecation 警告，未来版本将移除。stateless 模式（`--token`）下整个 `config` 子命令组（`set` / `use` / `list` / `remove` / `show`）都直接被拒绝。
>
> 当运行时子命令省略 `<kn-id>` 且未提供 `--kn-id` flag时，会回退到此处保存的 `current` 条目（仅为兼容历史用法）。新代码请直接传 `<kn-id>`。

```bash
openbkn context-loader config set --kn-id kn-123 [--name myconfig]
openbkn context-loader config use myconfig
openbkn context-loader config list
openbkn context-loader config show
openbkn context-loader config remove myconfig
```

## MCP 内省

下面所有示例中的 `<kn-id>` 也可以省略以回退到 deprecated 的 saved config（见上节）。

```bash
openbkn context-loader tools <kn-id>                       # 可用工具列表
openbkn context-loader tool-call <kn-id> <name> --args '<json>'  # 直接调用任意 MCP tool
openbkn context-loader resources <kn-id>                   # 可用资源列表
openbkn context-loader resource <kn-id> <uri>              # 读取资源
openbkn context-loader templates <kn-id>                   # 资源模板
openbkn context-loader prompts <kn-id>                     # 可用 prompt
openbkn context-loader prompt <kn-id> <name> [--args '<json>']
```

## Layer 1 — Schema 搜索

推荐使用 `search-schema`，它调用 MCP `search_schema`，支持 `object_types`、`relation_types`、`action_types`、`metric_types`。

```bash
openbkn context-loader search-schema <kn-id> "Pod"
openbkn context-loader search-schema <kn-id> "利润率" --scope object,metric --max 10 --brief --no-rerank
openbkn context-loader search-schema <kn-id> "Pod" --format toon
```

参数映射：`--format` -> `response_format`，`--scope` -> `search_scope`，`--max` -> `max_concepts`，`--brief` -> `schema_brief: true`，`--no-rerank` -> `enable_rerank: false`。

兼容命令仍保留，但**全部走 Context Loader 公共 HTTP endpoint**（`/api/agent-retrieval/v1/kn/kn_search` 与 `/semantic-search`），不再触碰已被移除的 MCP `kn_search` / `kn_schema_search`：

```bash
openbkn context-loader kn-search <kn-id> "Pod" [--only-schema]
openbkn context-loader kn-schema-search <kn-id> "Pod" [--max 10]
```

> SDK 层同样走 HTTP：TS `client.bkn.knSearch(...)`、Python `client.query.kn_search(...)` / `client.query.kn_schema_search(...)`。`ContextLoaderResource` 不再暴露 `kn_search` / `kn_schema_search` 方法——MCP 入口请直接用 `searchSchema` / `callTool`。

## Layer 2 — 实例查询

```bash
# 条件查询
openbkn context-loader query-object-instance <kn-id> '{"ot_id": "ot-1", "condition": {"operation": "and", "sub_conditions": [{"field": "name", "operation": "==", "value_from": "const", "value": "web-pod"}]}, "limit": 5}'

# 子图查询
openbkn context-loader query-instance-subgraph <kn-id> '{"relation_type_paths": [{"start_ot_id": "ot-1", "paths": [{"rt_id": "rt-1", "direction": "positive"}]}]}'
```

## Layer 3 — 逻辑属性 & Action

```bash
# 获取逻辑属性
openbkn context-loader get-logic-properties <kn-id> '{"ot_id": "ot-1", "query": "status", "_instance_identities": [{"id": "123"}], "properties": ["status", "cpu"]}'

# 获取 Action 信息
openbkn context-loader get-action-info <kn-id> '{"at_id": "at-1", "_instance_identity": {"id": "123"}}'
```

### find-skills — 召回对象类下的 Skill

按对象类（可选缩小到具体实例）召回挂载的 Skill。对应 MCP tool `find_skills`，0.7.0 起可用。

```bash
# 仅按对象类召回（top_k 默认 10）
openbkn context-loader find-skills <kn-id> ot_drug

# 加自然语言查询和 top_k
openbkn context-loader find-skills <kn-id> ot_drug --query "treatment" --top-k 5

# 缩小到具体实例 + 切到 toon 输出
openbkn context-loader find-skills <kn-id> ot_drug \
  --instance-identities '[{"drug_id": "DRUG_001"}]' \
  --format toon
```

**CLI 参数**

| 参数 | 必填 | 说明 |
|------|:----:|------|
| `<object_type_id>` | ✅ | 位置参数，对象类 id |
| `--query / -q <text>` | | 自然语言查询，缩小召回范围 |
| `--top-k / -n <N>` | | 1..20，默认 10 |
| `--instance-identities / -i '<json-array>'` | | 实例身份数组（来自 Layer 2 `_instance_identity`） |
| `--format / -f json\|toon` | | 输出格式，默认 `json` |

**返回结构**

```json
{
  "entries": [
    { "skill_id": "sk_xxx", "name": "Skill 1", "description": "..." }
  ],
  "message": "..."
}
```

**SDK 等价**

```ts
// TypeScript
const result = await client.contextLoader.findSkills({
  object_type_id: "ot_drug",
  skill_query: "treatment",
  top_k: 5,
  // instance_identities: [{ drug_id: "DRUG_001" }],
  // response_format: "json",
});
```

```python
# Python
result = client.context_loader.find_skills(
    "ot_drug",
    skill_query="treatment",
    top_k=5,
    # instance_identities=[{"drug_id": "DRUG_001"}],
    # response_format="json",
)
```

**校验规则（client side）**

- `object_type_id` 必填，空字符串直接抛错。
- `top_k`（若提供）必须在 `[1, 20]`，否则抛错；不传时由服务端按默认 10 处理。
- `instance_identities`（若提供）必须是数组，每个元素是普通对象（复用 `validateInstanceIdentities`）。
- `response_format` 仅接受 `"json"` / `"toon"`。

## JSON 格式

### condition

```json
{
  "operation": "and",
  "sub_conditions": [
    {"field": "name", "operation": "==", "value_from": "const", "value": "Pod-1"},
    {"field": "status", "operation": "in", "value_from": "const", "value": ["Running", "Pending"]}
  ]
}
```

支持的 operation：`==`, `!=`, `>`, `<`, `>=`, `<=`, `in`, `not_in`, `like`, `not_like`，以及逻辑组合 `and` / `or`（配合 `sub_conditions`）。

> 完整的「属性类型 → 可用操作符」对照表、`exist`/`not_exist` 用法、SQL 视图与 OpenSearch 兼容性差异，见 [`bkn.md` 的 object-type query 条件过滤一节](bkn.md#object-type-query-条件过滤)。实际可用操作符以对象类 `data_properties` 中各属性的 `condition_operations` 字段为准。
>
> **常见错误**：
> - `match` / `contain` / `prefix` 等仅 OpenSearch 索引模式可用，SQL 视图数据源会返回 500；做文本模糊匹配优先 `like`。
> - `eq`、`gt`、`lt`、`gte`、`lte` 不是合法操作符，请用 `==`、`>`、`<`、`>=`、`<=`。
> - **string** 字段更常用 `like` / `in`；**keyword** 字段为不分词关键字，不要使用 `like`。
> - `like` / `not_like` 不支持通配符 `%` / `_`，`value` 直接写普通子串。
> - `exist` / `not_exist` 不需要 `value` 字段。
