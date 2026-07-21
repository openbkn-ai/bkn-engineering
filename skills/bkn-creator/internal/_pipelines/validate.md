# 验证流程（Validate）

对现有知识网络执行独立检查与质量评估，不执行任何修改操作。

## Skill 路径索引

| skill | 路径（相对本文件） | 类型 |
|-------|-------------------|------|
| bkn-env | `../bkn-env/SKILL.md` | 核心 |
| bkn-test | `../_plugins/bkn-test/SKILL.md` | **插件** |
| bkn-review | `../bkn-review/SKILL.md` | 核心 |
| bkn-doctor | `../bkn-doctor/SKILL.md` | 核心 |
| bkn-report | `../bkn-report/SKILL.md` | 核心 |
| bkn-openbkn | `../bkn-openbkn/SKILL.md` | 核心 | OpenBKN CLI 操作（内化） |
| 公约 | `../_shared/contract.md` | — |
| 插件检测 | `../_shared/plugin-check.md` | — |
| 预检模块 | `../_shared/prepush-validation.md` | — |

> **插件类型**：路径指向 `_plugins/` 目录，开源版可能不存在。pipeline 调用前需检测 `plugin_availability`，不可用时执行降级分支。

## 阶段总览

```
输入识别（network_dir / kn_id / 网络名称）
  ↓
阶段一：网络定位（kn_id/名称 → pull 到本地）
  ↓
阶段二：环境检测（bkn-env，goal=validate）
  ↓
阶段三：门禁自检（7条硬前置）
  ↓
阶段四：关系映射预检（prepush-validation）
  ↓
阶段五：静态检查（bkn-test schema_review，插件可跳过）
  ↓
阶段六：质量评审（bkn-review）
  ↓
阶段七：诊断分析（基于检查结果分析根因）
  ↓
阶段八：报告（bkn-report，诊断报告类型）
  ↓
阶段九：后续行动选项（路由到 update/feedback pipeline）
```

> `[]` 标记的为插件阶段，`plugin_availability` 检测不可用时自动跳过。

## 与其他 Pipeline 的区别

| 特性 | validate | create/update | feedback |
|------|---------|---------------|----------|
| 执行修改 | **否** | 是 | 是 |
| 输入来源 | 已有 BKN（本地/推送） | PRD/文档/变更 | feedback_brief |
| 核心目标 | 检查+评估+诊断 | 创建/更新网络 | 修复反馈问题 |
| 后续路由 | → update/feedback | 完成推送 | 完成修复 |

---

## 阶段一：输入识别与网络定位

### 输入类型判定

用户输入可能是以下形式之一：

| 输入类型 | 识别条件 | 处理方式 |
|---------|---------|---------|
| 本地目录 | 路径包含 `archives/` 或指向存在的目录，且目录下有 `bkn/` 子目录 | 直接使用 |
| kn_id | 符合 kn_id 格式（如 `kn_xxx`）且非本地路径 | pull 到临时目录 |
| 网络名称 | 字符串，非路径格式 | 查询匹配 → pull |
| 模糊描述 | "检查某网络的..." | 追问确认 |

### 本地目录输入

直接使用提供的 `network_dir`，跳过定位阶段。

### kn_id 输入处理

1. 调用 `bkn-openbkn` 执行 `openbkn bkn get <kn_id>` 获取网络信息
2. 确认网络存在，提取 `network_name`
3. 创建临时目录：`{workspace}/.validate_temp/{kn_id}/{TIMESTAMP}/`
4. 执行 `openbkn bkn pull <kn_id> --output {temp_dir}/bkn/`
5. 将 `temp_dir` 作为本次检查的 `network_dir`
6. 在 `pipeline_state.yaml` 中标记 `source: pulled_from_platform`

### 网络名称输入处理

1. 调用 `bkn-openbkn` 执行 `openbkn bkn list --name-pattern "{名称}"` 查询匹配
2. 展示匹配结果（最多 5 个），用户选择目标网络
3. 获取选中网络的 `kn_id`
4. 按 kn_id 输入处理流程执行 pull

---

## 阶段二：环境检测

