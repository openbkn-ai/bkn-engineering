# 公约

所有 bkn-* skill 共享的唯一协议文件。

## 信封格式

skill 间交接使用统一信封：

```yaml
trace_id: "链路恢复键，同一流程内不变"
intent: create | copy | read | update | delete | extract | validate | skill_generate
from_skill: "产出方"
to_skill: "接收方"
payload:
  result_status: ready | need_user_confirm | blocked | failed
  next_action: "单一默认下一步"
  # ... 各 skill 专有字段
error: null
```

`result_status` 是编排主依据。`next_action` 仅为建议。

`payload` 内部允许中文 key（如 `对象清单`、`关系清单`），外层控制字段（`trace_id`、`intent`、`from_skill`、`to_skill`、`result_status`、`next_action`、`error`）统一使用英文。

## 门禁序列

所有流程统一遵循（pipeline 可插入子阶段）：

`discovery → preview → confirm → execute → verify → report`

- 写操作必须在 `confirm` 之后
- 每轮至多一个确认请求
- 含糊回复不视为确认（"看一下""先这样""嗯"不算）
- 明确确认才算（"确认""是""确定""按此继续"）

## 通用约束（适用于所有 bkn-* skill）

1. `.bkn` 文件的**创建**由 `bkn-draft` 和 `bkn-anchor` 负责，已有 `.bkn` 的**修改**由 `bkn-backfill` 独占负责。其余 skill 不直接读写 `.bkn` 文件
2. 不直接执行 `openbkn` CLI；需要时委托 `bkn-openbkn`（读取 `../bkn-openbkn/SKILL.md`）
3. 不直接调用兄弟 skill；需要时回交 pipeline，由 pipeline 调度
4. 不将推测标记为确定性结论
5. 输入不足时一次只追问一个最高优先级字段

## 通用失败处理

| 情况 | 处理 |
|------|------|
| 必填输入不足 | 追问最高优先级字段 |
| 工具调用失败 | 停在当前步骤，报错等待指令 |
| 结果不确定 | 降低置信度，不自动推进 |

## 推送重试规则

### 统一约束

所有 pipeline（create/update/feedback/copy）推送前必须执行 `_shared/prepush-validation.md` 预检，预检失败按本节重试规则处理。

### 预检失败重试

推送前预检（`_shared/prepush-validation.md`）失败时：
- 按错误类型执行自动批量修复（见 create pipeline 阶段七 7.2）
- 每次修复后**重新执行预检**
- **最多循环 3 次**，仍有错误则阻断推送，提示用户手动修复
- 所有修复动作记录到 `{network_dir}/push_retry_log.yaml`

### 平台推送失败重试

`openbkn bkn push` 返回错误时：
- **可重试错误**（属性不存在/无效、数据资源类型无效）：修复后重试，最多 3 次
- **不可重试错误**（网络/权限/服务端错误）：不重试，直接报错
- **结构冲突**：回退到阶段六调整，不重试
- 重试 3 次仍失败 → 输出完整错误日志，阻断推送

## 术语

| 术语 | 含义 |
|------|------|
| pipeline | 流程编排定义（`_pipelines/*.md`），描述 skill 的调用顺序和分支 |
| skill | 原子能力单元（`bkn-*/SKILL.md`），只做一件事 |
| 内化参考 | `bkn-openbkn/references/` 含 openbkn CLI 完整参考；`bkn-bind/references/` 含语义服务 API 参考（data-semantic），供内部快速查阅 |
| 确认步骤 | 需要用户明确确认才可继续的检查点 |
| `ARCHIVE_ID` | `bkn-archive` → `session_status` → `sessionKey` 最后一段 |
| `TIMESTAMP` | `YYYY-MM-DD-HH-MM-SS` |
| 归档根目录 | `archives/{ARCHIVE_ID}/` |
| 网络目录 | `archives/{ARCHIVE_ID}/{TIMESTAMP}/{NETWORK_DIR_NAME}/`，即 bkn-archive 的 `{ORIGIN_NAME}` |
| BKN 目录 | `{network_dir}/bkn/`，所有 `.bkn` 文件存放处，推送/验证目标 |

## 目录布局约定

所有流程产出的文件按职责分离，统一存放在 `{network_dir}/` 下：

