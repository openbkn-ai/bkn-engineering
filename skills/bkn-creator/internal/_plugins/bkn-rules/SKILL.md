---
name: bkn-rules
description: 从 PRD/对话/建模中提取业务规则，生成可复用的业务规则 Skill。必执行。
---

# 业务规则提取

公约：`../../_shared/contract.md`

## 做什么

从整个建模过程的输入中提取业务规则，组织为结构化的可复用 Skill 产物。
**这是业务沉淀的核心环节，在 create pipeline 中为必执行步骤。**

## 执行模式

`bkn-rules` 支持两种模式，由调用方指定：

| 模式 | 时机 | 输入 | 行为 |
|------|------|------|------|
| `full`（默认） | 阶段五（首次生成） | PRD + 对话 + 提取结果 + 绑定结果 | 完整提取、分组、生成 Skill 文件 |
| `incremental` | 阶段六评审循环结束后 / 阶段七推送后 | 已有 Skill 文件 + 评审调整结果 | 读取已有 Skill，合并新增/变更的规则，更新文件 |

## 输入

- `mode`：`full` | `incremental`（默认 `full`）
- `network_id`：网络唯一标识（必填）
- `network_dir`：网络目录（`.bkn` 文件位于 `{network_dir}/bkn/`，Skill 文件位于 `{network_dir}/skills/`）
- `network_name`：网络名称
- `prd_content`：PRD / 产品文档（`incremental` 模式可选）
- `rule_extraction_check`：阶段一末尾提取的结构化规则
- `network_context`：网络名称、领域
- `conversation_context`（可选）：建模对话中的补充知识
- `binding_rules`（可选）：绑定过程中发现的映射规则
- `review_adjustment_plan`（可选，`incremental` 模式）：`bkn-review` 输出的 `adjustment_plan`（包含规则缺失的诊断和修复建议）

## 规则来源与优先级

| 来源 | 提取内容 | 冲突时 |
|------|---------|--------|
| PRD / 产品文档 | 需求约束、流程规则、判定条件 | 以用户对话为准 |
| 用户对话 | 补充的业务知识、行业惯例 | 最高优先级 |
| rule_extraction_check | 已结构化的主键/过滤/枚举/计算/层级规则 | — |
| 绑定分析 | 字段语义映射规则、数据质量约束 | — |
| 评审调整（`incremental`） | `review_adjustment_plan` 中标识的规则缺失项，经调整后产生的新规则 | 以调整后为准 |

## Skill 自检（生成后必执行）

Skill 文件生成后，必须执行以下自检，失败则标注警告但不阻断流程：

| 检查项 | 判定标准 |
|--------|---------|
| frontmatter 完整性 | 必须包含 `name`、`description`、`domain`、`network`、`network_id`、`version`、`lifecycle_state` |
| 文件结构合法性 | 必须包含规则总览表 + 至少一组详细规则 |
| 规则可追溯性 | 每条规则必须标注 `source`（prd / conversation / extraction / binding / adjustment） |
| 规则可执行性 | 高风险规则必须包含明确的判定条件或验证要点 |
| 反例覆盖 | 高风险规则至少有一条反例或验证要点 |

## Skill 拆分策略

根据规则数量和复杂度决定拆分粒度：

| 规则总数 | 策略 | 说明 |
|---------|------|------|
| ≤ 30 | 单 Skill | 一个网络一个规则 Skill，文件名 `{network_id}-rules.md` |
| 31–80 | 按领域分组 | 每个概念分组一个 Skill，文件名 `{network_id}-{group_slug}.md` |
| > 80 | 按对象拆分 | 每个核心对象一个 Skill，文件名 `{network_id}-{object_slug}-rules.md` |

**拆分策略变化时的处理**（`incremental` 模式）：
- 规则数量跨越阈值导致需要重新拆分/合并时，所有被替换的旧文件全部移入 `_archived/`
- 新生成的文件按命名规范创建，版本号延续当前版本并递增
- 合并场景（如 50→20 条）：多个分组合并为单 Skill，旧文件全部归档，生成新的 `{network_id}-rules.md`
- 拆分场景（如 20→50 条）：单 Skill 拆分为多个，旧文件归档，生成新的 `{network_id}-{group_slug}.md` 系列

### 命名规范

- Skill 文件名统一使用 `network_id` 标识，不使用网络名（中文/特殊字符可能导致路径不稳定）
- 单 Skill 场景：`{network_id}-rules.md`
- 多 Skill 场景（按领域分组）：`{network_id}-{group_slug}.md`
  - `group_slug` 为概念分组名的英文短标识（如 `sales-rules`、`inventory-rules`）
- 文件名与 skill name 保持一致（`name` frontmatter = 文件名去 `.md`）
- 每个 Skill 文件必须有 YAML frontmatter：
  ```yaml
  ---
  name: "{network_id}-rules"  # 单 Skill
  # name: "{network_id}-{group_slug}"  # 多 Skill 场景
  description: "..."
  domain: ""
  network: ""
  network_id: ""
  version: "1.0.0"
  lifecycle_state: active | draft | archived
  ---
  ```

**版本规则**：
- `full` 模式首次生成：`version: "1.0.0"`，`lifecycle_state: active`
- `incremental` 模式：读取当前版本的 `version`，递增 minor（如 `1.0.0` → `1.1.0`）
- 语义版本：`major.minor.patch`，minor 增量更新，major 结构性大改时递增

