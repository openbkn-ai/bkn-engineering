# 从数据库构建知识网络

从已有数据库表构建知识网络并验证数据可查询的完整流程。

## 快速路径（推荐）

`create-from-ds` 一条命令完成「连接数据源 → 发现表 → 创建 OT → 构建索引」：

```bash
# 1. 连接数据源
openbkn ds connect mysql db.example.com 3306 erp --account root --password pass123
# → 返回 ds_id

# 2. 一键创建 KN（自动发现表、创建 OT、构建）
openbkn bkn create-from-ds <ds_id> --name "erp-kn" --build
# 指定表：--tables "orders,products,customers"
# 跳过构建：--no-build
# 超时：--timeout 600（默认 300s）

# 3. 验证
openbkn bkn object-type list <kn_id>
openbkn bkn object-type query <kn_id> <ot_id> '{"limit":5}'
openbkn bkn search <kn_id> "订单"
```

## 快速路径 vs 分步路径（能力差异）

| 能力 | `create-from-ds` | 手动 `object-type create` |
|------|------------------|----------------------------|
| 自动创建 dataview | 是 | 否（需先有 dataview） |
| 从 schema 生成全部属性 + `mapped_field` | 是 | 否；CLI 会按 dataview 拉取字段并补全 `mapped_field`（见下） |
| 自动 `bkn build` | 可选 `--build` | 需单独 `bkn build` |
| 自定义 PK/DK/属性名 | 有限（启发式） | 完全可控 |
| Data Source 类型 | `resource`（Vega 资源 ID） | 取决于 CLI 版本（`data_view` 或 `resource`） |

**推荐**：优先用快速路径；仅在需要自定义 PK/DK、筛选表或属性名时再走分步路径。

### Data Source 绑定类型说明

后端支持两种 `data_source.type`，均可成功创建对象类，但数据访问方式不同：

- **`resource`**：绑定 Vega 资源 ID（`vega resource list` 返回），`create-from-ds` 使用此类型。数据通过 Vega 实时查询，**不需要也不支持 `bkn build`**
- **`data_view`**：绑定 mdl 数据视图 UUID（`dataview list` 返回），传统路径。**需要 `bkn build` 构建索引**后才可查询

分步路径中 `--dataview-id` 可接受任一类型的 ID。如需获取 Vega 资源的列信息，可通过 `vega resource query <id> -d '{"limit":1}'` 查询一条数据推断字段。

## 分步路径

当需要精细控制（自定义 PK/display key、选择性建表等）时，手动逐步操作：

```bash
# 1. 连接数据源
openbkn ds connect mysql db.example.com 3306 erp --account root --password pass123

# 2. 发现表
openbkn ds tables <ds_id>

# 3. 创建空知识网络
openbkn bkn create --name "erp-kn"

# 4. 逐个创建对象类型（指定 PK 和展示字段）
openbkn bkn object-type create <kn_id> \
  --name "物料" --dataview-id <dv_id> \
  --primary-key material_code --display-key material_name

openbkn bkn object-type create <kn_id> \
  --name "库存" --dataview-id <dv_id> \
  --primary-key material_code --display-key material_name

# 说明（mapped_field）：
# - 不传 --property 时，CLI 会 GET 该 dataview 的字段列表，为每个字段生成 data_properties 与同名 mapped_field（与 create-from-ds 一致）。
# - 传 --property 时，可省略 mapped_field；CLI 会按属性 name/type/display_name 自动补全 mapped_field。
# - 构建引擎需要 PK 等列在 data_properties 中有映射；仅用 PK+DK 两列且未拉取 schema 时会导致 build 失败。

# 5. 构建索引（等待完成）
openbkn bkn build <kn_id> --wait --timeout 300

# 6. 验证数据已索引
openbkn bkn object-type list <kn_id>
openbkn bkn object-type query <kn_id> <ot_id> '{"limit":5}'
```

## 从 CSV 文件构建

当数据在本地 CSV 文件中，需要一个 OpenBKN 可访问的数据源作为中间存储。

### 前置：确定数据源

用户通常不知道 datasource_id。按以下顺序引导：

1. **先查已有数据源**：`openbkn ds list`，从返回的列表中选一个合适的（看 `type`、`name`、`database_name`）
2. **如果没有合适的**：帮用户连接一个新的 `openbkn ds connect mysql <host> <port> <db> --account <user> --password <pass>`
3. **向用户确认**：展示选中的数据源名称和数据库，确认后再执行导入

