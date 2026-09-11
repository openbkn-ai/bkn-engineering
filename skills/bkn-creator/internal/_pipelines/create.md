# 创建流程（Create）

从业务输入到可验证的 BKN 网络，含闭环评审与业务规则沉淀。

## Skill 路径索引

执行每一步时，读取对应 SKILL.md 并按其指令操作。

| skill | 路径（相对本文件） | 类型 |
|-------|-------------------|------|
| bkn-domain | `../bkn-domain/SKILL.md` | 核心 |
| bkn-extract | `../bkn-extract/SKILL.md` | 核心 |
| bkn-doctor | `../bkn-doctor/SKILL.md` | 核心 |
| bkn-draft | `../bkn-draft/SKILL.md` | 核心 | 内部委托 bkn-archive 生成归档路径 |
| bkn-env | `../bkn-env/SKILL.md` | 核心 |
| bkn-bind | `../bkn-bind/SKILL.md` | 核心 |
| bkn-relation-bind | `../bkn-relation-bind/SKILL.md` | 核心 | 关系类型判定 + 中间视图绑定 |
| bkn-map | `../bkn-map/SKILL.md` | 核心 |
| bkn-backfill | `../bkn-backfill/SKILL.md` | 核心 |
| bkn-test | `../_plugins/bkn-test/SKILL.md` | **插件** |
| bkn-review | `../bkn-review/SKILL.md` | 核心 |
| bkn-rules | `../_plugins/bkn-rules/SKILL.md` | **插件** |
| bkn-anchor | `../_plugins/bkn-anchor/SKILL.md` | **插件** |
| bkn-distribute | `../_plugins/bkn-distribute/SKILL.md` | **插件** |
| bkn-report | `../bkn-report/SKILL.md` | 核心 |
| bkn-openbkn | `../bkn-openbkn/SKILL.md` | 核心 | OpenBKN CLI 操作（内化） |
| bkn-archive | `../bkn-archive/SKILL.md` | 核心 | 全局归档协议（内化） |
| 公约 | `../_shared/contract.md` | — |
| 显示规范 | `../_shared/display.md` | — |
| 插件检测 | `../_shared/plugin-check.md` | — |

> **插件类型**：路径指向 `_plugins/` 目录，开源版可能不存在。pipeline 调用前需检测 `plugin_availability`，不可用时执行降级分支。

## 阶段总览

```
bkn-domain → bkn-extract → [bkn-doctor] → bkn-draft → bkn-env
  → bkn-bind（local 跳过，输出 resource_schema_map）
  → bkn-relation-bind（关系类型判定 + backing resource 绑定）
  → bkn-map（local 跳过，属性名对齐+回灌，关系 Resource Property 映射）
  → bkn-backfill（local 跳过，级联修正关系/动作属性引用 + 关系类回填 + 写入前预检）
  → [bkn-rules(full) → bkn-anchor → bkn-distribute]（插件，可跳过）
  → [bkn-test(schema_review) → bkn-review]（静态检查，可跳过）
       ↑                                │
       └──── 不达标（调整→重查）───────┘
                    │
                达标/接受
                    ↓
        [bkn-rules(incremental)]（插件，可跳过）
                    ↓
        [bkn-test(rules_verification)]（语义检查，可跳过，用户可"接受现状"跳过）
                    ↓
        推送门禁(7条) → 预检修复循环 → 推送重试 → 回读
                    ↓
        [bkn-test(qa_verify)]（插件，可跳过）→ 巡检 → bkn-report
```

> `[]` 标记的为插件阶段，`plugin_availability` 检测不可用时自动跳过。

## 测试类型说明

| 类型 | 时机 | 模式 | 依据 | 性质 |
|------|------|------|------|------|
| 静态检查 | 推送前（阶段六） | `schema_review` | 本地 `.bkn` 文件 + 业务规则 Skill | 结构验证，不依赖真实数据 |
| 规则验证 | 推送前（阶段六，bkn-rules incremental 之后） | `rules_verification` | 锚定的业务规则 Skill | 语义验证，不依赖真实数据 |
| 数据验证 | 推送后（阶段八） | `qa_verify` | 已推送网络 + 平台数据 + 业务规则 Skill | 真实 Q&A 验证 |

