# 更新流程（Update）

在不破坏现有网络完整性的前提下，执行可追踪的变更。

## Skill 路径索引

| skill | 路径（相对本文件） | 类型 |
|-------|-------------------|------|
| bkn-doctor | `../bkn-doctor/SKILL.md` | 核心 |
| bkn-draft | `../bkn-draft/SKILL.md` | 核心 | 内部委托 bkn-archive 生成归档路径 |
| bkn-rules | `../_plugins/bkn-rules/SKILL.md` | **插件** |
| bkn-distribute | `../_plugins/bkn-distribute/SKILL.md` | **插件** |
| bkn-anchor | `../_plugins/bkn-anchor/SKILL.md` | **插件** |
| bkn-bind | `../bkn-bind/SKILL.md` | 核心 |
| bkn-relation-bind | `../bkn-relation-bind/SKILL.md` | 核心 | 关系类型判定 + 中间视图绑定 |
| bkn-map | `../bkn-map/SKILL.md` | 核心 |
| bkn-backfill | `../bkn-backfill/SKILL.md` | 核心 |
| bkn-test | `../_plugins/bkn-test/SKILL.md` | **插件** |
| bkn-review | `../bkn-review/SKILL.md` | 核心 |
| bkn-report | `../bkn-report/SKILL.md` | 核心 |
| bkn-openbkn | `../bkn-openbkn/SKILL.md` | 核心 | OpenBKN CLI 操作（内化） |
| bkn-archive | `../bkn-archive/SKILL.md` | 核心 | 全局归档协议（内化） |
| 公约 | `../_shared/contract.md` | — |
| 插件检测 | `../_shared/plugin-check.md` | — |

> **插件类型**：路径指向 `_plugins/` 目录，开源版可能不存在。pipeline 调用前需检测 `plugin_availability`，不可用时执行降级分支。

## 流程

```
bkn-openbkn(读取) → [bkn-doctor] → bkn-draft(patch) → [bkn-rules(incremental) → bkn-anchor → bkn-distribute]（插件，可跳过）
  → [bkn-bind → bkn-relation-bind → bkn-map → bkn-backfill]
  → [bkn-test → bkn-review]（test 插件，可跳过）→ 门禁自检 → 预检修复循环 → 推送(bkn-openbkn) → 回读 → [bkn-rules(incremental)]（插件，可跳过）→ bkn-report
```

> `[]` 标记的为插件阶段，`plugin_availability` 检测不可用时自动跳过。

## 阶段

| # | 步骤 | 读取 | 说明 |
|---|------|------|------|
| 1 | 读取当前网络 | bkn-openbkn | 定位 kn_id + 现有结构 |
| 2 | 变更分析 | pipeline 判定 | 小改直接 patch；结构性变更走完整流程 |
| 3 | 建模调整 | `../bkn-doctor/SKILL.md` | 仅结构性变更时 |
| 4 | 草案更新 | `../bkn-draft/SKILL.md` | patch 模式 |
| 5 | 规则更新 | 见下方插件降级说明 | 仅结构性变更时；补充新对象/关系的业务规则，更新后分发到各平台 |
| 6 | 视图重绑定 | `../bkn-bind/SKILL.md` → `../bkn-relation-bind/SKILL.md` → `../bkn-map/SKILL.md` → `../bkn-backfill/SKILL.md` | 仅涉及绑定变更时；新增关系绑定阶段 |
| 7 | BKN 模型审查+评审 | 见下方插件降级说明 | 含循环 |
| 8 | 推送 | 见下方详细步骤 | 门禁自检 → 预检修复循环 → 推送 → 回读 → 规则更新 |
| 9 | 报告 | `../bkn-report/SKILL.md` | — |

---

### 阶段 5：规则更新（插件阶段）

**前置检测**：读取 `pipeline_state.yaml.plugin_availability.rules`

| plugin_availability.rules | 执行路径 |
|---------------------------|---------|
| `available` | `../_plugins/bkn-rules/SKILL.md`（incremental 模式）→ `../_plugins/bkn-anchor/SKILL.md` → `../_plugins/bkn-distribute/SKILL.md` |
| `unavailable` | 跳过本阶段，在 `pipeline_state.yaml.completed_stages` 记录 `stage5_rules: skipped(plugin_unavailable)` |

> 仅结构性变更时执行；补充新对象/关系的业务规则，更新后分发到各平台。

---

### 阶段 6：视图重绑定（含关系绑定）

**前置条件**：涉及绑定变更时执行。若 `bind_mode == deferred`，本阶段跳过。

| 步骤 | 读取 | 说明 |
|------|------|------|
| 对象级绑定 | `../bkn-bind/SKILL.md` | 仅涉及新对象或视图变更时 |
| 关系绑定 | `../bkn-relation-bind/SKILL.md` | 仅涉及新关系或关系类型变更时；判定关系类型 + 绑定中间视图 |
| 属性级映射 | `../bkn-map/SKILL.md` | 仅涉及属性变更时 |
| 回填 .bkn | `../bkn-backfill/SKILL.md` | 对象类 + 关系类回填 |

**关系绑定触发条件**：
- 新增关系类
- 修改关系的起点/终点对象（需重新判定类型）
- 原 pending 关系补绑中间视图

---

### 阶段 7：BKN 模型审查+评审（插件阶段）

