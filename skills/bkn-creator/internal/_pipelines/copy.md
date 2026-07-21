# 复制流程（Copy）

保留源网络核心结构，复制生成新网络。

## Skill 路径索引

| skill | 路径（相对本文件） | 类型 |
|-------|-------------------|------|
| bkn-draft | `../bkn-draft/SKILL.md` | 核心 | 内部委托 bkn-archive 生成归档路径 |
| bkn-openbkn | `../bkn-openbkn/SKILL.md` | 核心 | OpenBKN CLI 操作（内化） |
| bkn-env | `../bkn-env/SKILL.md` | 核心 |
| bkn-bind | `../bkn-bind/SKILL.md` | 核心 |
| bkn-map | `../bkn-map/SKILL.md` | 核心 |
| bkn-backfill | `../bkn-backfill/SKILL.md` | 核心 |
| bkn-rules | `../_plugins/bkn-rules/SKILL.md` | **插件** |
| bkn-test | `../_plugins/bkn-test/SKILL.md` | **插件** |
| bkn-review | `../bkn-review/SKILL.md` | 核心 |
| bkn-distribute | `../_plugins/bkn-distribute/SKILL.md` | **插件** |
| bkn-report | `../bkn-report/SKILL.md` | 核心 |
| 公约 | `../_shared/contract.md` | — |
| 插件检测 | `../_shared/plugin-check.md` | — |

> **插件类型**：路径指向 `_plugins/` 目录，开源版可能不存在。pipeline 调用前需检测 `plugin_availability`，不可用时执行降级分支。

## 流程

```
bkn-openbkn(读取源) → bkn-draft(copy) → [bkn-env] → [bkn-bind → bkn-map → bkn-backfill]
  → 门禁自检 → 预检修复循环 → 推送(bkn-openbkn) → 回读 → [bkn-rules(full)]（插件，可跳过）→ bkn-report
```

> `[]` 标记的为插件阶段，`plugin_availability` 检测不可用时自动跳过。

## 阶段

| # | 步骤 | 读取 | 说明 |
|---|------|------|------|
| 1 | 定位源网络 | bkn-openbkn | 确认 kn_id 唯一 |
| 2 | 确认复制计划 | 用户确认 | 范围、命名策略 |
| 3 | 生成复制草案 | `../bkn-draft/SKILL.md`（copy 模式） | 命名冲突按 _v2/_v3 策略 |
| 4 | 环境检查 | `../bkn-env/SKILL.md` | 可选，用户要求时；输出 `plugin_availability` |
| 5 | 视图重绑定 | `../bkn-bind/SKILL.md` → `../bkn-map/SKILL.md` → `../bkn-backfill/SKILL.md` | 可选 |
| 6 | 推送 | 见下方详细步骤 | 门禁自检 → 预检修复循环 → 推送 → 回读 → 规则生成 |
| 7 | 报告 | `../bkn-report/SKILL.md` | — |

---

## 阶段六：推送

推送前必须执行门禁自检和预检修复循环，遵循 `../_shared/contract.md` 推送重试规则。

### 6.1 门禁自检

扫描 `{network_dir}/bkn/` 目录，检查以下硬前置条件：

| # | 门禁条件 | 说明 |
|---|---------|------|
| 1 | 绑定完整性 | 若执行了视图重绑定，`backfill_status == success`；`bind_mode == deferred` 时豁免 |
| 2 | 无占位符 | 不存在 `待绑定` / `TBD` / 空 `view_id` 占位符；`bind_mode == deferred` 时豁免 |
| 3 | Type 合法性 | 所有 Data Properties 的 Type 在合法类型列表内 |
| 4 | ActionType 合法性 | Bound Object 表 Action Type 列值为 `add`/`modify`/`delete` |
| 5 | Display Key 非空 | 所有对象的 Display Key 非空 |
| 6 | local 对象豁免 | `存储位置: local` 的对象不参与门禁 1/2 检查，仍需满足门禁 3/5 |
| 7 | platform 对象数量 | platform 对象数量 >= 3（一票否决前置条件） |

门禁 1-7 任一不满足则 BLOCKED，阻断推送并提示修复。

### 6.2 关系映射预检 + 修复循环

1. **执行预检**：调用 `../_shared/prepush-validation.md`，输入 `network_dir`
2. **`status: pass`** → 进入 6.3
3. **`status: fail`** → 按预检模块的错误修复指引执行修复，修复后重新预检，**最多循环 3 次**

**修复执行规则**：
- 涉及 `.bkn` 文件修改的修复，必须委托 `../bkn-backfill/SKILL.md` 执行
- 只修确定性错误（拼写不一致、引用错位、遗漏/过期），不做语义判断
- 每次修复后重新预检确认效果
- 循环 3 次仍有错误 → 列出全部剩余错误和 `push_retry_log.yaml` 修复记录，**阻断推送**，提示用户手动修复
  - 用户手动修复完成后，**回到 6.1 重新执行门禁自检**（从 6.1 → 6.2 → 6.3 完整走一遍）
- 所有修复动作记录到 `{network_dir}/push_retry_log.yaml`

### 6.3 推送准备

| 步骤 | 行为 |
|------|------|
| 检查同名网络 | 委托 bkn-openbkn |
| 同名冲突处理 | 若存在同名网络，展示选项：A. 自动加版本后缀 `_v{n}` / B. 用户手动命名 / C. 覆盖推送（需二次确认"确认覆盖"） |
| 用户确认推送 | 确认步骤 |

### 6.4 推送执行 + 重试

| 步骤 | 行为 |
|------|------|
| 推送 | 委托 bkn-openbkn 执行 `bkn create` + `push {network_dir}/bkn/ --branch main` |
| 推送成功 | 进入 6.5 |
| 推送失败 | 解析平台返回的错误信息，按错误类型分类处理（见下方重试策略），修复后重试，**最多 3 次** |

**推送重试策略**：

| 平台错误类型 | 处理策略 |
|------------|---------|
| 属性不存在/无效 | 回到 6.2 预检修复流程，修正后重新预检 → 6.3 → 6.4 |
| 数据资源类型无效 | 检查 local 对象是否误生成了 Data Source，委托 bkn-backfill 修正后回到 6.2 重新预检 → 6.3 → 6.4 |
| 网络结构冲突 | 分析冲突详情，判断是否需回退到阶段三（草案）调整 |
| 网络/权限/服务端错误 | 不重试，直接报错，提示用户检查平台状态 |

重试 3 次仍失败 → 输出完整错误日志和修复记录，**阻断推送**，提示用户介入或进入 `bkn-doctor` 诊断。

### 6.5 完整性检查

回读验证：`openbkn bkn get <kn_id> --stats` 确认对象/关系/动作数量与本地一致。

### 6.6 规则生成（插件阶段）

**前置检测**：读取 `pipeline_state.yaml.plugin_availability.rules`

| plugin_availability.rules | 执行路径 |
|---------------------------|---------|
| `available` | 执行 `../_plugins/bkn-rules/SKILL.md`（`full` 模式），为复制后的新网络生成业务规则 Skill 文件。生成后按用户选择分发到目标平台（详见 `bkn-rules` 分发机制） |
| `unavailable` | 跳过本阶段，在 `pipeline_state.yaml.completed_stages` 记录 `stage6_rules: skipped(plugin_unavailable)`，报告中注明"规则插件不可用，未生成业务规则 Skill" |

## 视图绑定更新触发条件

以下情况必须走 bkn-bind → bkn-map → bkn-backfill：
- 复制后需要绑定新数据源
- 源网络视图与新环境不匹配
- 用户明确要求重新绑定