**重要**：推送前没有真实数据，静态检查只验证 `.bkn` 文件结构和业务规则的**存在性**，不验证规则是否实际生效。

## 阶段一：建模

| 步骤 | 读取 | 分支条件 |
|------|------|---------|
| 策略确认 | （pipeline 自身，独占一轮，不执行提取） | 按路径分类选择提取策略：A（结构化文档→直接提取）、A-fast（含汇总表→验证模式）、B（部分信息→领域提取）、C（模糊/冲突→bkn-doctor）；**提取时若领域识别结果 raw_score < 4，走通用提取路径，必须读取 `../bkn-domain/references/generic-extraction.md`** |
| 领域识别 | `../bkn-domain/SKILL.md` | **无论是否命中都需要用户确认**；未识别时走通用提取路径 |
| 对象关系提取 | `../bkn-extract/SKILL.md` | — |
| 质量检查 | （pipeline 判定） | pending >= 3 / 方向冲突 / 主键缺失 → bkn-doctor |
| 建模收敛 | `../bkn-doctor/SKILL.md` | 仅质量不达标时 |
| 清单确认 | 用户确认 | 确认后进入业务规则检查 |
| 业务规则检查 | `../bkn-extract/SKILL.md`（规则检查部分） | 关键规则（高风险）→ 用户必须确认"带风险继续" |

路径分类：
- **A（结构化文档）**：输入含明确定义 → 直接提取
- **A-fast（验证性提取）**：文档含汇总表 → 验证模式
- **B（部分信息）**：3+ 候选可识别 → 领域提取
- **C（不稳定）**：模糊/冲突/pending多 → bkn-doctor

## 阶段二：落盘

| 步骤 | 读取 |
|------|------|
| 生成 .bkn 文件 | `../bkn-draft/SKILL.md` |
| 结构校验 | 由 bkn-draft 委托 bkn-openbkn |
| 落盘摘要 | `../_shared/display.md`（pipeline 直接执行）生成精简摘要：对象类数量及名称、关系类数量及名称、概念分组数量；不展示完整 YAML 内容 |
| 用户复核 | 确认步骤，展示摘要后请用户确认，无异议则进入阶段三 |

## 阶段三：环境检查

| 步骤 | 读取 | 说明 |
|------|------|------|
| 就绪检查 | `../bkn-env/SKILL.md` | 输出 `bind_mode`、`env_capability_matrix`、`plugin_availability` |
| bootstrap | 需搭建则暂停，完成后回检 | 仅 `bind_mode == full` 时 |

`bkn-env` 判定 `bind_mode: deferred`（无可用资源）时，阶段四、阶段五整体跳过，直接进入阶段六。`env_capability_matrix` 和 `plugin_availability` 写入 `pipeline_state.yaml`，供后续阶段裁剪分支。

**插件可用性检测**：`bkn-env` 同时输出 `plugin_availability`：
- `plugin_availability.rules: available` → 阶段六可执行
- `plugin_availability.test: available` → 阶段七/九可执行

## 阶段四：对象资源绑定

**前置条件**：`bind_mode == full`。若 `bind_mode == deferred`，本阶段和阶段五整体跳过，在 `pipeline_state.yaml` 中记录 `stage4: skipped` 且 `relation_bind: skipped`。

**`存储位置: local` 对象处理**：`bkn-bind`、`bkn-relation-bind`、`bkn-map`、`bkn-backfill` 均跳过 local 对象，仅对 `platform` 对象执行绑定/映射/回填。

| 步骤 | 读取 | local 对象处理 |
|------|------|----------------|
| 对象级绑定 | `../bkn-bind/SKILL.md` | 跳过，不参与视图匹配 |
| 用户确认 | 确认步骤 | — |

