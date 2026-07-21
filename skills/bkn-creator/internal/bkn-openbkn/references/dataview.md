# 数据视图命令参考（dataview）

平台 **mdl-data-model** 服务暴露的原子视图 / 自定义视图（HTTP：`/api/mdl-data-model/v1/data-views`）。从数据源建表映射到对象类时，通常先对应一张原子数据视图。

## 命令

```bash
openbkn dataview list [--datasource-id <id>] [--type <atomic|custom>] [--limit <n>] [-bd value] [--pretty]
openbkn dataview find --name <name> [--exact] [--datasource-id <id>] [--wait] [--no-wait] [--timeout <ms>] [-bd value] [--pretty]
openbkn dataview get <id> [-bd value] [--pretty]
openbkn dataview query <id> [--sql <sql>] [--limit <n>] [--offset <n>] [--need-total] [--raw-sql] [-bd value] [--pretty]
openbkn dataview delete <id> [-y] [-bd value]
```

（Python 安装的 CLI 上，`query` 的 `--sql` 可简写为 `-s`。）

**`query`**：对指定数据视图执行 SQL 查询，走平台 **mdl-uniquery** `POST /api/mdl-uniquery/v1/data-views/:id`。省略 `--sql` 时使用视图自身保存的 SQL。CLI 默认仅允许 **SELECT** / **WITH**（CTE）；`--raw-sql` 跳过该检查（慎用）。

### 参数说明

| 选项 | 含义 |
|------|------|
| `--raw-sql` | 仅 **`query`**：允许非 SELECT 语句（跳过客户端校验） |
| `--datasource-id` | 按数据源 ID 过滤（`data_source_id`） |
| `--type` | 视图类型，如 `atomic`、`custom`（按平台实际枚举） |
| `--limit` | 返回条数上限；**`-1` 表示不限制**（与省略 `--limit` 时 CLI 默认行为一致） |
| `--name` | 仅用于 **`find`**：作为服务端 `keyword`（模糊）；加 `--exact` 时在客户端再按名称 **完全一致** 过滤 |
| `--exact` | 仅 **`find`**：在 keyword 结果上再做精确匹配 |
| `--wait` / `--no-wait` | 仅 **`find`**：是否轮询直到出现或超时（与 `bkn create-from-ds` 内「先找已有原子视图」一致） |
| `--timeout` | 仅 **`find`**：与 `--wait` 配合的总等待时间（毫秒） |
| `-bd` / `--biz-domain` | 业务域；默认来自 `openbkn config show`（可用 `openbkn config list-bd` 查看平台可选域） |

### `list` 与 `find`

| 子命令 | 作用 |
|--------|------|
| **`list`** | 仅列出视图，可按数据源 / 类型 / 条数筛选；**不支持按名称关键字搜索** |
| **`find`** | 按名称查找：默认 **模糊**（服务端 keyword）；`--exact` 为 **精确**（keyword + 客户端 `name ===`）；`--wait` 用于等待视图就绪 |

后端仅提供 `keyword` 模糊参数；**精确匹配** 由客户端在 keyword 结果上过滤实现。

## 端到端示例

```bash
# 列出某一数据源下的视图（无关键字）
openbkn dataview list --datasource-id <ds-uuid> --pretty

# 按名称模糊搜索
openbkn dataview find --name BOM --pretty

# 精确名称 + 数据源 + 不等待（替代旧 find-by-table --no-wait）
openbkn dataview find --name 产品信息 --exact --datasource-id <ds-uuid> --no-wait --pretty

# 精确名称 + 轮询等待（与 create-from-ds 内部行为一致）
openbkn dataview find --name orders --exact --datasource-id <ds-uuid> --wait --pretty

# 获取视图详情
openbkn dataview get <view-id> --pretty

# 对视图执行 SQL（或省略 --sql 使用视图默认 SQL）
openbkn dataview query <view-id> --sql "SELECT * FROM my_table LIMIT 10" --pretty
```

## 与 BKN 的关系

数据视图通过 Object Type 的 `Data Source` 段绑定到知识网络。后端支持两种绑定方式：

| Data Source Type | ID 来源 | 说明 |
|---|---|---|
| `data_view` | `dataview list` 返回的 mdl UUID | 传统 mdl-data-model 路径 |
| `resource` | `vega resource list` 返回的 Vega 资源 ID | Vega 路径（`create-from-ds` 内部使用） |

两种类型在后端均能成功创建对象类。`dataview` 命令和 `vega resource` 命令实际操作不同的后端接口，返回不同格式的 ID，但都可用于绑定。

### 知识网络 → SQL JOIN（高级用法）

1. `openbkn bkn relation-type list <kn-id>`：看每条关系的 `mapping_rules`（源/目标属性名即 JOIN 键）。
2. `openbkn dataview get <view-id>`：取参与表的 `meta_table_name`（`catalog."schema"."table"`）；多表须**同一数据源**。
3. `openbkn dataview query <任一视图id> --sql '... JOIN ... ON <键> ...'`：表名用 `meta_table_name`，键与 KN 一致即可。

若 JOIN 无行，多为样例外键与模型不一致（例如订单头/行 ID 前缀混用）；可换 KN 中已对齐数据的关系（如 BOM 头/行按 `bomId`）验证。

## 排障

- 结果为空：检查 `openbkn config show` 的 business domain 是否与平台一致；必要时 `openbkn config set-bd <uuid>`。
- `list` 结果过多：加 `--datasource-id` 或 `--limit`；按名称搜改用 `find`。
