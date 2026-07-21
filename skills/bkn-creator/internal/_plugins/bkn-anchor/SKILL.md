---
name: bkn-anchor
description: 将业务规则 Skill 锚定为 BKN 网络中的孤悬对象类。
---

# Skill 锚定

公约：`../../_shared/contract.md`

## 做什么

把 `bkn-rules` 生成的业务规则 Skill，作为孤悬对象类写入 BKN 网络，
让网络与其规则 Skill 自包含——查网络就能看到配套规则。

## 输入

- `business_rules_skill`：`bkn-rules` 的输出
- `network_dir`：网络目录（`.bkn` 文件位于 `{network_dir}/bkn/`）

## 锚定方式

为每个业务规则 Skill 创建一个独立 object_type（无关系指向其他业务对象）：

```yaml
object_type:
  name: "{skill_name}"
  description: "{规则摘要}"
  properties:
    - name: skill_path
      type: string
      description: "Skill 文件路径"
    - name: rule_type
      type: string
      description: "规则类型"
    - name: rule_count
      type: integer
      description: "规则条数"
    - name: source
      type: string
      description: "规则来源"
    - name: related_objects
      type: string
      description: "关联业务对象"
    - name: version
      type: string
      description: "版本"
```

锚定对象需在 `{network_dir}/bkn/network.bkn` 的 `concept_groups` 中引用，归入「业务规则」分组。若该分组不存在，在 `{network_dir}/bkn/network.bkn` 中创建：

```yaml
concept_groups:
  - id: business_rules
    name: 业务规则
    members:
      - "{anchored_object_id}"
```

## 流程

1. 读取 `bkn-rules` 输出的 `rule_groups`
2. 为每个 group 生成对应的孤悬 object_type
3. 写入 `{network_dir}/bkn/object_types/` 下的 .bkn 文件（直接写入，不走 bkn-backfill）
4. 更新 `{network_dir}/bkn/network.bkn` 的 concept_groups（添加或更新「业务规则」分组）
5. 回读校验

## 输出

```yaml
anchored_objects:
  - object_name: ""
    skill_path: ""
    rule_count: 0
anchor_status: success | partial | failed
```

## 锚定与推送的配合

锚定对象写入 .bkn 文件后随网络整体推送（推送目标为 `{network_dir}/bkn/`）。锚定时需遵守门禁规则：
- 锚定对象的属性类型必须为合法类型（`string`/`integer` 等）
- Display Key 不可为空（建议用 `skill_path`）
- 无需 Data Source（孤悬对象无数据视图绑定）

## 约束

- 孤悬对象不与业务对象建关系
- 不修改已有业务对象定义
- 归入专用 concept_group，便于区分
- 推送前必须通过门禁自检