**关键执行要求**：
- 禁止仅通过 `openbkn dataview list` 的名称列表做字面匹配就判定"无匹配视图"
- 必须通过 `openbkn dataview get <view_id>` 获取每个候选视图的字段 schema，再按 `bkn-bind` 的判定规则（字段兼容性验证）逐对象做绑定决议
- 即使 `dataview list` 中名称看起来不匹配，也可能存在语义相关的视图（如拼音缩写名 `zcfwyj` = "驻场服务业绩"），必须获取 schema 后才能判定

## 阶段五：关系绑定

**前置条件**：`bind_mode == full` 且阶段四已完成。若 `bind_mode == deferred`，本阶段跳过。

**职责**：基于对象资源绑定结果，判定关系类型（direct / indirect），并为 indirect 类型推荐/绑定 backing resource。

| 步骤 | 读取 | 说明 |
|------|------|------|
| 关系类型判定 | `../bkn-relation-bind/SKILL.md` | 分析已绑定对象的资源 property schema，判定每条关系的类型 |
| Backing Resource 确认 | 用户确认（如有 pending） | 对 indirect 类型关系，确认 backing resource 或标记 pending |
| pending 关系处理 | pipeline 判定 | pending 关系不阻断流程，后续可补绑 |

**关系类型判定时机说明**：
- 关系类型判定依赖对象绑定结果（起点/终点对象绑定到哪个视图）
- 在 bkn-bind 之后才能知道视图 schema（外键信息）
- 此时才能判断是否有直接关联字段（direct）或需要 backing resource（indirect）

**pending 关系处理**：
- pending 关系在 `pipeline_state.yaml` 中记录
- 后续可通过 update pipeline 补绑
- 推送前只做 warning，不阻断

## 阶段六：属性映射 + 回填

**前置条件**：`bind_mode == full` 且阶段四、五已完成。若 `bind_mode == deferred`，本阶段跳过，在 `pipeline_state.yaml` 中记录 `stage6: skipped` 且 `map_completed: skipped`。

| 步骤 | 读取 | local 对象处理 |
|------|------|----------------|
| 属性级映射 | `../bkn-map/SKILL.md` | 跳过，无视图可映射 |
| 关系 Resource Property 映射 | `../bkn-map/SKILL.md`（关系部分） | 仅处理 confirmed 的 indirect 关系 |
| 回填 .bkn | `../bkn-backfill/SKILL.md` | 跳过，无需回填 |
| 关系类回填 | `../bkn-backfill/SKILL.md`（关系部分） | 仅处理 confirmed 的关系 |
| 状态守卫 | （pipeline 直接执行） | 调用 bkn-backfill 前确认 `map_completed == true` 或 `skipped`，否则不执行 |
| 状态写入 | （pipeline 直接执行） | `bkn-map` 完成后将 `map_completed: true` 写入 `pipeline_state.yaml` |
| 用户确认 | 确认步骤 | — |

## 阶段七：业务规则沉淀

**前置检测**：读取 `pipeline_state.yaml.plugin_availability.rules`

| plugin_availability.rules | 执行路径 |
|---------------------------|---------|
| `available` | 读取 `../_plugins/bkn-rules/SKILL.md` → 执行完整流程 |
| `unavailable` | 跳过本阶段，在 `pipeline_state.yaml.completed_stages` 记录 `stage7_rules: skipped(plugin_unavailable)`，继续进入阶段八 |

> 插件不可用时，后续阶段八的 `schema_review` 将无 Skill 文件可用，bkn-review 评分规则覆盖率维度降级。

---

**当 `plugin_availability.rules == available` 时执行以下流程**：