| 步骤 | 读取 | 说明 |
|------|------|------|
| 执行环境检查 | `../bkn-env/SKILL.md` | 传入 `goal: validate` |
| 插件可用性检测 | `../_shared/plugin-check.md` | 由 bkn-env 内部执行 |
| 状态写入 | pipeline 执行 | 写入 `pipeline_state.yaml` |

`bkn-env` 输出的关键信息：
- `bind_mode`：full / deferred（决定门禁豁免）
- `plugin_availability`：rules / test 可用性
- `env_capability_matrix`：各项环境能力
- `dataview_availability`：数据视图列表（用于绑定完整性检查）

---

## 阶段三：门禁自检

扫描 `{network_dir}/bkn/` 目录，执行与 create pipeline 阶段七相同的门禁检查。

### 门禁条件

| # | 门禁条件 | 说明 | 豁免条件 |
|---|---------|------|---------|
| 1 | 绑定完整性 | platform 对象有 Data Source 且非占位符 | `bind_mode == deferred` |
| 2 | 无占位符 | 不存在 `待绑定` / `TBD` / 空 `view_id` | `bind_mode == deferred` |
| 3 | Type 合法性 | Data Properties Type 在合法类型列表内 | 无豁免 |
| 4 | ActionType 合法性 | Bound Object 表 Action Type 为 `add`/`modify`/`delete` | 无豁免 |
| 5 | Display Key 非空 | 所有对象 Display Key 非空 | 无豁免 |
| 6 | local 对象豁免 | `存储位置: local` 不参与门禁 1/2 | 无豁免（已设计） |
| 7 | platform 对象数量 | platform 对象数量 >= 3（一票否决） | 无豁免 |

### local 对象处理

- `存储位置: local` 的对象：
  - **豁免**门禁 1（绑定完整性）、门禁 2（无占位符）
  - **必须通过**门禁 3（Type 合法性）、门禁 5（Display Key 非空）
  - 不计入 platform 对象数量（门禁 7）

### 输出

```yaml
gate_check:
  status: pass | fail
  checks:
    binding_completeness: {status, affected_objects: []}
    no_placeholder: {status, affected_files: []}
    type_validity: {status, invalid_types: []}
    action_type_validity: {status, invalid_actions: []}
    display_key_nonempty: {status, empty_objects: []}
    platform_object_count: {status, count}
  summary:
    passed: 0
    failed: 0
    waived: 0  # 因 bind_mode deferred 或 local 对象豁免的检查数
```

---

## 阶段四：关系映射预检

调用 `_shared/prepush-validation.md` 执行关系映射完整性检查。

| 步骤 | 行为 |
|------|------|
| 执行预检 | 输入 `network_dir`，调用预检模块 |
| 结果记录 | 写入 `pipeline_state.yaml` |

### 输出

直接使用 `prepush-validation.md` 的标准输出格式：

```yaml
prepush_validation:
  status: pass | fail
  checks:
    relation_mapping: {status, errors: []}
    action_binding: {status, errors: []}
    concept_group_members: {status, errors: []}
    network_overview: {status, missing_in_overview: [], missing_files: []}
  summary:
    total_relations_checked: 0
    total_actions_checked: 0
    total_errors: 0
```

---

## 阶段五：静态检查（插件阶段）

**前置检测**：读取 `pipeline_state.yaml.plugin_availability.test`

| plugin_availability.test | 执行路径 |
|--------------------------|---------|
| `available` | 读取 `../_plugins/bkn-test/SKILL.md` → 执行 `schema_review` 模式 |
| `unavailable` | 跳过本阶段，在 `pipeline_state.yaml.completed_stages` 记录 `stage5_static_check: skipped(plugin_unavailable)` |

**当 `plugin_availability.test == available` 时执行**：

| 步骤 | 读取 | 说明 |
|------|------|------|
| 生成测试集 | `../_plugins/bkn-test/SKILL.md`（schema_review 模式） | 结构 + 规则存在性 + 绑定完整性检查 |
| BLOCKED 处理 | pipeline 判定 | 若数量/复杂度 BLOCKED → 标记为不达标，不重试（validate 不修复） |
| 结果记录 | pipeline 执行 | 测试结果写入 `pipeline_state.yaml` |

**validate 与 create/update 的区别**：
- validate 不执行修复循环，只记录问题
- BLOCKED 状态直接记录为"静态检查不达标"，不强制重试

