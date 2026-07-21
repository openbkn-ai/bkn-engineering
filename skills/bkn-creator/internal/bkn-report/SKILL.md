---
name: bkn-report
description: 生成阶段报告或最终归档报告。
---

# 报告生成

公约：`../_shared/contract.md`

## 做什么

汇总 pipeline 各阶段产物，生成结构化报告并归档。

## 输入

- `pipeline`：当前流程类型
- `artifacts`：各阶段产物（建模清单、绑定结果、测试结果、推送结果等）
- `network_dir`：归档目录

## 报告类型

| 类型 | 触发时机 | 内容 |
|------|---------|------|
| 阶段报告 | pipeline 中间节点 | 当前阶段摘要 + 下一步 |
| 最终报告 | pipeline 完成 | 全流程回顾 + 产物清单 + 质量评分 |
| 测试报告 | bkn-test 完成后 | 测试结果 + 覆盖率 + 通过率 |
| 诊断报告 | validate pipeline 完成 | 检查结果 + 问题清单 + 修复建议 |

## 最终报告结构

```
1. 网络概览：名称、领域、对象数、关系数
2. 建模摘要：路径（A/B/C）、收敛轮数
3. 绑定摘要：绑定率、覆盖率、风险项
4. 对象类详情：每个对象的属性列表、主键、映射状态（关键内容）
5. 测试摘要：通过率、关键失败项
6. 业务规则：规则数、锚定对象数
7. Q&A 验证：通过率、验证路径
8. 问题与修复：pipeline 执行中遇到的问题及解决方案
9. 产物清单：文件路径列表
```

## 诊断报告结构

当 `report_type: diagnosis` 时生成，输出文件为 `DIAGNOSIS_REPORT.md` / `DIAGNOSIS_REPORT.html`：

```
1. 网络概览：名称、kn_id、来源（本地/pulled）、检查时间
2. 检查摘要：各阶段状态总览（pass/fail/skipped）
3. 门禁检查详情：每条门禁结果 + 不达标项明细
4. 预检详情：关系映射/动作绑定/concept_group 检查结果
5. 静态检查详情：schema_review 结果（如有）
6. 质量评分：综合分 + 各维度分 + 降级说明
7. 问题清单：按严重度排序的问题列表（critical/warning/info）
8. 修复建议：每条建议 + 目标 skill + 预估工作量
9. 后续行动：推荐路由（update/feedback）+ 用户可选路径
```

诊断报告输入来源：
- `pipeline_state.yaml` 中的 `validation_result` 字段
- 各阶段检查结果（gate_check、prepush_validation、static_check、quality_score）
- `diagnosis` 字段中的问题清单和修复建议

## 输出

### 1. Markdown 报告（必须）

- 文件：`{network_dir}/reports/REPORT.md`
- 用途：版本控制友好、方便 diff
- 内容：完整报告（上述 1–9 节）

### 2. HTML 报告（可选）

- 生成条件：`references/report-template.html` 模板存在时生成，不存在时跳过
- 模板：`references/report-template.html`
- 文件：`{network_dir}/reports/REPORT.html`
- 用途：可视化展示、直接浏览器打开
- 生成方式：读取模板，替换 `{{placeholder}}` 占位符
- 占位符映射：

| 占位符 | 数据来源 |
|--------|---------|
| `{{network_name}}` | network.bkn → name |
| `{{domain}}` | network_context.domain |
| `{{timestamp}}` | 当前时间 |
| `{{trace_id}}` | ARCHIVE_ID |
| `{{quality_score}}` | bkn-review 评分（未执行写 N/A） |
| `{{verdict_class}}` | score >= 80 → pass, >= 60 → warn, < 60 → fail |
| `{{score_rows}}` | 各维度评分行 `<tr><td>维度</td><td>分数</td><td>权重</td></tr>` |
| `{{object_count}}` | 对象类数量 |
| `{{relation_count}}` | 关系类数量 |
| `{{binding_rate}}` | 绑定率 |
| `{{mapping_coverage}}` | 映射覆盖率 |
| `{{object_detail_rows}}` | 对象类属性详情（每对象一节，含属性表格 + Keys） |
| `{{test_rows}}` | 测试结果行 |
| `{{rule_count}}` | 业务规则数 |
| `{{anchor_count}}` | 锚定对象数 |
| `{{qa_pass_rate}}` | Q&A 通过率 |
| `{{artifact_list}}` | 产物路径 `<li>` 列表 |