| 步骤 | 读取 | 说明 |
|------|------|------|
| 提取业务规则 | `../_plugins/bkn-rules/SKILL.md` | 生成 Skill 文件 |
| 规则确认 | 用户确认 | 确认规则 Skill 内容后进入锚定 |
| 锚定到网络 | `../_plugins/bkn-anchor/SKILL.md` | 每个 Skill → 孤悬对象类 |
| 多平台分发 | `../_plugins/bkn-distribute/SKILL.md` | 用户选择目标平台，安装到各平台 |
| 平台发布 Skill | （pipeline 直接执行） | 可选，注册到 OpenBKN 市场，见下方说明 |

> 此阶段生成业务规则 Skill，供后续阶段八的 schema_review 静态检查使用。

**Skill 消费与分发**：

业务规则 Skill 有不同的消费者，对应不同的安装路径：

| 消费者 | 加载路径 | 是否依赖平台注册 |
|--------|---------|-----------------|
| Cursor Agent | 读取 `.cursor/skills/{network_id}-rules/` 下的 SKILL.md | 不依赖 |
| Claude Agent | 读取 `.claude/skills/{network_id}-rules/` 下的 SKILL.md | 不依赖 |
| OpenClaw Agent | 读取 `.openclaw/workspace/skills/{network_id}-rules/` 下的 SKILL.md | 不依赖 |
| OpenBKN Decision Agent | 读取 KN schema 的 Description 字段（推送后可用） | 不依赖 |
| 其他用户/团队 | 从 OpenClaw 市场搜索并安装 | 依赖 |

> 命名统一使用 `network_id` 标识，不使用网络名。详见 `contract.md` Skill 生命周期规范。

**多平台分发（必须）**：

bkn-anchor 完成后，委托 `bkn-distribute` 将业务规则 Skill 安装到用户选择的一个或多个 AI 平台本地目录中。

执行步骤（委托 `bkn-distribute`）：
1. 扫描工作区根目录，检测已知平台（`.cursor/`、`.claude/`、`.openclaw/` 等）
2. 向用户展示平台选择菜单，列出已检测到的平台及可新增的平台
3. 用户选择后，将 Skill 文件复制到对应平台的 skills 目录（OpenClaw 为 `.openclaw/workspace/skills/`）
4. 回读校验：确认每个目标路径下文件存在且内容非空
5. 分发结果写入信封

若用户选择跳过（选 N），分发状态记为 `skipped`，不阻断后续流程。

**平台发布（可选）**：

多平台分发完成后，询问用户是否将 Skill 发布到 OpenBKN/OpenClaw 市场：

```
业务规则 Skill 已安装到本地平台。
是否同时发布到 OpenBKN 市场，供其他用户搜索和安装？

  A. 是，发布到市场
  B. 否，仅本地使用
```

选 A 时：
1. 检查 `env_capability_matrix.skill_module`
   - `available` → 执行 `openbkn skill register --content-file <path>`，成功后 `openbkn skill list` 回读确认
   - `unavailable` → 提示"Skill 模块在当前环境不可用"，在 `{network_dir}/PUBLISH_MANUAL.md` 生成（含手动发布命令）
2. 发布结果写入信封（`published` / `publish_failed` / `publish_unavailable`）

选 B 则跳过，报告中注明"仅本地安装，未发布到市场"。

## 阶段八：BKN 静态检查 + 评审循环

**前置检测**：读取 `pipeline_state.yaml.plugin_availability.test`

| plugin_availability.test | 执行路径 |
|--------------------------|---------|
| `available` | 读取 `../_plugins/bkn-test/SKILL.md` → 执行 `schema_review` 模式（静态检查）+ bkn-review |
| `unavailable` | 跳过本阶段，在 `pipeline_state.yaml.completed_stages` 记录 `stage6_review: skipped(plugin_unavailable)`，直接进入推送门禁（阶段七） |

> 插件不可用时，推送前无静态检查，bkn-review 评分测试通过率维度降级为 N/A。

