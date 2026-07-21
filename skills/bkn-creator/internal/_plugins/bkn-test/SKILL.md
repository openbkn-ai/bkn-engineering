---
name: bkn-test
description: 生成测试集与验证用例。三种模式：schema_review / rules_verification / qa_verify。
---

# 测试与验证

公约：`../../_shared/contract.md`

## 做什么

根据 BKN 草案和业务规则生成可复用的测试集，或对已推送网络做 Q&A 验证。

## 三种模式

### 1. `schema_review`（推送前，静态检查，默认）

输入：对象/关系/动作草案 + 业务规则 + 绑定结果
输出：四类测试用例 + 覆盖率矩阵

**注意**：此为静态检查，不依赖真实数据。只验证 `.bkn` 文件的结构完整性、业务规则存在性、绑定关系完整性。

## 对象过滤规则

**`存储位置: local` 对象不参与 binding 测试**，local 对象没有数据视图绑定，binding 测试只统计 `platform` 对象。

| 测试类别 | local 对象处理 |
|---------|--------------|
| smoke | ✓ 参与（验证 schema 结构） |
| rules | ✓ 参与（规则可能引用 local 对象） |
| binding | ✗ 排除（local 对象无视图绑定） |
| risk | ✓ 参与（验证高风险动作定义） |

**binding 测试的最小数量**：
- binding_min = max(已绑定 platform 对象数, 3)
- 若 platform 对象数为 0 → binding 测试跳过，不计入 BLOCKED

## rules 插件不可用时的处理

**当 `plugin_availability.rules == unavailable` 时（阶段五跳过）**：

| 测试类别 | 处理方式 |
|---------|---------|
| smoke | 正常生成 |
| rules | 跳过（无业务规则 Skill 文件，无法生成规则类测试） |
| binding | 正常生成 |
| risk | 正常生成 |

此时：
- rules_min = 0（不计入 BLOCKED）
- bkn-review 评分中"规则覆盖率"维度降级（Skill 文件质量子项权重降为 0，剩余权重等比放大）
- 输出中标注：`rules: skipped(no_skill_file)`

## 数量硬约束（BLOCKED 机制）

**本节约束适用于 `schema_review` 模式**。`rules_verification` 模式有独立的数量约束（见 mode 2 说明）。

**生成测试集后必须自检数量，不达标则 BLOCKED**：

```
最小用例数计算：
- smoke_min = 对象数 + 关系数
- rules_min = max(规则数 × 2, 10)  # 每条规则至少1正例，关键规则额外1反例，总数不低于10
- binding_min = 3
- risk_min = 高风险动作数 × 2

总最小用例数 = smoke_min + rules_min + binding_min + risk_min
```

**执行逻辑**：
1. 统计生成的各类用例数量
2. 与最小规范对比
3. 任一类别不达标 → **BLOCKED**，输出：
   ```
   测试集数量不达标，BLOCKED：
   - smoke: 需至少 {smoke_min} 条，实际 {actual} 条
   - rules: 需至少 {rules_min} 条，实际 {actual} 条
   - binding: 需至少 {binding_min} 条，实际 {actual} 条
   - risk: 需至少 {risk_min} 条，实际 {actual} 条
   
   必须补充至达标后方可继续。
   ```
4. 用户确认补充后，重新生成缺失类别用例
5. 全部达标 → 输出完成回执

## 复杂度约束

### smoke 测试必须包含（每条）

| 检查项 | 说明 | 必须验证 |
|--------|------|---------|
| 对象存在性 | 对象类型在 schema 中存在 | ✓ |
| 属性完整性 | 至少验证 3 个核心属性非空 | ✓ |
| 主键有效性 | primary_key 字段存在且非空 | ✓ |
| 关系连通性 | 关系的 source/target 对象均存在 | ✓ |

**smoke 用例模板**：

```yaml
case_id: smoke_obj_{object_id}_001
title: 验证 {object_name} 对象结构完整
level: smoke
target: {object_id}
steps:
  - step: 查询 {object_id} 对象 schema
    action: openbkn bkn object-type get {object_id}
  - step: 检查属性列表包含核心属性
    check: properties contains [{核心属性1}, {核心属性2}, {核心属性3}]
  - step: 检查 primary_key 非空
    check: primary_key != null && primary_key != ""
  - step: 检查 display_key 非空
    check: display_key != null && display_key != ""
expected_result:
  - 对象存在
  - 属性数量 >= {最小属性数}
  - 核心属性: {列出3个核心属性} 均存在
  - primary_key: {主键字段名}
  - display_key: {显示字段名}
```