### 诊断报告 HTML 占位符映射

当生成诊断报告（`report_type: diagnosis`）时，使用以下占位符：

| 占位符 | 数据来源 |
|--------|---------|
| `{{network_name}}` | network.bkn → name 或平台查询结果 |
| `{{kn_id}}` | network.bkn → kn_id 或平台查询结果 |
| `{{source}}` | pipeline_state.yaml → source（local / pulled_from_platform） |
| `{{check_timestamp}}` | 检查执行时间 |
| `{{trace_id}}` | ARCHIVE_ID |
| `{{gate_status}}` | gate_check.status（pass/fail） |
| `{{gate_rows}}` | 门禁检查结果行 |
| `{{prepush_status}}` | prepush_validation.status（pass/fail） |
| `{{prepush_errors}}` | 预检错误列表 |
| `{{static_check_status}}` | static_check.status（pass/fail/blocked/skipped） |
| `{{quality_score}}` | bkn-review 评分 |
| `{{verdict_class}}` | score >= 80 → pass, >= 60 → warn, < 60 → fail |
| `{{score_rows}}` | 各维度评分行 |
| `{{problem_rows}}` | 问题清单行（按严重度排序） |
| `{{fix_rows}}` | 修复建议行 |
| `{{routing_primary}}` | diagnosis.routing_suggestion.primary |
| `{{routing_reason}}` | diagnosis.routing_suggestion.reason |

### `{{object_detail_rows}}` 生成格式

每个对象类生成如下 HTML 片段：

```html
<div class="object-section">
  <h3>{{object_name}} <small class="meta">{{存储位置}}</small></h3>
  <table class="properties-table">
    <tr><th>Name</th><th>Display Name</th><th>Type</th><th>Mapped Field</th><th>Status</th></tr>
    <!-- 每个属性一行 -->
    <tr><td>{{property_name}}</td><td>{{display_name}}</td><td>{{type}}</td><td>{{mapped_field}}</td><td class="{{status_class}}">{{status}}</td></tr>
  </table>
  <p class="keys">Primary Key: {{primary_key}} | Display Key: {{display_key}}</p>
</div>
```

字段说明：

| 字段 | 来源 |
|------|------|
| `{{object_name}}` | object_types/*.bkn → frontmatter.name |
| `{{存储位置}}` | 对象清单 → 存储位置（platform / local），local 对象标注灰色 |
| `{{property_name}}` | Data Properties 表格 → Name 列 |
| `{{display_name}}` | Data Properties 表格 → Display Name 列 |
| `{{type}}` | Data Properties 表格 → Type 列 |
| `{{mapped_field}}` | Data Properties 表格 → Mapped Field 列（无映射写 `-`） |
| `{{status}}` | bkn-map 输出 → 映射状态（mapped / waived / blocked），仅 platform 对象显示 |
| `{{status_class}}` | mapped → pass, waived → warn, blocked → fail |
| `{{primary_key}}` | object_types/*.bkn → Keys → Primary Keys |
| `{{display_key}}` | object_types/*.bkn → Keys → Display Key |

**local 对象处理**：存储位置为 `local` 的对象，`Mapped Field` 列显示 `-`，`Status` 列不显示（无数据视图绑定），整体标注灰色样式。

## 约束

- 报告不编造数据，数值必须来自实际产物
- 归档路径遵循 `_shared/contract.md` 中的归档规则
- Markdown 报告为必须产物；HTML 报告在模板可用时生成，不可用时跳过并在 Markdown 报告末尾注明
- HTML 报告必须基于 `references/report-template.html` 模板生成，不可自行编写 HTML