**重要**：此阶段为**推送前检查**，不依赖真实数据。包含两部分：
- 静态检查（schema_review）：验证 `.bkn` 文件的结构完整性、业务规则存在性、绑定关系完整性
- 规则语义检查（rules_verification）：验证业务规则的触发条件可判定性、正反例覆盖、溯源完整性
不涉及数据验证（数据验证在阶段八执行）。

---

**当 `plugin_availability.test == available` 时执行以下流程**：

| 步骤 | 读取 | 说明 |
|------|------|------|
| 静态检查 | `../_plugins/bkn-test/SKILL.md`（`schema_review` 模式） | 结构 + 规则存在性 + 绑定完整性 + 风险定义检查 |
| BLOCKED 处理 | （pipeline 判定） | 若数量/复杂度 BLOCKED → 要求重新生成测试集 |
| 评审 | `../bkn-review/SKILL.md` | 质量评分 + 调整分发（规则覆盖率包含 Skill 文件质量子项，见降级说明） |
| 达标? | pipeline 判定 | >= 80 通过；< 80 进入调整 |
| 调整 | 按 bkn-review 建议回调对应 skill | 模型→doctor/extract，绑定→bind/map/backfill，规则→bkn-rules，测试集→bkn-test 重新生成 |
| 重测 | 回到 bkn-test（基于调整后的模型和规则重新执行 schema_review） | 最多循环 3 轮 |
| 规则增量更新 | `../_plugins/bkn-rules/SKILL.md`（incremental 模式） | 仅当 `plugin_availability.rules == available` 时执行；基于最终网络结构更新业务规则 Skill |
| 规则语义检查 | `../_plugins/bkn-test/SKILL.md`（`rules_verification` 模式） | 仅当 `plugin_availability.rules == available` 时执行；在规则增量更新后执行，验证最终规则的语义质量 |
| 规则语义不达标处理 | （pipeline 判定） | rules_verification 不达标时提示用户，用户可确认"接受现状"后继续（不强制 BLOCKED，见 bkn-test SKILL.md）。状态写入 `pipeline_state.yaml` |

### BLOCKED 处理

bkn-test 的 schema_review 模式有数量硬约束。若生成后自检不达标：

1. **BLOCKED** → 输出详细不达标信息
2. pipeline 处理：
   - 若 agent 偷懒（用例简单/数量不足）→ 要求 bkn-test **重新生成**
   - 若模型本身太简单（对象/关系/规则太少）→ 进入**调整**阶段
3. BLOCKED 处理不计入循环次数（只有调整后重测才计数）
4. BLOCKED 处理最多重试 2 次，仍不达标 → 进入评审循环

### 调整路径

bkn-review 评分 < 80 时，按发现的具体问题分类回调：

| bkn-review 发现问题 | 调整目标 | 回调 skill |
|---------------------|---------|-----------|
| 对象缺失/关系错误/属性不完整 | 模型结构 | `bkn-doctor` 或 `bkn-extract`（重新提取） |
| 属性映射不完整/blocked 项多 | 视图绑定 | `bkn-bind` → `bkn-map` → `bkn-backfill` |
| 规则覆盖不足/关键规则缺失 | 业务规则沉淀 | `bkn-rules` |
| 测试集质量差/复杂度不足 | 测试集重新生成 | `bkn-test`（schema_review 模式） |

### 循环控制

- 重测循环最多 3 轮
- 循环 3 轮后仍 < 80 → 输出完整评审报告，询问用户：
  ```
  审查未达标（当前 {score}/100），已循环 3 轮。
  
    A. 接受现状，继续推送
    B. 手动调整后再审
  ```
- 选 A → 记录状态到 `pipeline_state.yaml`，继续阶段九
- 选 B → 暂停 pipeline，用户手动干预后重新进入阶段八

**bkn-review 降级评分逻辑**：

当 `plugin_availability.rules == unavailable` 时：
- 规则覆盖率维度：Skill 文件质量子项权重降为 0，剩余权重等比放大

当 `plugin_availability.test == unavailable` 时：
- 测试通过率维度：标记为 N/A，不参与评分