> 不要直接问用户要 datasource_id，而是帮他们查找和选择。

### 快速路径（一条命令）

```bash
openbkn bkn create-from-csv <datasource_id> --files "*.csv" --name "my-kn"

# 指定表前缀和部分表
openbkn bkn create-from-csv <ds_id> --files "物料.csv,库存.csv" \
  --name "supply-kn" --table-prefix sc_
```

### 分步路径

```bash
# 1. 导入 CSV 到数据源
openbkn ds import-csv <datasource_id> --files "*.csv" --table-prefix my_

# 2. 从数据源创建 KN
openbkn bkn create-from-ds <datasource_id> --name "my-kn" --build
```

### import-csv 参数

| 参数 | 必填 | 默认值 | 说明 |
|------|------|--------|------|
| `datasource_id` | 是 | — | OpenBKN 可访问的数据源 ID |
| `--files` | 是 | — | CSV 文件路径，逗号分隔或 glob |
| `--table-prefix` | 否 | `""` | 表名前缀 |
| `--batch-size` | 否 | 500 | 每批写入行数（1-10000） |
| `--recreate` | 否 | off | 首批发 overwrite 以覆盖重建表（列结构变更后重导同名表） |

## PK / Display Key 选择原则

| 原则 | 说明 |
|------|------|
| PK 必须唯一 | 选 `xxx_code`、`xxx_id` 等唯一标识列 |
| 避免浮点数做 PK | 浮点精度问题会导致查询失败 |
| Display key 选人可读字段 | 如 `material_name`、`product_name` |
| 无明显 PK 时 | 取第一个含 `code` 或 `id` 的列 |

常见表的推荐映射：

| 表名 | PK | Display Key |
|------|-----|-------------|
| 物料 | `material_code` | `material_name` |
| 库存 | `material_code` | `material_name` |
| 供应商 | `supplier_code` | `provided_material_name` |
| 产品 | `product_code` | `product_name` |
| 销售订单 | `id` | `product_name` |
| 员工 | `userid` | `username` |
| 部门 | `dept_code` | `name` |

## 构建后验证

构建完成后，用三种方式验证数据可用：

### REST 实例查询

```bash
openbkn bkn object-type query <kn_id> <ot_id> '{"limit":5}'
# 确认返回 datas 非空，且包含 _instance_identity
```

### 语义搜索

```bash
openbkn bkn search <kn_id> "物料"
# 确认 concepts 列表非空
```

### Context Loader（MCP 分层检索）

```bash
# 配置 KN 上下文
openbkn context-loader config set --kn-id <kn_id>

# Schema 搜索（推荐 MCP search_schema）
openbkn context-loader search-schema "物料" --scope object,relation,action

# 实例查询
openbkn context-loader query-object-instance '{"ot_id":"<ot_id>","condition":{"field":"material_code","operation":"==","value_from":"const","value":"101-000025"},"limit":1}'
```

## 绑定 Agent 进行对话

构建好的 KN 可以绑定到 Agent，实现自然语言问答：

```bash
# 创建 Agent 并绑定 KN
openbkn agent create --name "数据分析助手" --kn-id <kn_id>

# 发布
openbkn agent publish <agent_id>

# 对话
openbkn agent chat <agent_id> -m "最贵的物料是什么？"
openbkn agent chat <agent_id> -m "库存为零的物料有哪些？"
```

## 清理

```bash
openbkn agent delete <agent_id> --yes
openbkn bkn delete <kn_id> --yes
openbkn ds delete <ds_id> --yes
```

## 常见问题

| 问题 | 原因 | 解决 |
|------|------|------|
| 构建超时 | 数据量大 / 服务负载高 | `--timeout 600` 增加超时 |
| 查询返回空 | 构建未完成 / OT 未关联数据 | `openbkn bkn stats <kn_id>` 检查 doc_count |
| `match` 操作报 500 | SQL 视图不支持全文检索 | 改用 `like` 操作符 |
| PK 重复错误 | 选了非唯一列做 PK | 换用 `xxx_code`/`xxx_id` 列 |

更多排障条目见 [`troubleshooting.md`](troubleshooting.md)。