```yaml
case_id: smoke_rel_{relation_id}_001
title: 验证 {relation_name} 关系连通性
level: smoke
target: {relation_id}
steps:
  - step: 查询 {relation_id} 关系 schema
    action: openbkn bkn relation-type get {relation_id}
  - step: 检查 source 对象存在
    check: source_object_id in {所有对象ID列表}
  - step: 检查 target 对象存在
    check: target_object_id in {所有对象ID列表}
  - step: 检查 mapping 非空
    check: mapping_rules != null
expected_result:
  - 关系存在
  - source: {source_object_id} 存在
  - target: {target_object_id} 存在
  - mapping: 有映射规则
```

### rules 测试必须包含（每条）

| 检查项 | 说明 | 必须验证 |
|--------|------|---------|
| 触发条件 | 规则的触发条件明确 | ✓ |
| 预期行为 | 满足条件时的预期结果 | ✓ |
| 正例验证 | 满足条件的场景验证 | ✓ |
| 反例验证（关键规则） | 违反条件的拒绝验证 | 仅高/关键规则 |

**rules 用例模板（正例）**：

```yaml
case_id: rules_{rule_id}_positive_001
title: 验证规则 {rule_name} 正例触发
level: rules
target: {rule_id}
rule_type: {低|中|高/关键}
steps:
  - step: 识别规则触发条件
    action: 解析规则 {rule_id} 的触发条件
  - step: 构造满足条件的测试输入
    input: {具体输入数据或场景描述}
  - step: 触发规则执行
    action: {执行方式}
  - step: 验证预期行为
    check: actual_result matches expected_behavior
expected_result:
  - 触发条件: {列出具体条件}
  - 预期行为: {列出预期结果}
  - 关键词: [{expected_keyword_1}, {expected_keyword_2}]
  - 规则来源: {rule_source_location}
```

**rules 用例模板（反例 - 仅高/关键规则）**：

```yaml
case_id: rules_{rule_id}_negative_001
title: 验证规则 {rule_name} 反例拒绝
level: rules
target: {rule_id}
rule_type: 高/关键
is_counterexample: true
steps:
  - step: 构造违反条件的测试输入
    input: {违反条件的具体输入}
  - step: 尝试触发规则
    action: {执行方式}
  - step: 验证拒绝或报错
    check: result is rejection or error
expected_result:
  - 违反条件: {列出违反的具体条件}
  - 预期拒绝: 系统应拒绝或报错
  - 不应出现: [{expected_absent_keyword_1}, {expected_absent_keyword_2}]
  - 反例类型: {边界越界|条件缺失|逻辑冲突}
```

### binding 测试必须包含（每条）

| 检查项 | 说明 | 必须验证 |
|--------|------|---------|
| 绑定率 | platform 对象绑定 view 的比例 | ✓ |
| 映射覆盖率 | 属性映射完成的比例 | ✓ |
| blocked 项 | 绑定失败的项及原因 | ✓ |

**binding 用例模板**：

```yaml
case_id: binding_rate_001
title: 验证对象视图绑定率
level: binding
target: all_platform_objects
steps:
  - step: 统计 platform 对象总数
    action: count objects where 存储位置 == platform
  - step: 统计已绑定对象数
    action: count objects where data_source.view_id != null
  - step: 计算绑定率
    formula: bind_rate = bound_count / platform_count
expected_result:
  - platform对象数: {N}
  - 已绑定数: {M}
  - 绑定率: {M/N}%
  - 最低要求: >= 80%（若低于则需说明原因）

---

case_id: binding_coverage_001
title: 验证属性映射覆盖率
level: binding
target: all_bound_objects
steps:
  - step: 统计所有绑定对象的属性总数
    action: sum properties across bound objects
  - step: 统计已映射属性数
    action: count properties where mapped_field != "-" and mapped_field != null
  - step: 计算映射覆盖率
    formula: coverage = mapped_count / property_count
expected_result:
  - 属性总数: {N}
  - 已映射数: {M}
  - 映射覆盖率: {M/N}%
  - 未映射属性: [{列出未映射属性}]
```

