# BKN 命令参考

知识网络管理：schema CRUD、构建、导出、推送/拉取。

与 CLI 一致：运行 `openbkn bkn`、`openbkn bkn validate --help`、`openbkn bkn push --help` 可查看与当前版本同步的用法（含 `validate`/`push` 的编码相关选项）。

## 概览

```bash
openbkn bkn list                     # 列出知识网络
openbkn bkn get <kn_id> [--stats] [--export]
```

## 知识网络

```bash
openbkn bkn list [--name <n>] [--name-pattern <p>] [--tag <t>] [--sort update_time] [--direction desc] [--offset 0] [--limit 50] [-v]
openbkn bkn get <kn_id> [--stats] [--export]
openbkn bkn stats <kn_id>
openbkn bkn export <kn_id>
openbkn bkn create [options]                         # 创建空知识网络（或 --body-file）
openbkn bkn create-from-ds <ds_id> --name <name> [--tables <t1,t2>] [--build/--no-build] [--timeout 300]  # 自动 dataview + 全列 mapped_field；与手动的 `object-type create` 不同，后者需自备 dataview；CLI 会为 create 补全 mapped_field / 拉取 dataview 字段，见 `references/build-kn-from-db.md`
openbkn bkn create-from-csv <ds_id> --files <glob> --name <name> [--table-prefix <p>] [--build/--no-build] [--timeout 300]
openbkn bkn update <kn_id> [--name <n>] [--description <d>] [--tag <t> ...]
openbkn bkn build <kn_id> [--wait/--no-wait] [--timeout 300]
openbkn bkn delete <kn_id> [--yes]
openbkn bkn validate <directory> [--detect-encoding/--no-detect-encoding] [--source-encoding <name>]
openbkn bkn push <directory> [--branch main] [--detect-encoding/--no-detect-encoding] [--source-encoding <name>]
# 规范上 .bkn 应为 UTF-8。默认会检测 .bkn 编码并规范为 UTF-8 再校验/打包；可用 --no-detect-encoding 要求严格 UTF-8；
# 已知为 GBK/GB18030 等时可 --source-encoding gb18030（整目录统一）。检测置信度不足时会报错并提示指定编码。
openbkn bkn pull <kn_id> [<directory>] [--branch main]  # 下载 BKN tar 并解压
```

## Object Type

```bash
openbkn bkn object-type list <kn_id>
openbkn bkn object-type get <kn_id> <ot_id>                # -v 显示完整 data_properties
openbkn bkn object-type create <kn_id> --name <n> --dataview-id <dv> --primary-key <pk> --display-key <dk> [--property '<json>' ...]  # --dataview-id 接受 mdl UUID 或 Vega resource ID；无 --property 时从 dataview 拉全字段并生成 mapped_field；有 --property 时可省略 mapped_field（CLI 补全）
# update：合并模式会先 GET 当前对象类，再合并参数后 PUT（与 Studio 行为一致）
openbkn bkn object-type update <kn_id> <ot_id> [--name <n>] [--display-key <dk>] [--add-property|--update-property '<json>' ...] [--remove-property <name> ...] [--tags '["标签"]'] [--comment <s>] [--icon <i>] [--color <c>] [--branch main]
# update：原始 JSON 模式（整段 PUT body，勿与上面合并参数同时使用）
openbkn bkn object-type update <kn_id> <ot_id> '<full-json-body>'
openbkn bkn object-type delete <kn_id> <ot_ids> [--yes/-y]
openbkn bkn object-type query <kn_id> <ot_id> ['<json>']   # 查询实例（支持 --limit/--search-after）
openbkn bkn object-type properties <kn_id> <ot_id> '<json>' # 查询实例属性
```

### object-type update 说明（数据属性：增 / 改 / 删）

| 操作 | CLI 方式 | 说明 |
|------|----------|------|
| **添加** | `--add-property '<json>'` | `name` 在现有列表中不存在则**追加**一条属性。 |
| **更新** | `--update-property '<json>'` 或 `--add-property '<json>'` | 与添加共用逻辑：`name` **已存在**则**整段替换**该属性（改 display_name、type、mapped_field 等）。`--update-property` 与 `--add-property` 等价，仅语义区分。 |
| **删除** | `--remove-property <name>` | 按 `name` 从 `data_properties` 中移除；可重复多次删多个。 |