### 输出

```yaml
static_check:
  status: pass | fail | blocked | skipped
  mode: schema_review
  quantity_check: {overall, object_coverage, relation_coverage}
  quality_check: {overall, complexity_score}
  test_cases_generated: 0
  failed_cases: []
  skip_reason: ""  # 仅当 skipped 时
```

---

## 阶段六：质量评审

| 步骤 | 读取 | 说明 |
|------|------|------|
| 执行评审 | `../bkn-review/SKILL.md` | 基于检查结果计算质量评分 |
| 评分记录 | pipeline 执行 | 写入 `pipeline_state.yaml` |

`bkn-review` 输入来源：
- 阶段三门禁检查结果
- 阶段四预检结果
- 阶段五静态检查结果（如有）

### bkn-review 降级评分

当 `plugin_availability` 不可用时，bkn-review 自动降级：
- `test == unavailable` → 测试通过率维度 N/A
- `rules == unavailable` → 规则覆盖率维度降级

`bind_mode == deferred` 时：
- 绑定率、映射覆盖率维度 N/A

### 输出

直接使用 `bkn-review` 的标准输出格式：

```yaml
quality_score: 0
score_breakdown: {structure, relations, binding, mapping, rules, tests}
plugin_degraded_dimensions: []
verdict: pass | warn | fail
```

---

## 阨段七：诊断分析

基于阶段三至六的检查结果，分析问题根因并生成修复建议。

**不新增 skill**：诊断逻辑在 pipeline 中直接执行，必要时调用 `bkn-doctor` 辅助分析建模问题。

### 诊断规则

按问题类型分类诊断：

| 问题类型 | 根因分析 | 修复建议 | 目标 pipeline |
|---------|---------|---------|--------------|
| 门禁 1/2 失败 | 绑定不完整/有占位符 | 重新执行绑定流程 | `update` |
| 门禁 3/4/5 失败 | Type/ActionType/Display Key 不合法 | 修正 .bkn 文件 | `update` |
| 门禁 7 失败 | platform 对象不足 | 补充建模对象 | `update`（需重新建模） |
| 预检失败 | 关系映射属性不存在 | 修正属性引用 | `update` |
| 静态检查 BLOCKED | 测试集质量不足 | 可能需调整模型结构 | `update`（调用 bkn-doctor） |
| 评分 < 60 | 整体质量不达标 | 按 score_breakdown 分发 | `update` 或 `feedback` |
| 评分 60-80 | 有风险项 | 确认是否接受或修复 | 用户选择 |

### bkn-doctor 调用条件

以下情况调用 `bkn-doctor` 辅助分析：
- 门禁 7 失败（对象数量不足）且用户有补充建模意向
- 静态检查 BLOCKED 且根因涉及模型结构
- 评分 < 60 且 `structure` 或 `relations` 维度得分 < 50%

调用时读取 `../bkn-doctor/SKILL.md`，传入 `problem_signal: validate_diagnosis`。

### 输出

```yaml
diagnosis:
  problems:
    - problem_id: ""
      category: gate | prepush | static_check | quality
      severity: critical | warning | info
      description: ""
      root_cause: ""
      affected_objects: []
      affected_files: []
  suggested_fixes:
    - fix_id: ""
      target_problem: ""
      action: ""
      target_skill: ""
      estimated_effort: low | medium | high
  routing_suggestion:
    primary: update | feedback | none
    reason: ""
    fallback: ""  # 主方案不可用时的备选
```

---

## 阶段八：报告

调用 `../bkn-report/SKILL.md`，生成**诊断报告**类型。

### 报告类型参数

向 `bkn-report` 传入 `report_type: diagnosis`，触发诊断报告生成。

### 诊断报告结构

```
1. 网络概览：名称、kn_id、来源（本地/pulled）、检查时间
2. 检查摘要：各阶段状态总览（pass/fail/skipped）
3. 门禁检查详情：每条门禁结果 + 不达标项明细
4. 预检详情：关系映射/动作绑定/concept_group 检查结果
5. 静态检查详情：schema_review 结果（如有）
6. 质量评分：综合分 + 各维度分 + 降级说明
7. 问题清单：按严重度排序的问题列表
8. 修复建议：每条建议 + 目标 skill + 预估工作量
9. 后续行动：推荐路由 + 用户可选路径
```