### risk 测试必须包含（每条）

| 检查项 | 说明 | 必须验证 |
|--------|------|---------|
| 动作存在性 | 高风险动作在 schema 中存在 | ✓ |
| 参数完整性 | 动作参数绑定完整 | ✓ |
| 正例：成功执行 | 正常条件下的执行验证 | ✓ |
| 反例：拒绝执行 | 异常条件下的拒绝验证 | ✓ |

**risk 用例模板**：

```yaml
case_id: risk_{action_id}_positive_001
title: 验证高风险动作 {action_name} 正例执行
level: risk
target: {action_id}
risk_level: 高
steps:
  - step: 查询动作 schema
    action: openbkn bkn action-type get {action_id}
  - step: 检查参数绑定完整性
    check: parameter_binding all have valid source/binding
  - step: 构造合法执行条件
    input: {满足前置条件的输入}
  - step: 执行动作（模拟）
    action: {执行方式}
expected_result:
  - 动作存在
  - 参数绑定完整
  - 前置条件满足时可执行
  - 关键词: [{expected_keyword_1}]

---

case_id: risk_{action_id}_negative_001
title: 验证高风险动作 {action_name} 反例拒绝
level: risk
target: {action_id}
risk_level: 高
is_counterexample: true
steps:
  - step: 构造非法执行条件
    input: {违反前置条件的输入}
  - step: 尝试执行动作
    action: {执行方式}
  - step: 验证拒绝或报错
    check: result is rejection or error
expected_result:
  - 前置条件不满足时应拒绝
  - 不应出现: [{expected_absent_keyword}]
  - 拒绝类型: {权限不足|前置条件缺失|参数无效}
```

## 规则风险分级说明

| 等级 | 标签 | 含义 | 示例 | 反例要求 |
|------|------|------|------|---------|
| 低 | `低` | 只读或对单条记录的非破坏性操作 | 查询、生成报表 | 无 |
| 中 | `中` | 影响多条记录或触发下游流程 | 批量状态变更、MRP 计算 | 无 |
| 高/关键 | `高/关键` | 删除、跨系统推送、审批流触发、资金相关 | 删除网络、推送生产计划、关键决策规则 | **必须** |

**关键规则**：高风险级别的业务规则，涉及删除、审批、资金或跨系统推送。这类规则必须补充反例测试。

---

### 2. `rules_verification`（推送前，语义检查）

输入：锚定在网络中的业务规则 Skill
输出：规则语义测试集（正反例 + 溯源检查）

**与 schema_review 的关系**：
- `schema_review` 中的 rules 类用例只做"规则存在性检查"（规则 Skill 文件存在、rule_id 可追溯、规则数量与提取清单一致）
- `rules_verification` 做"规则语义检查"（触发条件是否可判定、预期行为是否明确、正反例是否覆盖边界场景）
- 两者互补，**rules_verification 在阶段六评审循环退出后、bkn-rules(incremental) 之后执行**，验证最终版本的业务规则 Skill

**检查项**：

| 检查项 | 说明 | 必须验证 |
|--------|------|---------|
| 触发条件可判定性 | 规则的触发条件是否包含可量化的判断标准 | ✓ |
| 预期行为明确性 | 满足/违反条件时的预期结果是否无歧义 | ✓ |
| 正例覆盖 | 每条规则至少有 1 个正例场景 | ✓ |
| 反例覆盖（关键规则） | 高/关键规则至少有 1 个反例场景 | 仅高/关键规则 |
| 溯源完整性 | 规则来源标注可追溯到原始文档段落 | ✓ |

**rules_verification 用例模板（正例）**：