**归档机制**（`incremental` 模式）：
- 更新前将旧版本移入 `{network_dir}/skills/_archived/`
- 归档命名：`{原文件名}-v{旧版本号}-{YYYYMMDD-HHmmss}.md`
- 例：`mat-master-rules-v1.0.0-20260410-143022.md`
- 写入新版本后，`lifecycle_state: active`
- `full` 模式首次生成无需归档

### Skill 文件结构

每个 Skill 文件遵循标准 SKILL.md 结构：
1. Frontmatter（name, description, domain, network, network_id, version, lifecycle_state）
2. 规则总览表（rule_id / 规则名 / 类型 / 关联对象）
3. 分组详细规则
4. 依赖关系图

## 流程

1. 确认执行模式（`full` 或 `incremental`）
2. **`full` 模式**：
   1. 收集所有规则来源
   2. 提取规则（每条标注 source）
   3. 去重归并（同规则多来源合并，保留最完整版本）
   4. 按领域/对象分组
   5. 决定拆分策略（按上表）
   6. 高优先级规则补反例 + 验证要点
   7. 分析规则间依赖
3. **`incremental` 模式**：
   1. 读取 `{network_dir}/skills/` 下已有规则 Skill 文件（排除 `_archived/` 目录）
   2. **降级判定**：若未找到任何已有 Skill 文件，降级为 `full` 模式执行，后续步骤按 `full` 模式处理
   3. 结合 `review_adjustment_plan` 中标识的规则缺失项和当前建模状态（对象/关系/动作草案），识别新增/变更/废弃的规则
   4. **归档旧版本**：将已有 Skill 文件移入 `{network_dir}/skills/_archived/`，重命名为 `{原文件名}-v{旧版本号}-{YYYYMMDD-HHmmss}.md`。确保 `_archived/` 目录存在（不存在则创建）
   5. 合并到已有 Skill 文件中（保留原有 source 标注，新增标注 `source: adjustment`），递增版本号
   6. 重新执行拆分策略判定（规则数量变化可能导致需要重新拆分/合并，见上方拆分策略变化处理规则）
4. 生成 Skill 产物，落盘到 `{network_dir}/skills/`
5. **Skill 自检**（见上方自检表），结果输出到 `skill_self_check`
6. 锚定与分发：
   - `full` 模式：每个 Skill 对应一个 `bkn-anchor` 锚定对象，后续由 pipeline 委托 `bkn-distribute` 分发
   - `incremental` 模式：仅新增 Skill 需要锚定，已有 Skill 原地更新
     - **create pipeline 场景**（阶段六评审后/阶段七推送后）：**不触发重新分发**（避免覆盖用户手动修改），仅输出差异清单供 pipeline 决定是否重新分发
     - **feedback pipeline 场景**：修复完成后 pipeline 会显式委托 `bkn-distribute` 覆盖各平台副本

## 输出

```yaml
business_rules_skill:
  skill_name: "{network_id}-rules"
  mode: full | incremental
  domain: ""
  version: "1.0.0"
  lifecycle_state: active
  rules_summary: {total, by_type, added, modified, removed}
  archived_files: []  # incremental 模式：被归档的旧版本路径列表
  rule_groups:
    - group_name: ""
      rules:
        - rule_id: BR-001
          rule_name: ""
          rule_type: 主键规则 | 过滤规则 | 枚举规则 | 计算规则 | 层级规则 | 约束规则 | 映射规则
          description: ""
          source: prd | conversation | extraction | binding | adjustment
          related_objects: []
          validation_points: []
          counterexample: ""
  dependency_graph: {}
skill_self_check: {frontmatter: pass|fail, structure: pass|fail, traceability: pass|fail, executability: pass|fail, counterexamples: pass|fail, warnings: []}
skill_artifact_path: ""
```

## 规则存储边界

业务规则存在**两个落地位置**，职责不同：

| 位置 | 内容 | 写入方 | 消费方 |
|------|------|--------|--------|
| `.bkn` 文件 Description 列 | 属性级的简短语义说明（≤ 30 字） | `bkn-draft` | 平台 schema 展示、语义搜索命中 |
| `skills/{network_id}-rules.md` | 完整规则定义（条件、公式、反例、依赖） | `bkn-rules`（本 skill） | Agent 推理、Q&A 验证、业务沉淀 |
| `skills/_archived/` | 旧版本 Skill 归档（`incremental` 模式生成） | `bkn-rules` | 版本回滚、变更追溯 |

**原则**：
- Description 是"标签"——告诉平台这个字段大概什么意思
- Skill 是"手册"——告诉 Agent 完整的业务逻辑
- 两者的内容可以重叠，但 Skill 必须是 Description 的超集
- BKN 规范明确禁止在 `.bkn` 文件中添加规范外的额外 section，所以复杂规则**只能放 Skill**

**验证方式**：
- `qa_verify` 路径 B（语义搜索）验证的是 Description 的可达性
- `qa_verify` 路径 C（Decision Agent）验证的是 Skill 的推理完整性
- 两个路径都通过，才说明"规则可用"

## 约束

- 所有规则必须有来源标注，不编造不存在的规则
- 未提取到的规则标注 TBD，不伪造
- 冲突表述以用户对话确认为准，标注冲突
- Description 列只写精炼语义，完整规则写 Skill 文件
