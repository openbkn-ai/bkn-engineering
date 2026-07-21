# 属性级映射判定规则

## 三态定义

### `mapped`（同时满足）
- 已定位明确字段来源
- 字段语义与属性语义一致
- 字段类型可兼容
- 映射路径稳定

必填：property_name / view_id / field_path / confidence / type_check / reason

### `waived`（满足任一）
- 属性为治理/展示补充项，非必填
- 业务确认允许暂不绑定
- 可由后续人工或规则推导补足

waived 是"有依据的豁免"，不是"找不到就跳过"。
必填：property_name / waive_reason / waive_basis / impact

### `blocked`（出现任一）
- 无可接受字段来源
- 字段语义明显不一致
- 类型冲突无法解释
- 多候选无法裁决

必填：property_name / block_reason / blocking_points / suggested_fix

## 类型校验

| 可接受 | 需谨慎/阻断 |
|--------|-------------|
| 字符串 → 文本字段 | 数值 → 自由文本 |
| 时间 → 标准时间字段 | 时间 → 不稳定字符串 |
| 数值 → 数值字段 | 主键 → 非唯一字段 |
| 枚举 → 稳定状态字段 | 属性依赖复杂拼接 |

## 覆盖率

`coverage = (mapped_count + waived_count) / total_properties`

约束：`mapped + waived + blocked = total`

## 放行建议

- `blocked_count > 0` → 不放行
- `coverage < 100%` 且要求全覆盖 → 不放行
- `coverage = 100%` 且 `blocked_count = 0` → 放行
- 有 waived 时须展示豁免依据

## 禁止

- 不将"猜测大概率正确"标为 mapped
- 不将"暂时没字段"标为 waived
- 不在类型冲突时继续映射
- 不漏写 waived / blocked 原因