```yaml
case_id: rv_{rule_id}_positive_001
title: 验证规则 {rule_name} 语义正例
level: rules_verification
target: {rule_id}
rule_type: {低|中|高/关键}
steps:
  - step: 解析规则触发条件的可判定性
    action: 检查 {rule_id} 的触发条件是否包含量化标准
  - step: 构造满足条件的具体场景
    input: {具体输入数据，需明确数值或状态}
  - step: 验证预期行为
    check: actual_result matches {明确的预期结果}
  - step: 验证溯源
    check: rule_source traces back to {原始文档/章节}
expected_result:
  - 触发条件: {具体可判定的条件}
  - 预期行为: {无歧义的结果描述}
  - 关键词: [{expected_keyword_1}, {expected_keyword_2}]
  - 规则来源: {rule_source_location}
  - 溯源判定: 可追溯 / 不可追溯（需标注原因）
```

**rules_verification 用例模板（反例 - 仅高/关键规则）**：

```yaml
case_id: rv_{rule_id}_negative_001
title: 验证规则 {rule_name} 语义反例
level: rules_verification
target: {rule_id}
rule_type: 高/关键
is_counterexample: true
steps:
  - step: 构造违反规则条件的边界场景
    input: {违反条件的具体输入，需说明违反的具体维度}
  - step: 验证系统拒绝或报错
    check: result is rejection or error with specific message
expected_result:
  - 违反条件: {具体违反的维度}
  - 预期拒绝: 系统应拒绝并给出明确提示
  - 不应出现: [{expected_absent_keyword}]
  - 反例类型: {边界越界|条件缺失|逻辑冲突}
```

**数量约束**：
- rv_min = 规则数 + 关键规则数（每条规则至少 1 正例，关键规则额外 1 反例）
- 规则数为 0 时，rules_verification 跳过（不计入 BLOCKED）
- 规则数 < 5 时，rv_min 不设下限（即不要求至少 10 条，因为这是语义检查而非结构检查）
- 规则数 >= 5 时，rv_min = max(规则数 + 关键规则数, 8)

**与 BLOCKED 机制的关系**：
- rules_verification 的 BLOCKED 独立于 schema_review 的全局 BLOCKED
- 若不达标，输出：
  ```
  rules_verification 语义检查不达标：
  - 需至少 {rv_min} 条，实际 {actual} 条
  - 缺失类型: {正例/反例/溯源}
  ```
- 用户可以确认"接受现状"后继续，不强制 BLOCKED 整个 pipeline（因为这是语义深度检查，不阻塞结构推送）

---

### 3. `qa_verify`（推送后，实际验证）

输入：业务规则 Skill + 已推送网络
输出：验收引导卡 + Q&A 验证结果

#### L3 规则验证（P1-7）

**L3 级问题必须基于 `skills/` 目录下的业务规则 Skill 文件生成**，不可凭空编造：
1. 读取 `{network_dir}/skills/` 下所有业务规则 Skill 文件
2. 从每条高风险规则中提取验证问题，确保 `rule_id` 可追溯到 Skill 文件中的具体规则
3. 若 Skill 文件不存在或为空，L3 级测试标记为 `skipped`，并在报告中注明"业务规则 Skill 不可用"
4. 若存在 `bkn-rules` 的 `skill_self_check` 且有 fail 项，在引导卡中提示用户注意

#### 问题分级与数量约束

| 级别 | 验证目标 | 最小数量 | 示例 |
|------|---------|---------|------|
| L1 结构 | 对象/属性是否存在 | 至少 2 题 | "预测单有哪些属性？" |
| L2 关系 | 关系连通性 | 至少 2 题 | "MRP 和预测单什么关系？" |
| L3 规则 | 业务规则是否可回答 | 关键规则 × 1 | "BOM 用量怎么计算？" |
| L4 推理 | 跨实体推导 | 至少 2 题 | "缺料时影响哪些生产计划？" |

**L3 级数量计算**：
- L3_min = 关键规则数（高/关键级别规则，每条至少1题）
- 不设最低下限：若关键规则数为 0，则 L3_min = 0
  - 业务本身简单时无需硬凑问题，验证价值低
  - 保证总量：L1 + L2 + L4 >= 6 即可
- 若用户主动要求"多做规则验证"，可从普通规则中追加

#### Agent 配置数据准备

在生成 Agent 配置前，先从以下来源收集所需参数：