```
{network_dir}/
├── bkn/                          # BKN 核心文件（推送目标）
│   ├── network.bkn
│   ├── object_types/*.bkn
│   ├── relation_types/*.bkn
│   ├── action_types/*.bkn
│   └── concept_groups/*.bkn
├── skills/                       # 业务规则 Skill（本地 + 分发到平台）
│   ├── {network_id}-rules.md
│   └── _archived/
├── tests/                        # 测试集
│   └── testset.yaml
├── reports/                      # 报告
│   ├── REPORT.md
│   └── REPORT.html
├── patrol/                       # 巡检相关
│   ├── PATROL_CONFIG.md
│   ├── PATROL_PROMPT.md
│   └── patrol_log/
├── pipeline_state.yaml           # 流程状态
├── push_retry_log.yaml           # 推送重试日志
└── PUBLISH_MANUAL.md             # 手动发布指南
```

- `openbkn bkn validate` / `openbkn bkn push` / `openbkn bkn pull` 的目标目录统一为 `{network_dir}/bkn/`
- 辅助产物（报告、测试、skill、巡检、日志）不进入 `bkn/` 目录
- 所有引用 `.bkn` 文件的 skill/pipeline 必须使用 `{network_dir}/bkn/` 而非 `{network_dir}/`

## 归档规则

写入文件前必须获取 `ARCHIVE_ID` + `TIMESTAMP`。获取失败则中止写入。
写入后回读校验（存在性、非空、关键字段）。

## 恢复协议

恢复键：`trace_id`。存在时从上次成功步骤的下一步恢复。状态不一致时标记 FAILED。

## Pipeline 状态持久化

每个 pipeline 执行过程中，在 `{network_dir}/pipeline_state.yaml` 记录进度：

```yaml
pipeline: create | update | copy | feedback | validate
trace_id: ""
started_at: ""
current_stage: ""
bind_mode: full | deferred
plugin_mode: full | limited
plugin_availability:
  rules: available | unavailable
  test: available | unavailable
map_completed: true | false | skipped  # bkn-map 执行完成后设为 true；bind_mode == deferred 时为 skipped
env_capability_matrix: {}
skipped_steps: []  # 用户明确要求跳过的步骤记录
completed_stages:
  - stage: ""
    status: success | skipped | failed
    reason: ""  # skipped 时记录原因：plugin_unavailable | user_skipped | bind_mode_deferred
    timestamp: ""
    summary: ""
```

每完成一个阶段更新此文件。恢复时读取此文件定位断点。`bind_mode` 和 `env_capability_matrix` 由 `bkn-env` 写入。`plugin_availability` 和 `plugin_mode` 由 `bkn-env` 在插件检测时写入。`map_completed` 由 `bkn-map` 执行完成后写入（`bind_mode == deferred` 时由 pipeline 设为 `skipped`）。`bkn-backfill` 执行前必须读取此字段确认。后续阶段据此裁剪分支（包括插件阶段跳过逻辑）。

## Skill 生命周期规范

### 命名

| 规则 | 说明 |
|------|------|
| 唯一标识 | 所有 Skill 文件使用 `network_id` 命名，不使用网络名 |
| 单 Skill | `{network_id}-rules.md` |
| 多 Skill | `{network_id}-{group_slug}.md`，`group_slug` 为概念分组英文短标识 |
| 禁止格式 | 含中文、特殊字符、空格的 Skill 文件名 |

### 版本

| 规则 | 说明 |
|------|------|
| 格式 | `major.minor.patch` 语义版本 |
| 首次生成 | `1.0.0` |
| 增量更新 | 递增 minor（`1.0.0` → `1.1.0`） |
| 结构性大改 | 递增 major（`1.5.0` → `2.0.0`） |
| 元信息 | frontmatter 必须包含 `version` 和 `lifecycle_state` |

### 归档

| 规则 | 说明 |
|------|------|
| 归档位置 | `{network_dir}/skills/_archived/` |
| 触发时机 | `bkn-rules` `incremental` 模式更新时 |
| 归档命名 | `{原文件名}-v{旧版本号}-{YYYYMMDD-HHmmss}.md` |
| 平台目录 | `{platform_root}/skills/{network_id}-rules/` 下不归档，直接覆盖 |
| 回滚 | 从 `_archived/` 恢复旧版本到 `skills/` 根目录 |

### 加载

| 消费者 | 加载路径 | 加载规则 |
|--------|---------|---------|
| Cursor Agent | `.cursor/skills/{network_id}-rules/` | 加载目录下所有 `.md` 文件 |
| Claude Agent | `.claude/skills/{network_id}-rules/` | 同上 |
| OpenClaw Agent | `.openclaw/workspace/skills/{network_id}-rules/` | 同上 |
| OpenBKN Decision Agent | KN schema Description 字段 | 通过推送后的 API 获取 |
| 其他用户 | OpenBKN 市场搜索安装 | 通过平台注册机制 |
