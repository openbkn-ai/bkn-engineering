---
name: bkn-extract
description: 从文本提取对象/关系/动作，输出四分组结构化清单。
---

# 对象关系提取

公约：`../_shared/contract.md`

## 做什么

给定业务文本和领域识别结果，提取对象类、关系类、动作候选，分组输出。

## 输入

- `source_text`：业务文本
- `domain`：`bkn-domain` 的识别结果
- `domain_reference`（可选）：对应领域参考文件路径

## 流程

1. 按领域参考（或通用规则 `../bkn-domain/references/generic-extraction.md`）提取
2. 对象分为四组：
   - **显式对象**（`explicit`）：文本直接出现，有明确定义
   - **推断对象**（`inferred`）：领域闭环补全（每项必须含推断理由）
   - **待确认对象**（`pending`）：证据不足，需用户确认
   - **排除候选**（`rejected`）：复合表达/噪音/不适合建模的
3. 对每个对象判定 **存储位置**（`platform` 或 `local`）：
   - **`platform`**：需要与外部系统/数据视图绑定的业务对象（如物料、订单、工单等），有实际数据源
   - **`local`**：仅用于模型内部逻辑推理、规则锚定、中间态表示的对象（如监控任务、状态标记、规则容器等），无外部数据源
   - 判定依据：是否提及数据来源/系统对接/视图绑定；若无则为 `local`
   - 不确定时默认标记为 `platform`，并在业务含义中备注
4. 提取关系（name 用中文业务名）
5. 提取动作候选
6. 领域闭环完整性检查（缺失对象补入推断组）
7. 质量校验：命名归一化、去重、引用完整性

## 输出

所有键名和分组标签使用中文，便于非技术人员阅读：

```yaml
对象清单:
  显式对象:
    - 名称: ""
      别名: []
      业务含义: ""
      候选主键: ""
      来源证据: ""
      存储位置: platform | local
  推断对象:
    - 名称: ""
      别名: []
      业务含义: ""
      候选主键: ""
      推断理由: ""
      存储位置: platform | local
  待确认对象:
    - 名称: ""
      不确定原因: ""
      存储位置: platform | local
  排除候选:
    - 候选名: ""
      排除原因: ""
关系清单:
  - 名称: ""
    关系ID: ""
    源对象: ""
    目标对象: ""
    基数: ""
    来源证据: ""
动作候选:
  - 名称: ""
    触发条件: ""
    目标对象: ""
    风险等级: 低 | 中 | 高
```

### 风险等级说明

| 等级 | 标签 | 含义 | 示例 |
|------|------|------|------|
| 低 | `低` | 只读或对单条记录的非破坏性操作 | 查询、生成报表 |
| 中 | `中` | 影响多条记录或触发下游流程 | 批量状态变更、MRP 计算 |
| 高 | `高` | 删除、跨系统推送、审批流触发、资金相关 | 删除网络、推送生产计划 |

## 升级条件

以下情况建议 pipeline 调用 `bkn-doctor`：
- 待确认对象 >= 3
- 关系方向冲突
- 主键缺失
- 清单质量不足

## 业务规则检查（阶段一末尾）

清单确认后，逐对象扫描规则：

| 规则类型 | 示例 |
|---------|------|
| 主键规则 | forecast 以 billno 为主键 |
| 外键规则 | mrp.rootdemandbillno → forecast.billno |
| 过滤规则 | mrp 仅取 closestatus_title='正常' |
| 状态枚举 | 采购状态 = normal / watch / abnormal |
| 计算规则 | coverage = (mapped + waived) / total |
| 层级规则 | BOM 多版本取 audit_date 最近 |
| 约束规则 | alt_priority == 0 才参与 MRP |

输出 `rule_extraction_check`，供后续 `bkn-rules` 直接消费：

```yaml
rule_extraction_check:
  - rule_id: "RE-001"
    rule_type: 主键规则 | 外键规则 | 过滤规则 | 状态枚举 | 计算规则 | 层级规则 | 约束规则
    description: ""
    related_objects: [""]
    source_evidence: ""
    confidence: high | medium | low
    risk_level: 低 | 中 | 高
```
