---
name: bkn-bind
description: 为 BKN 对象匹配 Vega resource，输出绑定决议。
---

# 资源绑定

公约：`../_shared/contract.md` | 绑定规则：`references/binding-rules.md` | 语义服务：`references/semantic-guide.md`

## 做什么

给定对象草案和候选资源，为每个对象找到最佳绑定资源，输出 bound / pending / rejected。

## 输入

- `object_draft_list`：对象清单（含属性和 `存储位置` 标记）
- `candidate_resources`：候选资源（空则使用 `bkn-env` 输出的 `resource_availability.available_resources` 进行匹配）
- `resource_availability`：可选，`bkn-env` 输出的完整信封
- `network_context`：网络名、领域

## 流程

1. **存储位置过滤**：
   - 跳过 `存储位置: local` 的对象（本地对象无需资源绑定）
   - 在输出中记录被跳过的 local 对象列表
   - 仅对 `platform` 对象执行后续步骤
2. 从对象描述提取数据源线索
3. 委托 `bkn-openbkn` 查资源存在性 + property schema（`openbkn vega resource query`）
4. **property schema 输出**：将每个已绑定对象的资源属性写入 `resource_schema_map`，供 `bkn-map` 做属性回灌和 `bkn-draft` 做属性命名预对齐
5. 字段兼容性验证：逐属性比对，`not_found > 50%` 降级为 ambiguous
6. 补充匹配：语义服务（`references/semantic-api.md`）+ GKN 复用
7. 逐对象决议：bound / pending / rejected
8. 数据源一致性校验：不同 datasource_id 标记风险

判定规则见 `references/binding-rules.md`

## 输出

```yaml
binding_decision_list:
  bound: [{object_id, object_name, selected_resource_id, confidence, reason}]
  pending: [{object_id, object_name, candidates, blocking_points}]
  rejected: [{object_id, object_name, reason}]
  skipped_local_objects: [对象名]
binding_summary:
  total_objects: 0       # 仅计算 platform 对象
  bound_objects: 0
  binding_rate: 0.0      # bound_objects / total_objects（不含 local 对象）
resource_schema_map:
  {object_name}: 
    resource_id: ""
    resource_name: ""
    datasource_id: ""           # 数据源 ID（用于跨数据源判断）
    properties: [{name, type, description}]
    foreign_keys:               # 外键信息（用于关系类型判定）
      - field: ""               # 本视图中的外键字段名
        references_view_id: ""  # 引用的视图 ID（如有）
        references_table: ""    # 引用的物理表名（如有）
        references_field: ""    # 引用的字段名
```

## 资源匹配职责

`bkn-bind` 是对象-资源匹配的唯一决策点。`bkn-env` 仅提供可用资源列表（`resource_availability`），不做匹配推荐。

**候选资源来源**：
1. `candidate_views` 非空 → 使用用户传入的候选列表（`dataview_availability` 仅用于截断风险检查）
2. `candidate_views` 为空 + `dataview_availability` 存在 → 使用 `dataview_availability.available_views`
3. `candidate_views` 为空 + `dataview_availability` 不存在（如 update pipeline 跳过了 bkn-env）→ warn 用户"无候选视图列表，请手动指定 view_id 或 view_name 辅助匹配"

**截断风险处理**：当 `dataview_availability.truncated == true` 时：
- 向用户发出 warning："平台视图列表可能存在截断，自动匹配可能遗漏目标视图。建议手动指定 view_id 或 view_name 辅助匹配。"
- 继续用已获取的候选列表执行匹配，不阻断流程
- 匹配结果为 pending 时，提示用户可能需要手动确认 view_id

## 约束

- 只做对象级绑定，属性映射交 `bkn-map`
- 绑定值必须是 `resource_id`，不用名称替代
- 多候选无法裁决时输出 pending，不强选
- 不静默忽略数据源不一致
