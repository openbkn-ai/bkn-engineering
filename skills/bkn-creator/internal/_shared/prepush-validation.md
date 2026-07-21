# 推送前关系映射完整性预检

可复用模块，供所有 pipeline（create/update/feedback/copy）的推送步骤前调用。

## 做什么

扫描 `{network_dir}/bkn/` 目录，检查跨文件引用一致性，防止推送时平台返回"属性不存在"等错误。

## 输入

- `network_dir`：网络目录（`.bkn` 文件位于 `{network_dir}/bkn/`）
- `skip_local_objects`：是否跳过 `存储位置: local` 对象的检查（默认 true）

## 检查规则

### 1. 关系映射完整性

遍历所有 `{network_dir}/bkn/relation_types/*.bkn` 文件：

```
对每个关系文件:
  1. 解析 Endpoint 中的 Source 和 Target 对象 ID
  2. 确认 Source 对象文件存在于 `{network_dir}/bkn/object_types/` 目录
  3. 确认 Target 对象文件存在于 `{network_dir}/bkn/object_types/` 目录
  4. 解析 Endpoint 表的 Type 列（direct / data_view / filtered_cross_join）
  
  **direct 类型**：
    5. 解析 Mapping Rules 中的 Source Property 和 Target Property
    6. 确认 Source Property 存在于 Source 对象的 Data Properties 中
    7. 确认 Target Property 存在于 Target 对象的 Data Properties 中
    
  **data_view 类型**：
    5. 解析 Mapping View 表，确认 Type 和 ID 非空
    6. 解析 Source Mapping 中的 Source Property 和 View Property
    7. 确认 Source Property 存在于 Source 对象的 Data Properties 中
    8. 确认 View Property 非空（中间视图字段存在性校验在 logics 层执行，需平台 API）
    9. 解析 Target Mapping 中的 View Property 和 Target Property
    10. 确认 Target Property 存在于 Target 对象的 Data Properties 中
    11. 确认 View Property 非空
    
  **filtered_cross_join 类型**：
    5. 检查 Source Condition 和 Target Condition 格式正确（不检查属性存在性，因为条件可能跨对象）
    
  **pending 关系处理**：
    - 如果 Mapping View ID 为空或占位符（如"待绑定"、"TBD"），标记为 warning 而非 fail
    - pending 关系不阻断推送，但在报告中标注
```

### 2. 动作参数绑定完整性

遍历所有 `{network_dir}/bkn/action_types/*.bkn` 文件：

```
对每个动作文件:
  1. 解析 Bound Object 表中的对象 ID
  2. 确认对象文件存在于 `{network_dir}/bkn/object_types/` 目录
  3. 解析 Parameter Binding 中引用的属性名
  4. 确认引用的属性存在于绑定对象的 Data Properties 中
  5. 解析 Pre-conditions 中引用的属性名
  6. 确认引用的属性存在于绑定对象的 Data Properties 中
```

### 3. Concept Group 成员存在性

遍历所有 `{network_dir}/bkn/concept_groups/*.bkn` 和 `{network_dir}/bkn/network.bkn` 中的 concept_groups：

```
对每个 concept_group:
  确认 members 列表中的每个对象 ID 存在于 `{network_dir}/bkn/object_types/` 目录
```

### 4. Network Overview 一致性

检查 `{network_dir}/bkn/network.bkn` 的 Network Overview 表格：

```
确认 Overview 中列出的每个 ID 都存在对应的 .bkn 文件
确认每个 .bkn 文件的 ID 都在 Overview 中被引用
```

### 5. local 对象豁免

- `存储位置: local` 的对象不参与关系映射的 **自身端** Property 存在性检查（因为 local 对象可能没有完整的数据属性定义）
- 关系两端中，`platform` 端的 Property 存在性检查仍需执行。例：`platform_A → local_B` 的关系，platform_A 端的 Source Property 必须存在于 A 的 Data Properties 中，local_B 端的 Target Property 跳过检查
- local 对象仍需参与 Endpoint 对象存在性检查、Concept Group 成员检查、Network Overview 一致性检查

## 输出

```yaml
prepush_validation:
  status: pass | fail
  checks:
    relation_mapping:
      status: pass | fail
      errors:
        - file: "bkn/relation_types/xxx.bkn"
          error: "起点关联属性[prop]在起点对象类[Obj]中不存在"
          source_object: ""
          source_property: ""
          target_object: ""
          target_property: ""
    action_binding:
      status: pass | fail
      errors:
        - file: "bkn/action_types/xxx.bkn"
          error: "属性[prop]在对象类[Obj]中不存在"
          bound_object: ""
          referenced_property: ""
    concept_group_members:
      status: pass | fail
      errors:
        - file: "bkn/concept_groups/xxx.bkn"
          missing_member: ""
    network_overview:
      status: pass | fail
      missing_in_overview: []    # 文件存在但 Overview 未引用
      missing_files: []          # Overview 引用但文件不存在
  summary:
    total_relations_checked: 0
    total_actions_checked: 0
    total_errors: 0
```

## 使用方式

在 pipeline 的推送步骤前调用：

```markdown
## 推送前预检

执行 `_shared/prepush-validation.md` 预检模块。

- `status: pass` → 继续推送
- `status: fail` → 列出全部错误，阻断推送，进入修复流程
```

## 错误修复指引

按错误类型分类修复：

| 错误类型 | 修复策略 |
|---------|---------|
| Source/Target 对象不存在 | 检查关系文件的 Endpoint ID 是否拼写错误，或创建缺失对象 |
| 属性不存在 | 检查属性名是否与对象 Data Properties 中的 name 一致，修正映射 |
| 动作参数属性不存在 | 同上，修正 Parameter Binding 中的属性引用 |
| Concept Group 成员缺失 | 检查 member ID 是否拼写错误，或从 group 中移除 |
| Network Overview 不一致 | 同步更新 Overview 表格，添加遗漏 ID 或移除过期引用 |