- **`--tags`**：替换标签，值为 JSON 字符串数组，如 `'["足球","球员"]'`。
- **原始 JSON（整表替换）**：第三个参数传入完整对象类 JSON（或以 `openbkn call` PUT），适合从 Studio 复制整段 body；勿与上面合并参数混用。
- **`openbkn call -d`**：带 body 时会自动设置 **`Content-Type: application/json`**（未手动指定 `-H Content-Type` 时）。

## object-type query 条件过滤

`query` 的 JSON body 支持 `condition` 字段，用于服务端过滤，避免全量拉取后本地筛选。

### 语法

```jsonc
// 单条件（limit 不超过 30）
{"limit": 20, "condition": {"field": "quantity", "operation": ">", "value": 4000}}

// 多条件组合（AND）
{"limit": 20, "condition": {
  "operation": "and",
  "sub_conditions": [
    {"field": "quantity", "operation": ">", "value": 4000},
    {"field": "status", "operation": "==", "value": "active"}
  ]
}}

// OR 组合
{"limit": 20, "condition": {
  "operation": "or",
  "sub_conditions": [
    {"field": "category", "operation": "==", "value": "A"},
    {"field": "category", "operation": "==", "value": "B"}
  ]
}}
```

### 属性类型与可用操作符

写 `condition` 前先看 object type 的 `data_properties` 中各属性的 `type` 与 `condition_operations`。常见类型与推荐操作符：

| 属性类型 | 推荐操作符 |
|----------|------------|
| 数值 (short, integer, long, float, double, decimal) | `==`, `!=`, `>`, `>=`, `<`, `<=`, `in`, `not_in` |
| string（变长字符串） | `like`, `not_like`, `in`, `not_in`（避免用 `==` 做「包含」类需求；`==` 为精确等值，语义不直观时优先 `like` 或 `in`） |
| keyword（不分词关键字） | `==`, `!=` |
| text（分词文本） | `==`, `!=`, `match`（`match` 仅 OpenSearch 索引模式可用；SQL 视图见上表） |
| boolean | `==`, `!=`, `exist`, `not_exist` |
| timestamp / datetime | `==`, `!=`, `>`, `>=`, `<`, `<=` |

> 实际可用操作符以各属性的 **`condition_operations`** 为准；需要完整定义时可 `bkn object-type get <kn_id> <ot_id> -v` 查看 `data_properties`。

### 支持的 operation

| 类型 | operation | SQL 视图兼容 |
|------|-----------|:----------:|
| 比较 | `==`, `!=`, `>`, `>=`, `<`, `<=` | ✅ |
| 范围 | `in`, `not_in` | ✅ |
| 文本 | `like`, `not_like` | ✅ |
| 逻辑组合 | `and`, `or`（配合 `sub_conditions` 数组） | ✅ |
| 范围（索引） | `range`, `out_range` | ⚠️ 仅 OpenSearch |
| 文本（索引） | `contain`, `not_contain`, `prefix`, `regex` | ⚠️ 仅 OpenSearch |
| 全文搜索 | `match`, `match_phrase`, `multi_match` | ❌ SQL 视图报 500 |
| 空值/存在 | `exist`, `not_exist`, `null`, `not_null`, `empty`, `not_empty` | ⚠️ 仅 OpenSearch |

- **`exist` / `not_exist`**：不需要 `value` 字段。例：`{"field": "description", "operation": "exist"}`。
- **`like` / `not_like`**：不支持通配符 `%` / `_`；`value` 为普通子串，按「是否包含子串」匹配（大小写等取决于后端/数据源配置）。

> **重要**：大部分数据源（MySQL / PostgreSQL）通过 SQL 视图查询。上表"SQL 视图兼容"列为 ✅ 的操作符可安全使用；标 ⚠️ 或 ❌ 的仅在 OpenSearch 索引模式下可用，SQL 视图下会返回 400 或 500 错误。
>
> **常见错误**：
> - `eq`、`gt`、`lt`、`gte`、`lte` **不是合法操作符**，会返回 400 InvalidParameter。正确写法是 `==`、`>`、`<`、`>=`、`<=`。
> - **string** 字段更常用 `like` / `in` 做「子串包含 / 多值」筛选；若用 `==` 须明确是精确等值需求。
> - **keyword** 字段为精确关键字，**不要**使用 `like`（不适用或行为与预期不符）。