| 占位符 | 数据来源 | 获取方式 |
|--------|---------|---------|
| `{network_name}` | `pipeline_state.yaml` 或 `.bkn` 文件 Network Overview | 读取 `network_name` 字段 |
| `{model_id}` / `{model_name}` | 平台可用模型列表 | 执行 `openbkn llm list`（委托 bkn-openbkn）获取平台模型；优先使用默认模型，否则询问用户 |
| `{kn_id}` | 阶段七推送返回的网络 ID | 从 `pipeline_state.yaml.kn_id` 读取 |
| `{all_object_type_ids}` | `{network_dir}/bkn/` 下所有 object_type 的 id 字段 | 扫描 `.bkn` 文件，提取 `Object Type:` 行的 id 值 |

**数据准备步骤**：
1. 读取 `pipeline_state.yaml` 获取 `kn_id`、`network_name`
2. 扫描 `{network_dir}/bkn/*.bkn`，提取所有 object_type ID 列表
3. 执行 `openbkn llm list`（委托 bkn-openbkn）获取平台模型列表；若无可用模型，提示用户选择
4. 将上述参数填入 Agent 配置模板

#### Agent 配置模板

委托验证需要 Agent 时，使用以下配置结构（基于 openbkn-sdk e2e 验证）：

```json
{
  "input": {"fields": [{"name": "user_input", "type": "string", "desc": ""}]},
  "output": {"default_format": "markdown"},
  "system_prompt": "你是{network_name}的Decision Agent。基于知识网络中的对象、关系和业务规则回答用户问题。",
  "llms": [{
    "is_default": true,
    "llm_config": {
      "id": "{model_id}",
      "name": "{model_name}",
      "model_type": "llm",
      "temperature": 0.7,
      "top_p": 0.8,
      "top_k": 1,
      "frequency_penalty": 0,
      "presence_penalty": 0,
      "max_tokens": 4096
    }
  }],
  "data_source": {
    "kg": [{"kg_id": "{kn_id}", "fields": ["{all_object_type_ids}"]}],
    "advanced_config": {
      "kg": {
        "text_match_entity_nums": 60,
        "vector_match_entity_nums": 60,
        "graph_rag_topk": 25,
        "long_text_length": 256,
        "reranker_sim_threshold": -5.5,
        "retrieval_max_length": 20480
      }
    }
  }
}
```

**关键要点**：
- `data_source` 使用 `kg`（非 `knowledge_network`），这是 agent-factory API 的遗留字段名
- `fields` 填入该网络所有 object_type 的 ID 列表
- `llm_config` 必须包含 `temperature`/`top_p`/`top_k`，缺失会导致 `FormatError`
- 不使用 dolphin 模式（无需设置 `is_dolphin_mode`）

#### 交互流程（两阶段）

**第一阶段：生成验收引导卡（必执行）**

生成分级问题后，以引导卡形式呈现给用户，并询问验证方式：

```
知识网络「{network_name}」已推送完成

以下是根据业务规则生成的验收问题：

【L1 结构】（至少 {L1_min} 题）
  - {question}
  - {question}

【L2 关系】（至少 {L2_min} 题）
  - {question}
  - {question}

【L3 规则】（至少 {L3_min} 题）关键规则必问
  - {question}（关联规则：{rule_id}）
  - {question}（关联规则：{rule_id}）

【L4 推理】（至少 {L4_min} 题）
  - {question}
  - {question}

问题总数：{total}，达标：{达标 or 不达标}

请选择验证方式：
  A. 我自己去 DIP 验证，完成后把结果反馈给你
  B. 快速验证（L1+L2，使用 context-loader，无需 Agent）
  C. 完整验证（L1-L4，需要 Agent，尝试端到端验证）
```

**第二阶段：按用户选择执行**

| 选择 | 执行路径 |
|------|---------|
| A（用户自验） | 等待用户反馈 → 用户描述 Agent 回答 → 执行判定 → 输出结果 |
| B（快速验证） | L1+L2 自动验证（见下方），L3/L4 输出为"建议用户自行验证" |
| C（完整验证） | L1-L4 自动验证（见下方），Agent 不可用时自动降级为 B |

#### 验证模式

| 模式 | 覆盖级别 | 工具 | 适用场景 |
|------|---------|------|---------|
| 快速验证（B） | L1 + L2 | `openbkn context-loader` + `openbkn bkn search` | 无 Agent、schema_only、快速迭代 |
| 完整验证（C） | L1-L4 | 上述 + `openbkn agent chat` | 有关联 Agent、需验证推理能力 |

