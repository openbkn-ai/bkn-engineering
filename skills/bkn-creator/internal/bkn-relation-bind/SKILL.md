---
name: bkn-relation-bind
description: 基于对象绑定结果判定关系类型，绑定中间视图（data_view 类型）。
---

# 关系绑定

公约：`../_shared/contract.md`

## 做什么

在对象类绑定完成后，分析已绑定对象的视图 schema，判定每条关系的类型（direct 或 data_view），并为 data_view 类型推荐/绑定中间视图。

## 触发时机

在 `bkn-bind` 之后、`bkn-map` 之前执行。此时对象类已绑定到具体数据视图，可以分析视图间的关联关系。

## 输入

- `binding_decision_list`：`bkn-bind` 的输出（bound 对象列表）
- `view_schema_map`：各对象视图的字段 schema + 外键信息
- `relation_list`：关系清单（来自 `bkn-extract`）
- `dataview_availability`：可选，平台可用视图列表（用于中间视图推荐）

## 关系类型判定规则

### direct 类型

满足以下条件之一：

| 条件 | 说明 |
|------|------|
| 起点视图有字段直接引用终点视图 | 外键关系：`orders.product_code → products.product_code` |
| 起点属性与终点属性同名且类型兼容 | 如 `supplier_code` 在两端都存在 |
| 单数据源内的主键-外键关系 | 同一数据库内可直接 JOIN |

### data_view 类型

满足以下条件之一：

| 条件 | 说明 |
|------|------|
| 起点视图无字段直接引用终点视图 | 需中间表做桥梁 |
| 跨数据源 | 起点对象和终点对象来自不同数据源 |
| 多对多关系的中间表 | 如"订单-产品"通过"订单明细"关联 |
| 业务描述提及中间实体 | 文本中有"订单明细"、"关联表"等线索 |

## 流程

### 1. 遍历关系清单

对每条关系执行：

```
1. 获取起点对象的 view_schema
2. 获取终点对象的 view_schema
3. 检查起点视图是否有字段直接引用终点视图（外键检查）
4. 检查起点属性名是否与终点属性名匹配（同名检查）
5. 判定类型：
   - 有直接关联 → direct
   - 无直接关联 → data_view
```

### 2. direct 类型处理

```
1. 提取 Mapping Rules：
   - 从外键信息提取 Source Property → Target Property
   - 或从同名属性提取映射
2. 输出到 direct_relations 列表
```

### 3. data_view 类型处理

```
1. 中间视图推荐：
   - 扫描 dataview_availability，找同时引用起点和终点视图的候选
   - 或根据业务名称线索匹配（如"订单明细"、"关联表"）
   - 输出候选列表（最多 5 个）
   
2. 用户确认：
   - 有候选 → 展示候选列表，用户选择或跳过
   - 无候选 → 标记 pending，提示后续补绑
   
3. 确认后提取映射：
   - Source Mapping: 起点属性 → 中间视图字段
   - Target Mapping: 中间视图字段 → 终点属性
```

### 4. pending 关系处理

```
- 不阻断流程
- 输出到 pending_relations 列表
- 在 pipeline_state.yaml 记录
- 后续可通过 update pipeline 补绑
```

## 中间视图推荐策略

### 策略 1：外键链路追踪

```
起点视图 → 找引用它的视图 → 这些视图是否也引用终点视图？
           ↓
        中间候选
```

### 策略 2：名称线索匹配

```
业务文本中提到的中间实体：
  "订单包含多个产品" → 搜索名称含"订单明细"、"order_item"的视图
  
匹配规则：
  - 起点对象名 + 终点对象名 → "订单产品"
  - 起点对象名 + "明细" → "订单明细"
  - 起点/终点的别名组合
```

### 策略 3：数据源关联表

```
起点视图所属数据源 → 扫描该数据源的关联表 → 找同时含起点主键和终点主键的表
```

## 输出

```yaml
relation_binding_result:
  direct_relations:
    - 关系ID: ""
      关系名称: ""
      源对象: ""
      目标对象: ""
      mapping_rules:
        - source_property: ""
          target_property: ""
      判定依据: ""  # 如"起点视图 product_code 外键引用终点视图"
      confidence: high | medium | low
      
  data_view_relations:
    - 关系ID: ""
      关系名称: ""
      源对象: ""
      目标对象: ""
      intermediate_view_id: ""    # 确认后填入
      intermediate_view_name: ""  # 确认后填入
      intermediate_view_candidates:  # 推荐候选
        - view_id: ""
          view_name: ""
          match_reason: ""
          confidence: high | medium | low
      status: confirmed | pending | rejected
      source_mapping_rules:       # 确认后填入
        - source_property: ""
          view_property: ""
      target_mapping_rules:       # 确认后填入
        - view_property: ""
          target_property: ""
      判定依据: ""  # 如"跨数据源，需中间视图"
      
  pending_relations:
    - 关系ID: ""
      关系名称: ""
      原因: ""  # 如"无匹配中间视图候选"
      建议: ""  # 如"请手动指定中间视图或后续补绑"
      
relation_binding_summary:
  total_relations: 0
  direct_count: 0
  data_view_confirmed_count: 0
  data_view_pending_count: 0
  pending_count: 0
```

## 用户确认阶段

当存在 data_view 类型关系需要确认时，pipeline 会暂停并展示：

```
检测到 {n} 条关系需要中间视图绑定：

【{关系名称}】
  起点对象: {源对象} → 已绑定视图: {起点视图名}
  终点对象: {目标对象} → 已绑定视图: {终点视图名}
  判定依据: {判定依据}
  
  推荐中间视图：
    1. {候选1名} ({候选1ID}) - 匹配原因: {原因1}
    2. {候选2名} ({候选2ID}) - 匹配原因: {原因2}
    ...
  
请选择：
  A. 选择推荐候选（输入序号 1-5）
  B. 手动指定中间视图（输入 view_id 或 view_name）
  C. 暂时跳过，标记为 pending（后续可补绑）
  D. 改为 direct 类型（起点属性直接关联终点属性）
```

## 与后续 Skill 的衔接

| Skill | 消费内容 |
|-------|----------|
| `bkn-map` | `direct_relations.mapping_rules` + `data_view_relations.source/target_mapping_rules`（仅 confirmed） |
| `bkn-draft` | 关系类型 + Mapping 结构（pending 关系生成占位符版本） |
| `bkn-backfill` | 关系类 Mapping View + Source/Target Mapping 回填 |

## 约束

- 仅处理 `bkn-bind` 输出的 `bound` 对象涉及的关系
- 起点或终点对象为 `pending` 或 `rejected` 时，对应关系也标记为 `pending`
- `pending` 关系不阻断流程，可后续补绑
- 推荐候选最多 5 个，置信度排序
- 不编造映射规则，必须有外键或用户确认依据

## 跳过条件

以下情况跳过本 skill：

- `bind_mode == deferred`（无可用数据视图）
- `relation_list` 为空
- 所有起点/终点对象都是 `pending` 或 `rejected`