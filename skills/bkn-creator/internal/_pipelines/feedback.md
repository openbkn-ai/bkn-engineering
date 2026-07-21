# 反馈巡检流程（Feedback Review）

从定时任务传入的使用反馈摘要出发，识别知识网络的改进项并驱动修复。

## Skill 路径索引

| skill | 路径（相对本文件） | 类型 |
|-------|-------------------|------|
| bkn-doctor | `../bkn-doctor/SKILL.md` | 核心 |
| bkn-map | `../bkn-map/SKILL.md` | 核心 |
| bkn-bind | `../bkn-bind/SKILL.md` | 核心 |
| bkn-relation-bind | `../bkn-relation-bind/SKILL.md` | 核心 | 关系类型判定 + 中间视图绑定 |
| bkn-backfill | `../bkn-backfill/SKILL.md` | 核心 |
| bkn-rules | `../_plugins/bkn-rules/SKILL.md` | **插件** |
| bkn-anchor | `../_plugins/bkn-anchor/SKILL.md` | **插件** |
| bkn-distribute | `../_plugins/bkn-distribute/SKILL.md` | **插件** |
| bkn-test | `../_plugins/bkn-test/SKILL.md` | **插件** |
| bkn-review | `../bkn-review/SKILL.md` | 核心 |
| bkn-report | `../bkn-report/SKILL.md` | 核心 |
| bkn-openbkn | `../bkn-openbkn/SKILL.md` | 核心 | OpenBKN CLI 操作（内化） |
| 公约 | `../_shared/contract.md` | — |
| 巡检标准 | `../references/patrol-standard.md` | — |
| 插件检测 | `../_shared/plugin-check.md` | — |

> **插件类型**：路径指向 `_plugins/` 目录，开源版可能不存在。pipeline 调用前需检测 `plugin_availability`，不可用时执行降级分支。

## 阶段总览

```
输入（feedback_brief / patrol_result）
  ↓
阶段一：解析 & 分类（如为 patrol_result，先提取 issues）
  ↓
阶段二：用户确认修复范围
  ↓
阶段三：按类型分发修复
  ├── 知识缺口    → bkn-doctor + [bkn-rules(incremental) + bkn-anchor + bkn-distribute]（插件，可跳过）
  ├── 语义描述不准 → bkn-map / bkn-backfill
  ├── 绑定问题    → bkn-bind + bkn-relation-bind + bkn-map + bkn-backfill
  ├── 关系绑定问题 → bkn-relation-bind + bkn-backfill
  └── 规则缺失    → [bkn-rules(incremental) + bkn-anchor + bkn-distribute]（插件，可跳过）
  ↓
阶段四：[BKN 模型审查 + 评审]（test 插件，可跳过）
  ↓
阶段五：推送 + 报告（门禁自检 → 预检修复循环 → 推送 → [qa_verify]（插件，可跳过）→ 报告）
```

> `[]` 标记的为插件阶段，`plugin_availability` 检测不可用时自动跳过。

## 输入格式

### 格式一：feedback_brief（直接传入）

由定时任务（仅限有 `bkn-test` 插件的进阶版）或用户手动传入，格式如下：

```yaml
feedback_brief:
  network_id: ""
  network_name: ""
  period: "YYYY-MM-DD ~ YYYY-MM-DD"
  source: scheduled_task | manual  # 定时任务触发（进阶版）or 用户手动提交
  issues:
    - issue_id: "F-001"
      signal: "无法回答" | "检索得分低" | "用户追问" | "用户纠正" | "会话极短"
      question: "用户原始提问"
      frequency: 1          # 同类问题出现次数
      affected_objects: []  # 可选，巡检时能识别出的关联对象类
      suggestion: ""        # 巡检任务的初步改进建议（可为空）
```

**开源版说明**：当 `plugin_availability.test == unavailable` 时，定时任务不会创建，`source: scheduled_task` 的输入不会出现。feedback pipeline 仅支持 `source: manual`（用户手动提交反馈）。
```

### 格式二：patrol_result（仅限进阶版）

由巡检任务（基于 `../references/patrol-standard.md`，需要 `bkn-test` 插件）生成，pipeline 自动提取 `issues` 转为 `feedback_brief`：

```yaml
patrol_result:
  network_id: ""
  network_name: ""
  period: "YYYY-MM-DD ~ YYYY-MM-DD"
  test_regression: ...  # 测试集回归结果
  qa_accuracy: ...      # 问答准确性指标
  trend: ...            # 趋势对比
  issues: ...           # 问题清单（直接转为 feedback_brief.issues）