**前置检测**：读取 `pipeline_state.yaml.plugin_availability.test`

| plugin_availability.test | 执行路径 |
|--------------------------|---------|
| `available` | `../_plugins/bkn-test/SKILL.md`（model_review 模式）→ `../bkn-review/SKILL.md`（含循环） |
| `unavailable` | 跳过本阶段，在 `pipeline_state.yaml.completed_stages` 记录 `stage7_review: skipped(plugin_unavailable)`，直接进入推送门禁 |

---

## 阶段八：推送

推送前必须执行门禁自检和预检修复循环，遵循 `../_shared/contract.md` 推送重试规则。

### 8.1 门禁自检

扫描 `{network_dir}/bkn/` 目录，检查以下硬前置条件：

| # | 门禁条件 | 说明 |
|---|---------|------|
| 1 | 绑定完整性 | 若涉及视图变更，`backfill_status == success`；`bind_mode == deferred` 时豁免 |
| 2 | 无占位符 | 不存在 `待绑定` / `TBD` / 空 `view_id` 占位符；`bind_mode == deferred` 时豁免 |
| 3 | Type 合法性 | 所有 Data Properties 的 Type 在合法类型列表内 |
| 4 | ActionType 合法性 | Bound Object 表 Action Type 列值为 `add`/`modify`/`delete` |
| 5 | Display Key 非空 | 所有对象的 Display Key 非空 |
| 6 | local 对象豁免 | `存储位置: local` 的对象不参与门禁 1/2 检查，仍需满足门禁 3/5 |
| 7 | 新增对象合理性 | 本次新增 platform 对象须有对应的业务规则或绑定意图；仅新增 1-2 个对象时需用户二次确认（防止误操作） |

门禁 1-7 任一不满足则 BLOCKED，阻断推送并提示修复。

### 8.2 关系映射预检 + 修复循环

1. **执行预检**：调用 `../_shared/prepush-validation.md`，输入 `network_dir`
2. **`status: pass`** → 进入 8.3
3. **`status: fail`** → 按预检模块的错误修复指引执行修复，修复后重新预检，**最多循环 3 次**

**修复执行规则**：
- 涉及 `.bkn` 文件修改的修复，必须委托 `../bkn-backfill/SKILL.md` 执行
- 只修确定性错误（拼写不一致、引用错位、遗漏/过期），不做语义判断
- 每次修复后重新预检确认效果
- 循环 3 次仍有错误 → 列出全部剩余错误和 `push_retry_log.yaml` 修复记录，**阻断推送**，提示用户手动修复
  - 用户手动修复完成后，**回到 8.1 重新执行门禁自检**（从 8.1 → 8.2 → 8.3 完整走一遍）
- 所有修复动作记录到 `{network_dir}/push_retry_log.yaml`

### 8.3 推送准备

| 步骤 | 行为 |
|------|------|
| 检查同名网络 | 委托 bkn-openbkn |
| 同名冲突处理 | 若存在同名网络，展示选项：A. 自动加版本后缀 `_v{n}` / B. 用户手动命名 / C. 覆盖推送（需二次确认"确认覆盖"） |
| 用户确认推送 | 确认步骤 |

### 8.4 推送执行 + 重试

| 步骤 | 行为 |
|------|------|
| 推送 | 委托 bkn-openbkn 执行 `bkn push {network_dir}/bkn/` |
| 推送成功 | 进入 8.5 |
| 推送失败 | 解析平台返回的错误信息，按错误类型分类处理（见下方重试策略），修复后重试，**最多 3 次** |

**推送重试策略**：

| 平台错误类型 | 处理策略 |
|------------|---------|
| 属性不存在/无效 | 回到 8.2 预检修复流程，修正后重新预检 → 8.3 → 8.4 |
| 数据资源类型无效 | 检查 local 对象是否误生成了 Data Source，委托 bkn-backfill 修正后回到 8.2 重新预检 → 8.3 → 8.4 |
| 网络结构冲突 | 分析冲突详情，判断是否需回退到阶段六（视图重绑定）或阶段三（建模调整） |
| 网络/权限/服务端错误 | 不重试，直接报错，提示用户检查平台状态 |

重试 3 次仍失败 → 输出完整错误日志和修复记录，**阻断推送**，提示用户介入或进入 `bkn-doctor` 诊断。

### 8.5 完整性检查

回读验证：`openbkn bkn get <kn_id> --stats` 确认对象/关系/动作数量与本地一致。

### 8.6 规则最终更新（插件阶段）

**前置检测**：读取 `pipeline_state.yaml.plugin_availability.rules`

| plugin_availability.rules | 执行路径 |
|---------------------------|---------|
| `available` | 执行 `../_plugins/bkn-rules/SKILL.md`（`incremental` 模式），基于最终推送确认的网络结构更新业务规则 Skill 文件 |
| `unavailable` | 跳过，在 `pipeline_state.yaml.completed_stages` 记录 `stage8_rules_update: skipped(plugin_unavailable)` |

> 增量更新自动执行版本递增和旧版本归档（详见 `bkn-rules` 归档机制）。

## 视图绑定更新触发条件

以下情况必须走 bkn-bind → bkn-relation-bind → bkn-map → bkn-backfill：
- 修改对象绑定视图
- 修改或重算 mapped field
- 属性调整导致需重做映射