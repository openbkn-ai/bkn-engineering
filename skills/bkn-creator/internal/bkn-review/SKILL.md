---
name: bkn-review
description: 网络质量评审、评分与调整分发。闭环关键。
---

# 网络评审

公约：`../_shared/contract.md`

## 做什么

分析测试结果，计算质量评分，判定通过或需要调整，分发调整任务到对应 skill。

**插件降级说明**：当 `plugin_availability.rules == unavailable` 或 `plugin_availability.test == unavailable` 时，评分维度自动降级。

## 输入

- 测试执行结果（`bkn-test` 输出；若 test 插件不可用则为空）
- 当前网络状态（对象/关系/绑定/映射汇总）
- 业务规则清单（如有；若 rules 插件不可用则为空）
- `plugin_availability`：来自 `pipeline_state.yaml`

## 质量评分

| 维度 | 检查内容 | 权重 |
|------|---------|------|
| 结构完整性 | 每个对象有主键、有描述、属性非空 | 20% |
| 关系健全性 | 关系两端对象存在、方向一致、无悬空引用、**属性引用存在**（Source/Target Property 存在于对应对象 Data Properties） | 20% |
| 绑定率 | bound_objects / total_platform_objects | 20% |
| 映射覆盖率 | (mapped + waived) / total_platform_properties | 20% |
| 规则覆盖率 | 有规则的对象数 / 总对象数；**拆分为子项**（见下方） | 10% |
| 测试通过率 | passed_cases / total_cases | 10% |

### 规则覆盖率子项

规则覆盖率维度由两个子项组成：

| 子项 | 检查内容 | 占规则覆盖率权重 |
|------|---------|-----------------|
| 规则覆盖广度 | 有规则的对象数 / 总对象数 | 60% |
| Skill 文件质量 | 读取 `bkn-rules` 的 `skill_self_check`，5 项自检全部 pass 为 100%，每项 fail 扣 20% | 40% |

- `skill_self_check` 来自 `bkn-rules` 输出的 `skill_self_check` 字段
- 若 `bkn-rules` 未输出 `skill_self_check`（旧版本或插件不可用），Skill 文件质量子项记为 0

### 插件降级评分逻辑

当 `plugin_availability.rules == unavailable` 时：
- **规则覆盖率维度**：Skill 文件质量子项权重降为 0，仅保留规则覆盖广度子项（权重 10%）
- 若 rules 插件完全不可用导致无规则数据，规则覆盖率维度标记为 N/A，不参与评分
- 剩余维度权重等比放大：结构 22.2%、关系 22.2%、绑定 22.2%、映射 22.2%、测试 11.1%

当 `plugin_availability.test == unavailable` 时：
- **测试通过率维度**：标记为 N/A，不参与评分
- 剩余维度权重等比放大：结构 22.2%、关系 22.2%、绑定 22.2%、映射 22.2%、规则 11.1%

**双重降级**（rules 和 test 都不可用）：
- 规则覆盖率 + 测试通过率均标记为 N/A
- 剩余维度权重等比放大：结构 25%、关系 25%、绑定 25%、映射 25%

### 存储位置过滤

- `存储位置: local` 的对象 **不参与绑定率和映射覆盖率计算**
- 绑定率分母只计算 `platform` 对象数量
- 映射覆盖率分母只计算 `platform` 对象的属性数量
- local 对象仍需通过结构完整性检查（有主键、有描述、属性非空）

- 综合分 >= 80 → 通过，可推送
- 60-80 → 有风险，需用户确认
- < 60 → 建议修复

### `bind_mode == deferred` 时的评分调整

当 `bkn-env` 判定 `bind_mode: deferred`（无可用数据视图）时：
- 绑定率（20%）和映射覆盖率（20%）标记为 N/A，不参与评分
- 剩余维度权重等比放大：结构完整性 33%、关系健全性 33%、规则覆盖率 17%、测试通过率 17%

### 一票否决

以下情况无论综合分如何，verdict 强制为 fail：
- `bind_mode == full` 且绑定率 < 30%（有视图但几乎没绑上），**且 platform 对象数量 >= 3**
- 结构完整性维度 < 50%（基础质量不达标）

### 外部校验锚点

以下维度必须包含外部验证数据，不可纯粹依赖 Agent 自评：
- **结构完整性**：`openbkn bkn validate` 通过 → 该维度底线 60%；未通过 → 上限 40%
- **绑定率**：来自 `bkn-bind` 的 `binding_summary` 实际数值
- **映射覆盖率**：来自 `bkn-map` 的 `mapping_gate_summary` 实际数值
- **关系健全性-属性引用**：`_shared/prepush-validation.md` 预检结果，有错误则该维度分数减半

## 调整分发

不达标时，按失败根因分发到对应 skill：

| 根因 | 调整 skill |
|------|-----------|
| 对象/关系问题 | `bkn-doctor` → `bkn-draft` |
| 绑定问题 | `bkn-bind` → `bkn-map` → `bkn-backfill` |
| 规则缺失 | `../_plugins/bkn-rules/SKILL.md`（仅当 `plugin_availability.rules == available` 时） |
| 属性映射不足 | `bkn-map` |

**插件不可用时的分发调整**：
- 若 `plugin_availability.rules == unavailable` 且根因涉及"规则缺失"：
  - 不分发到 bkn-rules，改为输出提示"规则插件不可用，请手动补充业务规则或安装插件"
  - 在 `adjustment_plan` 中标记 `target_skill: none(plugin_unavailable)`

每轮调整必须有 diff（改了什么）。最多自动循环 3 轮，超过则暂停让用户介入。

## 输出

```yaml
quality_score: 0
score_breakdown: {structure, relations, binding, mapping, rules, tests}
plugin_degraded_dimensions: []  # 记录因插件不可用而降级的维度，如 ["rules", "tests"]
verdict: pass | warn | fail
adjustment_plan:
  - target_skill: ""
    reason: ""
    suggested_action: ""
iteration: {current_round, max_rounds, score_history}
```

## 约束

- 评分必须基于实际测试数据，不编造
- 不替代具体 skill 做修复，只做诊断和分发
- 插件不可用时在输出中明确标注降级原因