### 分页与大数据量查询

对于字段多或数据量大的对象类型，单次查询返回数据可能非常大。建议：

- **设置合理的 limit**：默认 limit=30，字段多的宽表建议 10~20
- **使用 `search_after` 游标分页**：首次查询返回 `search_after` 字段，传入下一次查询实现翻页

```bash
# 首次查询
openbkn bkn object-type query <kn-id> <ot-id> '{"limit":20}'
# 返回 {"datas": [...], "search_after": ["val1","val2","val3"]}

# 翻页：将上次返回的 search_after 传入
openbkn bkn object-type query <kn-id> <ot-id> '{"limit":20,"search_after":["val1","val2","val3"]}'
```

### 示例

```bash
# 数量大于 4000 的产品
openbkn bkn object-type query <kn-id> <ot-id> '{"limit":30,"condition":{"field":"quantity","operation":">","value":4000}}'

# 名称模糊匹配"手机"且价格 >= 1000
openbkn bkn object-type query <kn-id> <ot-id> '{"limit":30,"condition":{"operation":"and","sub_conditions":[{"field":"name","operation":"like","value":"手机"},{"field":"price","operation":">=","value":1000}]}}'

# 状态为 active 或 pending（用 in）
openbkn bkn object-type query <kn-id> <ot-id> '{"limit":30,"condition":{"field":"status","operation":"in","value":["active","pending"]}}'
```

> **注意**：不支持 `{"field": {">": value}}` 这种简写语法，必须使用 `{"field": ..., "operation": ..., "value": ...}` 结构。

## Object-type query strategy for LLM and Agent

**所有 `object-type query` 调用必须遵守以下规则，否则返回数据过大会导致 JSON 截断和解析失败：**

1. **limit 不要超过 30**。CLI 默认 limit=30。字段多的宽表建议 limit 10~20，按实际单条记录大小调整
2. **需要更多数据时使用 `search_after` 分页**，不要加大 limit：
   ```
   # 第一页
   openbkn bkn object-type query <kn> <ot> '{"limit":20}'
   # → 返回 search_after: ["v1","v2","v3"]
   # 第二页
   openbkn bkn object-type query <kn> <ot> '{"limit":20,"search_after":["v1","v2","v3"]}'
   ```
3. **尽量使用 `condition` 过滤**，缩小返回集：按编号、名称、状态等精确或模糊过滤，避免全表扫描
4. **优先使用 `==`、`like`、`in` 操作符**（SQL 视图兼容）。`match`/`contain` 仅 OpenSearch 索引支持，SQL 视图下会报错
5. **自动裁剪**：当查询结果超过 100KB 时，CLI 会自动裁剪 `datas` 数组并附加 `_truncated` 字段。如果返回中包含 `_truncated`，按其中的 `hint` 提示执行下一轮查询：
   ```json
   {"_truncated": {"returned": 176, "total_fetched": 200, "remaining": 24,
     "next_search_after": ["v1","v2","v3"],
     "hint": "Pass --search-after '[...]' --limit 176 to fetch the next page."}}
   ```
   直接使用 `next_search_after` 的值作为下一次查询的 `--search-after` 参数即可

## Relation Type

```bash
openbkn bkn relation-type list <kn_id>
openbkn bkn relation-type get <kn_id> <rt_id>
openbkn bkn relation-type create <kn_id> --name <n> --source <ot_id> --target <ot_id> [--mapping src:tgt ...]
openbkn bkn relation-type update <kn_id> <rt_id> [--name <n>]
openbkn bkn relation-type delete <kn_id> <rt_ids> [--yes/-y]
```

## Search

```bash
openbkn bkn search <kn_id> <query> [--max-concepts <n>] [--mode <mode>]   # 语义搜索
```

## Subgraph

```bash
openbkn bkn subgraph <kn_id> '<json>'   # 子图查询
```

CLI maps JSON body to ontology-query `query_type`: if `relation_type_paths` is present, append `query_type=relation_path`; if only `source_object_type_id` (default subgraph mode), omit `query_type`.