用户可主动说"跳过审查"或"接受现状推送"退出循环。

## 阶段九：推送

**硬前置门禁**：推送前必须满足以下条件，否则 BLOCKED：
1. 阶段四已执行完毕且 `backfill_status == success`；**若 `bind_mode == deferred`，此条豁免**
2. 所有 `.bkn` 文件中不存在 `待绑定` / `TBD` / 空 `view_id` 占位符；**若 `bind_mode == deferred` 且 `.bkn` 中无 Data Source 段，此条豁免**
3. 所有 Data Properties 的 Type 列值在合法类型列表内（无 `number`）
4. 所有 Action Type 的 Bound Object 表 Action Type 列值为 `add`/`modify`/`delete`（无 `create`/`update`/`query`）
5. 所有对象的 Display Key 非空
6. **`存储位置: local` 的对象**：不参与门禁 1（视图绑定）、门禁 2（Data Source 占位符）检查，但仍需满足门禁 3（Type 合法性）、门禁 5（Display Key 非空）
7. **关系映射完整性预检**（`_shared/prepush-validation.md`）通过：所有 relation_types 的 Source/Target Property 存在、所有 action_types 的 Parameter Binding 属性存在、Concept Group 成员存在、Network Overview 一致
8. **静态检查完成**：若阶段八执行，测试集 `quantity_check.overall` 必须为"达标"；若 BLOCKED 但用户已确认"接受现状"，需记录状态到 `pipeline_state.yaml`；**若阶段八跳过（test 插件不可用），此条豁免**
9. **审查评分记录**：若阶段八执行，bkn-review 评分必须已输出（无论是否达标）；**若阶段八跳过（test 插件不可用），此条豁免**，记录 `stage8_review: skipped(plugin_unavailable)`

### 9.1 门禁自检

扫描 `{network_dir}/bkn/` 目录，逐项校验门禁 1-6（根据 `bind_mode` 和 `存储位置` 应用豁免）。门禁 1-6 任一不满足则 BLOCKED。门禁 7-9（状态检查）统一在 9.2 后执行。

### 9.2 关系映射预检 + 修复循环

1. **执行预检**：调用 `_shared/prepush-validation.md`，输入 `network_dir`，获取预检结果
2. **`status: pass`** → 进入 9.3
3. **`status: fail`** → 按 `_shared/prepush-validation.md` 中的错误修复指引执行修复，修复后重新预检，**最多循环 3 次**

**修复执行规则**：
- 涉及 `.bkn` 文件修改的修复（属性名修正、Overview 同步等），必须委托 `bkn-backfill` 执行，pipeline 不直接写 `.bkn` 文件（遵循 `contract.md` 约定）
- 只修**确定性错误**（拼写不一致、引用错位、遗漏/过期），不做语义判断
- 每次修复后必须**重新执行预检**确认修复效果
- 循环 3 次仍有错误 → 列出全部剩余错误和 `push_retry_log.yaml` 修复记录，**阻断推送**，提示用户手动修复
  - 用户手动修复完成后，**回到 9.1 重新执行门禁自检**（从 9.1 → 9.2 → 9.3 完整走一遍）
- 所有修复动作记录到 `{network_dir}/push_retry_log.yaml`，可追溯

### 9.3 推送准备

| 步骤 | 行为 |
|------|------|
| 检查门禁 8-9 | 验证阶段八状态（测试集达标情况、审查评分记录） |
| 检查同名网络 | 委托 bkn-openbkn |
| 同名冲突处理 | 若存在同名网络，展示选项：A. 自动加版本后缀 `_v{n}` / B. 用户手动命名 / C. 覆盖推送（需二次确认"确认覆盖"） |
| 用户确认推送 | 确认步骤 |

### 9.4 推送执行 + 重试