### 输出文件

- `{network_dir}/reports/DIAGNOSIS_REPORT.md`（必须）
- `{network_dir}/reports/DIAGNOSIS_REPORT.html`（可选，模板可用时）

---

## 阶段九：后续行动选项

展示诊断摘要后，询问用户后续行动：

```
诊断完成（质量评分：{score}/100，verdict：{verdict}）

后续行动选项：
  A. 进入修复流程（推荐）
     - 路由目标：{routing_suggestion.primary}
     - 修复范围：{suggested_fixes 数量} 个建议项
  B. 查看完整诊断报告
  C. 仅输出报告，暂不修复
  D. 保留临时目录（仅当 source: pulled_from_platform 时可选）
```

### 选项处理

| 选项 | 行为 |
|------|------|
| A | 路由到 `update` 或 `feedback` pipeline，携带诊断结果作为输入 |
| B | 展示完整报告内容，然后重新询问 |
| C | 结束流程，输出报告路径 |
| D | 保留临时目录（默认会清理），供后续手动操作 |

### 路由到 update pipeline

携带以下信息进入 `update` pipeline：
- `network_dir`（本地或临时目录）
- `validation_result`（完整诊断结果）
- `fix_scope`：用户选择的修复范围（全部/关键/手动选择）

update pipeline 在阶段一读取 `validation_result`，直接进入阶段三（建模调整），跳过变更分析阶段。

### 路由到 feedback pipeline

当诊断结果指向"规则缺失"或"语义描述不准"时，可路由到 `feedback` pipeline：
- 将诊断结果转为 `feedback_brief` 格式
- `source: manual`
- `issues` 从 diagnosis.problems 提取

---

## 输入格式

用户可通过以下方式触发：

### 直接触发

```
/bkn-creator 检查网络 {network_dir}
/bkn-creator 评估 {kn_id} 的质量
/bkn-creator 诊断 {网络名称}
```

### 从其他 pipeline 转入

- `create` pipeline 阶段八评审不达标 → 可选择转入 validate 进行独立诊断
- `update` pipeline 推送前检查 → 可选择转入 validate 进行完整诊断

---

## 输出格式

最终输出信封：

```yaml
trace_id: ""
intent: validate
from_skill: "bkn-creator"
to_skill: "user"
payload:
  result_status: ready | need_user_confirm
  next_action: "选择后续行动"
  network_info:
    network_id: ""
    network_name: ""
    source: local | pulled_from_platform
    network_dir: ""
  validation_summary:
    gate_check: {status, passed, failed}
    prepush_validation: {status, errors}
    static_check: {status, skip_reason}
    quality_score: 0
    verdict: pass | warn | fail
  diagnosis:
    problems_count: 0
    critical_count: 0
    suggested_fixes_count: 0
  report_path: ""
  routing_suggestion: {primary, reason}
  temp_dir_cleanup: true | false  # 是否清理临时目录
error: null
```

---

## Pipeline 状态持久化

在 `{network_dir}/pipeline_state.yaml` 记录：

```yaml
pipeline: validate
trace_id: ""
started_at: ""
current_stage: ""
source: local | pulled_from_platform
kn_id: ""  # 仅当 pulled_from_platform 时
bind_mode: full | deferred
plugin_availability:
  rules: available | unavailable
  test: available | unavailable
completed_stages:
  - stage: ""
    status: success | skipped | failed
    reason: ""
    timestamp: ""
    summary: ""
validation_result: {}  # 完整诊断结果
temp_dir: ""  # 仅当 pulled_from_platform 时记录临时目录路径
```

---

## 约束

1. **只读原则**：validate pipeline 不修改任何 `.bkn` 文件，不执行 push/delete/update
2. **不强制修复**：发现问题只记录和建议，不自动进入修复流程
3. **临时目录清理**：pull 到临时目录的场景，默认在流程结束后清理；用户可选择保留
4. **诊断不编造**：诊断结果必须基于实际检查数据，不推测未检查的内容
5. **插件降级透明**：插件不可用时明确标注跳过原因，不影响核心检查流程