## Action Type / Log / Execution

```bash
openbkn bkn action-type list <kn_id>
openbkn bkn action-type query <kn_id> <at_id> '<json>'
openbkn bkn action-type inputs <kn_id> <at_id>             # 列出 value_from=input 的参数 + dynamic_params 模板
openbkn bkn action-type execute <kn_id> <at_id> '<envelope-json>'  # 形式 A：传完整信封
openbkn bkn action-type execute <kn_id> <at_id> \           # 形式 B：CLI 帮你拼信封（推荐）
  --dynamic-params '<json>' [--instance '<json>']... [--trigger-type manual]
openbkn bkn action-execution get <kn_id> <execution_id> [--wait/--no-wait] [--timeout 300]
openbkn bkn action-log list <kn_id> [--offset 0] [--limit 20] [--sort create_time] [--direction desc]
openbkn bkn action-log get <kn_id> <log_id>
openbkn bkn action-log cancel <kn_id> <log_id> [--yes/-y]
```

### `action-type execute` 请求体契约（重要）

后端只识别下面这种"信封"结构，**顶层散字段会被静默丢弃**——这是新手最常见的踩坑点：

```json
{
  "trigger_type": "manual",
  "_instance_identities": [{ "<primary_key>": "<value>" }],
  "dynamic_params": {
    "<param_name>": "<value>",
    "Authorization": "Bearer <token>"
  }
}
```

| 字段 | 必需性 | 说明 |
|---|---|---|
| `trigger_type` | 推荐 | 触发来源标识，常用值 `manual`；缺省时后端会用默认值，但显式写更利于排查 |
| `_instance_identities` | 看 ActionType 定义 | 数组，标识要操作的实例；"创建类"动作可传 `[]` |
| `dynamic_params` | **必填** | ActionType 中所有 `value_from: input` 的参数都放这里——**包括声明为 header 的 `Authorization`/`token`**（注意：这里的 `Authorization` 通常是**下游被调系统**的凭证，不一定等于你登录 OpenBKN 的会话 token，先看参数 `description`） |

#### `value_from` 参数来源（决定调用方要不要传）

ActionType 的每个 `parameter` 都带一个 `value_from` 字段，**只有 `input` 类型需要调用方在 `dynamic_params` 里传值**，其余两种你不用管：

| value_from | 含义 | 调用方要做什么 | 传了会怎样 |
|---|---|---|---|
| `input` | 值由调用方在执行时通过 body 传入（含 `source: header` 的参数，如 `Authorization`/`token`） | **必须**塞进 `dynamic_params`，缺一个下游就拿到 `null` | 正常生效 |
| `const` | 值在 ActionType 定义里写死，执行时直接用 snapshot 里的常量 | 不用管 | 传了也被丢弃，最终用定义里的常量 |
| `property` | 值从 `_instance_identities` 解析出的目标实例属性自动取（参数名对应 ObjectType 的 property 名） | 不用管，确保 `_instance_identities` 能解析出实例即可 | 传了也以实例真实属性值为准 |

> 还有一种 `param` 仅出现在 RiskFunction 体系，普通 `action-type execute` 用不到。

##### 实战速查：用 `action-type inputs` 一步查完

```bash
# 直接列出该 ActionType 的所有 value_from=input 参数（含 type/source/required/description）
# 并附带一份可直接复制的 dynamic_params 模板
openbkn bkn action-type inputs <kn_id> <at_id>

# 只要模板（适合脚本）：
openbkn bkn action-type inputs <kn_id> <at_id> --template

# 只要原始 JSON：
openbkn bkn action-type inputs <kn_id> <at_id> --json
```

> 老办法（也能用）：`openbkn bkn action-type query <kn_id> <at_id> '{}' | jq '.parameters[] | select(.value_from=="input")'`。

#### ⚠️ 三条必知规则