| 步骤 | 行为 |
|------|------|
| 推送 | `openbkn bkn create` + `push {network_dir}/bkn/ --branch main`（委托 bkn-openbkn） |
| 推送成功 | 进入 9.5 |
| 推送失败 | 解析平台返回的错误信息，按错误类型分类处理（见下方重试策略），修复后重试，**最多 3 次** |

**推送重试策略**：

| 平台错误类型 | 处理策略 |
|------------|---------|
| 属性不存在/无效 | 回到 9.2 预检修复流程，修正后重新预检 → 9.3 → 9.4 |
| 数据资源类型无效 | 检查 local 对象是否误生成了 Data Source，委托 bkn-backfill 修正后回到 9.2 重新预检 → 9.3 → 9.4 |
| 网络结构冲突 | 分析冲突详情，判断是否需回退到阶段八调整 |
| 网络/权限/服务端错误 | 不重试，直接报错，提示用户检查平台状态 |

重试 3 次仍失败 → 输出完整错误日志和修复记录，**阻断推送**，提示用户介入或进入 `bkn-doctor` 诊断。

### 9.5 完整性检查

回读验证：`openbkn bkn get <kn_id> --stats` 确认对象/关系/动作数量与本地一致。

### 9.6 规则最终更新

**前置检测**：读取 `pipeline_state.yaml.plugin_availability.rules`

| plugin_availability.rules | 执行路径 |
|---------------------------|---------|
| `available` | 执行 `../_plugins/bkn-rules/SKILL.md`（`incremental` 模式），基于最终推送确认的网络结构更新业务规则 Skill 文件 |
| `unavailable` | 跳过，在 `pipeline_state.yaml.completed_stages` 记录 `stage9_rules_update: skipped(plugin_unavailable)` |

> 增量更新自动执行版本递增和旧版本归档（详见 `bkn-rules` 归档机制）。

## 阶段十：Q&A 数据验证 + 巡检 + 报告

**前置检测**：读取 `pipeline_state.yaml.plugin_availability.test`

| plugin_availability.test | 执行路径 |
|--------------------------|---------|
| `available` | 执行完整流程：验收引导 + Q&A 数据验证 + 巡检 + 报告 |
| `unavailable` | 跳过 Q&A 验证和巡检，仅执行报告；在 `pipeline_state.yaml.completed_stages` 记录 `stage10_qa_verify: skipped(plugin_unavailable)`、`stage10_patrol: skipped(plugin_unavailable)` |

**重要**：此阶段为**数据验证**，基于已推送网络上的真实数据。与阶段八的静态检查（schema_review）不同，qa_verify 使用平台上的实际数据验证 KN 的语义可达性和 Agent 推理能力。

---

**当 `plugin_availability.test == available` 时执行以下流程**：

| 步骤 | 读取 | 说明 |
|------|------|------|
| 数据准备 | （pipeline 直接执行） | 收集 kn_id、network_name、available_models、all_object_type_ids（见下方说明） |
| 验收引导 + Q&A 数据验证 | `../_plugins/bkn-test/SKILL.md`（qa_verify 模式） | 生成分级问题 → 呈现验收引导卡 → 询问用户选择验证方式 → 使用真实数据验证 |
| 巡检配置 | `../references/patrol-standard.md` | 读取巡检指标和标准 |
| 巡检任务创建 | 读取 `pipeline_state.yaml` 中的 `env_capability_matrix.patrol_cron` | 能力检测决定自动创建或降级 |
| 报告 | `../bkn-report/SKILL.md` | 最终归档报告 |

> 此阶段基于阶段七已生成的业务规则 Skill，在已推送网络上做实际验证。若阶段七跳过（rules 插件不可用），L3 规则验证将标记为 `skipped(no_skill_file)`。

**qa_verify 不依赖平台发布**。qa_verify 验证的是 KN schema（已推送到平台）和本地 SKILL.md（阶段七已分发），两者都不需要平台注册。