```

**插件依赖**：此格式仅当 `plugin_availability.test == available` 时才会出现。开源版删除 `_plugins/` 目录后，巡检任务不创建，不会有 `patrol_result` 输入。
```

**转换规则**：当输入为 `patrol_result` 时，pipeline 自动将 `issues` 转为 `feedback_brief` 格式（`issue_id` 从 `P-xxx` 映射为 `F-xxx`），保留 `source: scheduled_task`。

## 阶段一：解析 & 分类

### 输入类型判断

先判断输入类型：
- **`feedback_brief`**：直接使用 `feedback_brief.issues`
- **`patrol_result`**：提取 `patrol_result.issues`，将 `issue_id` 从 `P-xxx` 映射为 `F-xxx`，构造 `feedback_brief` 信封

### 分类规则

读取 `feedback_brief`，将 issues 按修复类型分类：

| 修复类型 | 判定条件 | 分发目标 |
|---------|---------|---------|
| `knowledge_gap` | 信号为"无法回答"，或问题涉及网络中不存在的对象/规则 | bkn-doctor + bkn-rules(incremental) |
| `semantic_poor` | 信号为"检索得分低"，或问题能对应到对象但语义描述模糊 | bkn-map + bkn-backfill |
| `binding_issue` | 信号为"检索得分低"且数据字段映射不完整 | bkn-bind + bkn-relation-bind + bkn-map + bkn-backfill |
| `relation_binding_issue` | 问题涉及关系查询失败，或关系类型/中间视图配置问题 | bkn-relation-bind + bkn-backfill |
| `rule_missing` | 信号为"用户纠正"或"用户追问"，问题指向业务规则缺失 | bkn-rules(incremental) + bkn-anchor |
| `ambiguous` | 信号为"会话极短"，无法判断根因 | 标记人工确认，不自动分发 |

分类后输出摘要，不展示逐条 issue 明细，只展示各类型的问题数量：

```
反馈摘要（{network_name}，{period}）

共 {total} 个问题，分类如下：
  · 知识缺口（knowledge_gap）：{n} 个
  · 语义描述不准（semantic_poor）：{n} 个
  · 绑定问题（binding_issue）：{n} 个
  · 关系绑定问题（relation_binding_issue）：{n} 个
  · 规则缺失（rule_missing）：{n} 个
  · 待人工判断（ambiguous）：{n} 个

建议修复 {actionable} 个，跳过 {skip} 个（ambiguous 类）。
```

## 阶段二：用户确认修复范围

展示分类摘要后，询问用户：

```
请确认修复范围：
  A. 全部修复（跳过 ambiguous 类）
  B. 仅修复高频问题（frequency >= 3）
  C. 手动选择修复项
  D. 查看明细后再决定
```

- 选 D 则展示 issue 逐条明细，再重新询问
- 确认后锁定本次修复的 issue 列表，不可中途扩大范围
- `ambiguous` 类默认跳过，用户可手动选入

## 阶段三：分发修复

按分类逐组执行，每组完成后提示用户再进入下一组。

---

### knowledge_gap → bkn-doctor + [bkn-rules + bkn-anchor + bkn-distribute]（插件阶段）

**前置检测**：读取 `pipeline_state.yaml.plugin_availability.rules`

1. 将问题描述作为"待补充知识点"输入 `../bkn-doctor/SKILL.md`，触发建模补充
2. **若 `plugin_availability.rules == available`**：
   - 补充完成后，由 `../_plugins/bkn-rules/SKILL.md`（incremental）检查是否需要新增业务规则
   - 新规则由 `../_plugins/bkn-anchor/SKILL.md` 锚定到网络
   - 由 bkn-rules 的归档机制自动处理旧版本归档（旧版本移入 `{network_dir}/skills/_archived/`，新版本写入 `skills/` 根目录）
   - 分发更新后的 Skill 到已安装平台：委托 `../_plugins/bkn-distribute/SKILL.md`，直接覆盖各平台目录下副本
   - 若之前已发布到 OpenClaw 且 `env_capability_matrix.skill_module == available`：重新发布覆盖旧版本
