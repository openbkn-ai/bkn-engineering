---
name: bkn-domain
description: 评分式领域识别。从业务文本判断所属领域。
---

# 领域识别

公约：`../_shared/contract.md`

## 做什么

对输入文本做关键词评分，在 supply_chain / crm_sales / project_delivery / generic 之间选择领域。

## 输入

- `source_text`：业务文本（PRD、流程说明等）
- `domain_hints`（可选）：用户指定的候选领域

## 流程

1. 提取关键词、专业缩写、流程节点
2. 按权重累计各领域得分（规则见 `references/scoring-rules.md`）
3. 归一化：`normalized = raw_score / domain_max × 100`
4. 判定路径：

| 条件 | 判定 | 建议动作 |
|------|------|------|
| `normalized_top >= 70` | 高置信直通 | 领域提取（pipeline 可要求确认） |
| `normalized_top >= 20` 且分差 >= 8 | 高置信命中 | 领域提取（pipeline 可要求确认） |
| `normalized_top >= 12` 且分差 < 8 | 候选冲突 | 请用户确认 |
| 其他 | 未识别 | 通用提取 |

基础门槛：`raw_score_top >= 4`，低于此直接走通用。

**确认决策权**：`next_action` 为建议，实际确认策略由调用方 pipeline 决定。pipeline 可选择"无论是否命中都需要用户确认"。

## 输出

```yaml
top_domain: ""
normalized_top: 0
confidence: high | medium | low
evidence: [{keyword, weight, matched_text}]
next_action: domain_extract | ask_user_confirm | generic_extract
```

## 参考

- `references/scoring-rules.md`
- `references/domain-supply-chain.md`
- `references/domain-crm-sales.md`
- `references/domain-project-delivery.md`
