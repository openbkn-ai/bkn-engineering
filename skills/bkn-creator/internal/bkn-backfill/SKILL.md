---
name: bkn-backfill
description: 将绑定/映射结果写回 .bkn 文件。
---

# BKN 回填

公约：`../_shared/contract.md`

## 做什么

把 `bkn-bind` 和 `bkn-map` 的结果合并，写回 `.bkn` 文件，完成数据视图绑定闭环。

## 输入

- `binding_decision_list`：对象级绑定结果
- `property_mapping_draft`：属性级映射结果
- `relation_binding_result`：`bkn-relation-bind` 的输出（关系类型 + 中间视图）
- `relation_mapping_draft`：`bkn-map` 的关系类映射结果
- `network_dir`：网络目录（`.bkn` 文件位于 `{network_dir}/bkn/`）

## 流程

1. **属性回灌门禁**（P1-2）：
   - 读取 `{network_dir}/pipeline_state.yaml`（文件不存在时视为 `map_completed` 缺失）
   - **`map_completed == true`** → 继续
   - **`map_completed == skipped`**（`bind_mode == deferred` 场景）→ 记录 `map_status: skipped (deferred)`，继续
   - **`map_completed` 不存在或为 false**：
     - 用户明确要求跳过 → 在 `pipeline_state.yaml` 中追加 `skipped_steps: [bkn-map]`，输出警告后继续
     - 否则，**阻断并回交 pipeline 调度**，输出：
       `"bkn-backfill 需要 bkn-map 先执行完成（map_completed 未就绪）。请 pipeline 调度 bkn-map 后再调用本 skill。"`
2. 合并绑定变更 + 属性变更 + 映射状态为统一差异清单
3. **存储位置过滤**：
   - 跳过 `存储位置: local` 的对象（本地对象无需数据视图绑定，不参与回填）
   - 在输出摘要中单独列出被跳过的 local 对象
4. **对象类回填**（仅 platform 对象）：
   - Data Source 绑定值替换（名称/技术名 → 稳定 `view_id`）
   - Mapped Field 更新为数据视图真实字段名
   - 数据视图中不存在的属性：默认标注 Mapped Field 为 `-`（保留属性定义，便于后续补绑）；用户可指定删除该行
   - 不改动 Description 或其他业务语义字段
5. **级联修正**（关键改进）：
   - 构建 `rename_map`：`{object_id: {old_property_name → new_property_name}}`
     - 来源 1：`bkn-map` 的 `property_regen_summary.rename` 动作（draft 属性名 → 视图字段名）
     - 来源 2：用户手动指定的属性重命名
   - 扫描 `{network_dir}/bkn/relation_types/*.bkn` 的 Mapping Rules，将 Source Property / Target Property 中引用了已重命名属性的值替换
   - 扫描 `{network_dir}/bkn/action_types/*.bkn` 的 Parameter Binding / Pre-conditions，将 Binding 列中引用了已重命名属性的值替换
   - 级联修正只改引用名，不改语义
6. **关系类回填**（新增）：
   - 仅处理 `relation_binding_result` 中 `status: confirmed` 的关系
   - **direct 类型**：
     - 回填 Endpoint 表的 Type 列为 `direct`
     - 回填 Mapping Rules 表（Source Property → Target Property）
   - **data_view 类型**：
     - 回填 Endpoint 表的 Type 列为 `data_view`
     - 回填 Mapping View 表（Type = data_view, ID = intermediate_view_id）
     - 回填 Source Mapping 表（Source Property → View Property）
     - 回填 Target Mapping 表（View Property → Target Property）
   - **pending 关系**：
     - 仅回填 Endpoint 表（Type 列保留占位符或留空）
     - Mapping View / Source/Target Mapping 段不生成
   - 跳过 `relation_mapping_quality: blocked` 的关系（在输出中标记）
7. **推送前关系映射完整性预检**（执行 `../_shared/prepush-validation.md`）：
   - 检查所有 `relation_types` 的 Source/Target Property 是否存在于对应对象的 Data Properties
   - 检查所有 `action_types` 的 Parameter Binding 属性是否存在于绑定对象的 Data Properties
   - 检查 Concept Group 成员和 Network Overview 一致性
   - 检查 data_view 类型关系的 Mapping View ID 是否非空
   - 检查 data_view 类型关系的 Source/Target Mapping 属性是否存在于对应视图 schema
   - `local` 对象跳过属性存在性检查，但仍参与对象存在性检查
   - `pending` 关系仅 warning，不阻断
   - **预检失败 → 阻断写入，列出全部错误**，提示修复后重试
8. 执行写入（写入 `{network_dir}/bkn/` 下对应的 `.bkn` 文件）
9. 回读校验（文件存在、关键字段已替换、级联引用一致、关系类结构完整）
10. 输出精简摘要，提示用户检查

## 输出

```yaml
backfill_plan:
  binding_replacements: [{object, old_value, new_view_id}]
  property_changes: [{object, property, action, layer}]
  cascade_renames: [{file, table, old_ref, new_ref}]
  # 新增：关系类回填计划
  relation_replacements:
    - 关系ID: ""
      关系名称: ""
      关系类型: direct | data_view
      actions:
        - action: ""                  # fill_endpoint_type / fill_mapping_view / fill_source_mapping / fill_target_mapping
          target_section: ""          # Endpoint / Mapping View / Source Mapping / Target Mapping
          old_value: ""
          new_value: ""
  skipped_local_objects: [对象名]
  skipped_pending_relations: [关系名]  # pending 关系，跳过回填
  map_gate: {map_completed: true|false|skipped, user_skipped_map: true|false, blocked_for_map: true|false}
  prepush_validation: {status, errors}  # 写入前预检结果
backfill_status: success | partial | failed
verification: 
  files_checked: 0
  fields_replaced: 0
  cascade_refs_fixed: 0
  relations_filled: 0              # 新增：已回填的关系数量
  pending_relations: 0             # 新增：pending 关系数量
  issues: []
```

## 约束

- 只写 Data Source、Mapped Field 和级联引用，不改业务语义字段
- 级联修正仅限属性名引用，不修改关系方向或动作语义
- **写入前必须执行 `../_shared/prepush-validation.md` 预检，失败则阻断写入**
- **执行回填前必须确认 `map_completed == true` 或 `skipped`，否则阻断并回交 pipeline 调度**
- 写入后必须回读校验，包括级联引用一致性
- 回填失败时保留差异清单，可重试