3. **若 `plugin_availability.rules == unavailable`**：
   - 跳过规则相关步骤，在 `pipeline_state.yaml.completed_stages` 记录 `stage3_rules: skipped(plugin_unavailable)`
   - 仅完成 bkn-doctor 建模补充

---

### semantic_poor → bkn-map + bkn-backfill

1. 将涉及对象传入 `../bkn-map/SKILL.md`，重新审查属性语义描述
2. 修订后由 `../bkn-backfill/SKILL.md` 回填 `.bkn` 文件

---

### binding_issue → bkn-bind + bkn-relation-bind + bkn-map + bkn-backfill

1. 对绑定不完整的对象重新执行 `../bkn-bind/SKILL.md`
2. 对涉及的关系重新执行 `../bkn-relation-bind/SKILL.md`（判定关系类型 + 绑定中间视图）
3. 更新映射后由 `../bkn-map/SKILL.md` + `../bkn-backfill/SKILL.md` 完成回填

---

### relation_binding_issue → bkn-relation-bind + bkn-backfill

1. 将问题涉及的关系传入 `../bkn-relation-bind/SKILL.md`
2. 重新判定关系类型（direct / data_view）
3. 如需中间视图，推荐候选并用户确认
4. 确认后由 `../bkn-backfill/SKILL.md` 回填关系类 Mapping View + Source/Target Mapping

---

### rule_missing → [bkn-rules + bkn-anchor + bkn-distribute]（插件阶段）

**前置检测**：读取 `pipeline_state.yaml.plugin_availability.rules`

| plugin_availability.rules | 执行路径 |
|---------------------------|---------|
| `available` | 执行完整流程（见下方） |
| `unavailable` | 跳过本修复项，标记为 `skipped(plugin_unavailable)`，提示用户"规则插件不可用，无法自动补充业务规则" |

**当 `plugin_availability.rules == available` 时执行**：

1. 将用户纠正/追问的问题作为规则线索输入 `../_plugins/bkn-rules/SKILL.md`（incremental）
2. 提取/补充规则后，展示规则摘要，用户确认
3. 确认通过后由 `../_plugins/bkn-anchor/SKILL.md` 锚定
4. 由 bkn-rules 的归档机制自动处理旧版本归档（旧版本移入 `{network_dir}/skills/_archived/`，新版本写入 `skills/` 根目录）
5. 分发更新后的 Skill 到已安装平台：委托 `../_plugins/bkn-distribute/SKILL.md`，直接覆盖各平台目录下副本
6. 若之前已发布到 OpenClaw 且 `env_capability_matrix.skill_module == available`：重新发布覆盖旧版本

---

## 阶段四：BKN 模型审查 + 评审（插件阶段）

**前置检测**：读取 `pipeline_state.yaml.plugin_availability.test`

| plugin_availability.test | 执行路径 |
|--------------------------|---------|
| `available` | 执行完整评审流程（见下方） |
| `unavailable` | 跳过本阶段，在 `pipeline_state.yaml.completed_stages` 记录 `stage4_review: skipped(plugin_unavailable)`，直接进入推送门禁 |

**当 `plugin_availability.test == available` 时执行**：

复用 create pipeline 阶段八逻辑：

| 步骤 | 读取 | 说明 |
|------|------|------|
| 生成测试集 | `../_plugins/bkn-test/SKILL.md`（model_review 模式） | 覆盖本次修复涉及的对象/规则/绑定 |
| 评审 | `../bkn-review/SKILL.md` | 质量评分；>= 80 通过 |
| 不达标 | 回调对应修复 skill | 最多 2 轮，超出则标记人工介入 |

---

## 阶段五：推送 + 报告

推送前必须执行门禁自检和预检修复循环，遵循 `../_shared/contract.md` 推送重试规则。

### 5.1 门禁自检

扫描 `{network_dir}/bkn/` 目录，检查以下硬前置条件（与 create pipeline 阶段七相同的 7 条硬前置，根据 `bind_mode` 和 `存储位置` 应用豁免）：