**快速验证流程**：
1. 前置：`openbkn context-loader config set --kn-id <network_id>`
2. L1 结构：`openbkn context-loader kn-schema-search "<question>"` → 检查对象/属性命中
3. L2 关系：`openbkn context-loader kn-schema-search "<question>"` → 检查关系连通性 + `openbkn bkn search` → 检查语义可达性

**完整验证流程**：
1. L1 + L2 同快速验证
2. L3 规则：`openbkn bkn search <kn_id> "<question>"` → 检查规则描述命中
3. L4 推理：`openbkn agent chat <agent_id>` → 端到端回答验证（需用户确认判定）

**降级规则**：
- 完整验证时若 `env_capability_matrix.agent_factory == unavailable`，自动降级为快速验证
- `openbkn agent list` 找不到关联 Agent 或调用失败 → 降级为快速验证，不重试
- L4 推理题无论走哪条路，最终判定均需用户确认

#### 反例测试

对高风险关键规则，额外生成反例问题：
- 违反约束条件的查询（如"BOM 用量为负数时如何处理？"）
- 超出业务边界的查询（如"预测单能否直接生成采购订单？"）
- 预期结果：网络应**不匹配**或**明确拒绝**
- 反例通过条件：实际回答不包含 `expected_absent_keywords` 中的任何词

```yaml
qa_results:
  - question: ""
    rule_id: ""
    level: L1 | L2 | L3 | L4
    is_counterexample: false
    expected_keywords: []
    expected_absent_keywords: []
    verify_mode: user_self | quick | full
    tool_used: context-loader | bkn-search | agent-chat
    actual_answer: ""
    match: true | false
    data_status: has_data | schema_only | not_found
    l4_user_confirmed: null | true | false
    final_verdict: pass | fail | skipped
```

## 通用输出

```yaml
testset_summary: 
  scope: {network_name}
  focus: {model_review|rules_verification|qa_verify}
  total_cases: {N}
  smoke_cases: {N}
  rules_cases: {N}
  binding_cases: {N}
  risk_cases: {N}
  passed: {N}
  failed: {N}
  blocked: {N}
coverage_matrix:
  objects: {covered}/{total}
  relations: {covered}/{total}
  actions: {covered}/{total}
  rules: {covered}/{total}
  bindings: {covered}/{total}
gaps: []
testset_path: ""  # 落盘路径，必须向用户展示
quantity_check:
  smoke: {actual}/{min} {达标|不达标}
  rules: {actual}/{min} {达标|不达标}
  binding: {actual}/{min} {达标|不达标}
  risk: {actual}/{min} {达标|不达标}
  overall: {达标|BLOCKED}
```

## 测试集持久化

`schema_review` 和 `rules_verification` 模式必须将测试集落盘：

- 路径：`{network_dir}/tests/testset.yaml`
- 内容：完整测试用例列表（完整模板，含 case_id / title / level / target / steps / expected_result）
- 用途：巡检任务（patrol）定期回放此文件，验证网络是否退化
- `qa_verify` 模式不持久化测试集（验证是实时执行的，不生成可回放用例）

## 完成回执

测试集落盘且数量达标后，按回显模板输出：

```
### 测试集已生成（完成）
说明：
- 落盘路径：{network_dir}/tests/testset.yaml
- 用例数：{total_cases}
- 数量检查：全部达标 ✓
  - smoke: {actual}/{min} ✓
  - rules: {actual}/{min} ✓
  - binding: {actual}/{min} ✓
  - risk: {actual}/{min} ✓
下一步：执行 bkn-test --mode qa_verify 或进入推送流程
```

## 约束

- **数量不达标 BLOCKED**：必须补充至达标后方可继续
- **复杂度必须达标**：每类测试必须包含指定检查项
- **local 对象过滤**：binding 测试排除 local 对象；smoke/risk/rules 包含 local 对象
- **rules 插件不可用时**：rules 类测试跳过，不计入 BLOCKED
- 不为不存在的对象虚构测试
- 高风险动作必须有风险校验测试
- 不将待确认假设写成确定性预期
- L3 级问题必须基于 Skill 文件生成，不可凭空编造