1. **顶层散字段会被吞**：`{"_instance_identities":[], "task_id": "x"}` 这种写法里 `task_id` **不会**到达下游，落库后 `dynamic_params` 为 `undefined`。表现是下游 tool 拿到一堆 `null`，可能直接 401（缺 Authorization）或 500（缺必填业务字段）。
2. **`Authorization` 是 ActionType 自定义的普通参数，不是平台会话 token**：当 ActionType 把它声明为 `value_from: input` + `source: header` 时，**它的语义由 ActionType 自己决定**，通常是要传给下游被调系统的凭证（ERP、外部 LLM、第三方 API 等）。先看参数 `description` 确认要谁的 token，**不要无脑把 `openbkn auth` 的输出粘进去**——SDK 也不会自动塞，避免误把平台 token 泄漏给不相干的下游服务。
3. **`value_from: const` 无法被 body 覆盖**：要让调用方传值生效，先在 ActionType 定义里把对应参数改为 `value_from: input`；否则 `dynamic_params` 里塞了也会被 snapshot 里的常量盖掉。

#### 推荐：用 flag 形式，让 CLI 帮你拼信封

不想手写整段 envelope JSON？直接传语义槽位，CLI 内部组装出与上面**完全一样**的 body 发出去：

```bash
openbkn bkn action-type execute supplychain_bkn_v4 create_monitoring_task \
  --dynamic-params '{
    "task_id":"demo-001","task_name":"demo task","task_status":"开始",
    "product_code":"P-001","product_name":"demo product",
    "demand_start":"2026-04-20","demand_end":"2026-04-21","demand_quantity":3,
    "production_start":"2026-04-22","production_end":"2026-04-25","production_quantity":3,
    "created_by":"you",
    "Authorization":"Bearer <下游系统的 token>"
  }' \
  --no-wait
# 需要操作已有实例时再加 --instance（可重复）：
#   --instance '{"task_id":"demo-001"}'
# trigger_type 缺省为 manual，需覆盖时：--trigger-type schedule
```

互斥规则：positional envelope JSON 和 `--dynamic-params` / `--instance` / `--trigger-type` 不能同时出现，CLI 会直接报错。

#### 完整示例（envelope 形式，生产环境验证通过）

```bash
TOK=<your bearer token>
openbkn bkn action-type execute supplychain_bkn_v4 create_monitoring_task \
  "$(printf '{
    "trigger_type": "manual",
    "_instance_identities": [],
    "dynamic_params": {
      "task_id": "demo-001",
      "task_name": "demo task",
      "task_status": "开始",
      "product_code": "P-001",
      "product_name": "demo product",
      "demand_start": "2026-04-20",
      "demand_end": "2026-04-21",
      "demand_quantity": 3,
      "production_start": "2026-04-22",
      "production_end": "2026-04-25",
      "production_quantity": 3,
      "created_by": "you",
      "Authorization": "Bearer %s"
    }
  }' "$TOK")" \
  --no-wait
# 拿到 execution_id 后用 action-log get 看结果
openbkn bkn action-log get <kn_id> <execution_id>
```

#### 排错速查

| 现象 | 大概率原因 |
|---|---|
| 下游 `401 token expired` / `Common.UnAuthorization` | `dynamic_params` 里没有 `Authorization`（或参数还在顶层散着） |
| 下游 500 `could not convert ... 'null'` | `dynamic_params` 缺业务必填字段（如 `product_code`） |
| `action-log get` 看到 `dynamic_params: undefined`、各 result 的 `parameters: {}` | body 是扁平形式，整体被后端丢弃 |
| 改了 body 仍然走旧值 | ActionType 定义里该参数是 `value_from: const`，需平台侧改为 `input` |

## Data Source 绑定类型

Object Type 的 `data_source` 支持两种 type：

| `data_source.type` | ID 来源 | 数据访问方式 | `bkn build` |
|---|---|---|---|
| `data_view` | `dataview list` 的 mdl UUID | 构建索引后查询 | 需要 |
| `resource` | `vega resource list` 的 Vega 资源 ID | 通过 Vega 实时查询 | 不需要 / 不支持 |

`object-type create --dataview-id` 的 `--dataview-id` 参数可接受任一类型的 ID，CLI 会根据当前配置自动确定 `data_source.type`。

## 端到端示例

```bash
# 接入数据源 → 创建 KN → 查询
openbkn ds connect mysql db.example.com 3306 erp --account root --password pass
openbkn bkn create-from-ds <ds_id> --name "erp-kn" --build
openbkn bkn object-type list <kn_id>
openbkn bkn search <kn_id> "订单"
```