| # | 门禁条件 | 说明 |
|---|---------|------|
| 1 | 绑定完整性 | `backfill_status == success`；`bind_mode == deferred` 时豁免 |
| 2 | 无占位符 | 不存在 `待绑定` / `TBD` / 空 `view_id` 占位符；`bind_mode == deferred` 时豁免 |
| 3 | Type 合法性 | 所有 Data Properties 的 Type 在合法类型列表内 |
| 4 | ActionType 合法性 | Bound Object 表 Action Type 列值为 `add`/`modify`/`delete` |
| 5 | Display Key 非空 | 所有对象的 Display Key 非空 |
| 6 | local 对象豁免 | `存储位置: local` 的对象不参与门禁 1/2 检查，仍需满足门禁 3/5 |
| 7 | platform 对象数量 | platform 对象数量 >= 3（一票否决前置条件） |

门禁 1-7 任一不满足则 BLOCKED，阻断推送并提示修复。

### 5.2 关系映射预检 + 修复循环

1. **执行预检**：调用 `../_shared/prepush-validation.md`，输入 `network_dir`
2. **`status: pass`** → 进入 5.3
3. **`status: fail`** → 按预检模块的错误修复指引执行修复，修复后重新预检，**最多循环 3 次**

**修复执行规则**：
- 涉及 `.bkn` 文件修改的修复，必须委托 `../bkn-backfill/SKILL.md` 执行
- 只修确定性错误（拼写不一致、引用错位、遗漏/过期），不做语义判断
- 每次修复后重新预检确认效果
- 循环 3 次仍有错误 → 列出全部剩余错误和 `push_retry_log.yaml` 修复记录，**阻断推送**，提示用户手动修复
  - 用户手动修复完成后，**回到 5.1 重新执行门禁自检**（从 5.1 → 5.2 → 5.3 完整走一遍）
- 所有修复动作记录到 `{network_dir}/push_retry_log.yaml`

### 5.3 推送执行

| 步骤 | 行为 |
|------|------|
| 用户确认推送 | 确认步骤 |
| 推送 | 委托 bkn-openbkn 执行 `bkn push {network_dir}/bkn/` |
| 推送失败 | 解析平台返回的错误信息，按错误类型分类处理（见下方重试策略），修复后重试，**最多 3 次** |

**推送重试策略**：

| 平台错误类型 | 处理策略 |
|------------|---------|
| 属性不存在/无效 | 回到 5.2 预检修复流程，修正后重新预检 → 5.3 |
| 数据资源类型无效 | 检查 local 对象是否误生成了 Data Source，委托 bkn-backfill 修正后回到 5.2 重新预检 → 5.3 |
| 网络结构冲突 | 分析冲突详情，判断是否需回退到阶段三（修复分发）调整 |
| 网络/权限/服务端错误 | 不重试，直接报错，提示用户检查平台状态 |

重试 3 次仍失败 → 输出完整错误日志和修复记录，**阻断推送**，提示用户介入或进入 `bkn-doctor` 诊断。

### 5.4 Q&A 验证（插件阶段）

**前置检测**：读取 `pipeline_state.yaml.plugin_availability.test`

| plugin_availability.test | 执行路径 |
|--------------------------|---------|
| `available` | 执行 `../_plugins/bkn-test/SKILL.md`（qa_verify 模式）；仅验证本次 feedback_brief 中的问题 |
| `unavailable` | 跳过，在 `pipeline_state.yaml.completed_stages` 记录 `stage5_qa_verify: skipped(plugin_unavailable)` |

### 5.5 报告

`../bkn-report/SKILL.md`；报告中注明本次修复来源为 feedback_brief，列出已修复/未修复 issue。插件跳过的阶段在报告中标注原因。

## 约束

- 不修改本次 feedback_brief 范围之外的对象或规则，避免引入新风险
- `ambiguous` 类不自动修复，必须人工确认才能纳入
- 修复轮次上限 2 轮，超出后停止并在报告中说明，等待人工介入
- 定时任务传入的 `feedback_brief`（仅限进阶版，`source: scheduled_task`）视为已经过巡检过滤，无需再做信号有效性校验
- 开源版仅支持 `source: manual` 的手动反馈，用户需自行判断问题有效性