**qa_verify 执行说明**：
- 引导卡为必出产物，无论用户选哪种验证方式
- L1-L3 验证基于 KN schema（`openbkn context-loader` / `openbkn bkn search`）
- **L3 级问题必须基于 `{network_dir}/skills/{network_id}-rules.md` 业务规则 Skill 文件生成**（见 `bkn-test` qa_verify L3 规则验证）
- 若阶段七跳过导致 Skill 文件不存在，L3 级验证标记为 `skipped(no_skill_file)`
- L4 推理验证基于 Agent 加载本地 SKILL.md 的能力（如当前工作区已分发），或 OpenBKN Agent chat（如可用）
- L4 推理题的最终判定，始终需要用户确认
- 有 fail 项不阻断报告生成，在报告中标记并提示可进入 update pipeline 修复

**数据准备步骤（阶段十第一步）**：
1. 从 `pipeline_state.yaml` 读取 `kn_id`（阶段九推送返回）和 `network_name`
2. 扫描 `{network_dir}/bkn/*.bkn`，提取所有 object_type ID 列表，用于 Agent 配置的 `fields` 字段
3. 执行 `openbkn llm list`（委托 bkn-openbkn）获取平台可用模型；若无可用模型，询问用户选择
4. 将上述参数传递给 `bkn-test` 的 qa_verify 模式（详见 bkn-test SKILL.md 中 "Agent 配置数据准备" 节）

---

**巡检任务创建（P1-5/P1-6）**：

读取 `env_capability_matrix.patrol_cron` 决定巡检创建策略：

**路径 A：定时任务可用 → 自动创建巡检任务**

读取 `pipeline_state.yaml` 中的 `env_capability_matrix.patrol_cron` 确认为 `available` 后：

```
检测到 OpenClaw 定时任务能力可用，将为「{network_name}」创建自动巡检任务。

巡检配置：
  · 网络：{network_name}（{network_id}）
  · 执行频率：{weekly | every_3_days | daily}（根据对象数量自动判定）
  · 分析指标：结构完整性（L1-L3）+ 问答准确性（详见 patrol-standard.md）
  · 触发动作：异常时自动生成 feedback_brief → 触发 feedback pipeline 修复

请确认：
  A. 确认创建，按上述配置
  B. 调整配置（可修改频率、指标阈值）
  C. 跳过，仅生成 Prompt 模板
```

选 A/B 时：
1. 读取 `../references/patrol-standard.md` 中的指标定义和阈值
2. 根据网络对象数量自动判定推荐频率（≤10 → 每周，11-30 → 每3天，>30 → 每日）
3. 尝试创建巡检任务（委托 bkn-openbkn 调用 OpenClaw 调度 API；如 API 不可用则降级为输出 PATROL_CONFIG.md 供手动配置）
4. 输出 `PATROL_CONFIG.md` 到 `{network_dir}/patrol/`，包含：
   - 任务 ID 和执行配置（如自动创建成功）或手动配置指令
   - 巡检指标和阈值
   - 触发条件和触发动作
   - 手动触发和查询指令
5. 报告中记录巡检任务配置路径

选 C 或 API 不可用时降级为路径 B。

**路径 B：不可用或用户选择跳过 → 生成 Prompt 模板（v1 降级方案）**

```
当前环境不支持自动创建巡检任务，将生成 Prompt 模板。
模板可手动配置到定时任务系统，定期分析 Agent 对话质量并触发改进建议。

  A. 生成模板
  B. 跳过
```

选 A 后，在 `{network_dir}/patrol/PATROL_PROMPT.md` 中输出：
1. 巡检 prompt 模板（包含 network_id、Agent ID、分析指令）
2. 推荐执行频率（周/日）
3. 异常信号判定标准（无法回答 / 检索得分低 / 用户追问或纠正 / 会话极短）
4. 触发 feedback_review pipeline 的指令模板（含 `feedback_brief` 格式）

报告中记录巡检模板路径。选 B 则跳过，报告中注明"未生成巡检模板，如需启用请手动触发 feedback_review"。
