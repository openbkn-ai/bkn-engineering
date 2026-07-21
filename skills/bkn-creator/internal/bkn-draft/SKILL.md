---
name: bkn-draft
description: 将确认后的建模清单生成 .bkn 文件。
---

# BKN 草案落盘

公约：`../_shared/contract.md`

## 委托技能

| 技能 | 用途 | 必须 |
|------|------|------|
| `bkn-archive` | 归档路径生成、ARCHIVE_ID/TIMESTAMP、回读校验 | 是 |
| `bkn-openbkn` | `openbkn bkn validate` 结构校验 | 是 |

**执行前必须读取 `references/SPECIFICATION.llm.md` 获取规范模板**，不可凭记忆生成 `.bkn` 格式。
**落盘路径必须走 `bkn-archive`**，确保双轨路径（根段 `archives/`）和回读校验。

## 做什么

将用户确认的对象/关系/动作清单转化为 `.bkn` 文件目录，落盘到归档路径。

## 输入

- 已确认的对象/关系/动作清单（含 `存储位置` 标记）
- `network_context`：网络名称、领域
- `mode`：`create`（新建） | `patch`（更新） | `copy`（复制）

## 流程

1. 读取 `../bkn-archive/SKILL.md`，获取 `ARCHIVE_ID` + `TIMESTAMP` + 归档路径
2. 读取 `references/SPECIFICATION.llm.md`，按 BKN v2.0.1 规范生成 .bkn 文件
   - **业务规则放置**：遵循 `references/SPECIFICATION.llm.md` 的"业务规则放置"章节
   - **输出规则**：遵循 `references/SPECIFICATION.llm.md` 的"输出规则"章节
3. 落盘到 `{network_dir}/bkn/`（即 `archives/{ARCHIVE_ID}/{TIMESTAMP}/{NETWORK_DIR_NAME}/bkn/`）
   - `network.bkn` 放入 `bkn/` 根
   - `object_types/`、`relation_types/`、`action_types/`、`concept_groups/` 等子目录均在 `bkn/` 下创建
4. `network.bkn` 的 `id` 使用 `{network_name}` 的 slug 化形式（中文转拼音或英文直用，全小写、连字符分隔，如 `bu-profit-settlement`），补齐 `icon: icon-dip-graph`、`color: #0e5fc5`
   - 推送后若平台返回新 `kn_id`，委托 `bkn-backfill` 回填更新 `network.bkn` 中的 `id`
   - 禁止使用空字符串 `""` 作为 id，平台 validator 会拒绝
5. 对象类分配随机颜色
6. **存储位置处理**：
   - **`platform` 对象**：生成完整 BKN 格式（含 Data Source、Logic Properties 等）
   - **`local` 对象**（仅用于模型内部逻辑推理，无外部数据源）：
     - **省略 `### Data Source` 节**
     - **省略 `Mapped Field` 列**（Data Properties 表格中不生成此列）
     - **省略 `### Logic Properties` 节**
     - **省略 `Incremental Key`**
     - 仅保留 `Data Properties`（定义对象的基本属性结构）
   - **注意**：`relation_types` 和 `action_types` 中的关系/动作可以引用 local 对象的属性，draft 阶段照常生成对应关系/动作文件
7. **Data Source 处理**（仅 platform 对象）：
   - 若此时已完成视图绑定（有 `binding_decision_list`）→ 写入真实 `view_id`
   - 若尚未绑定 → **省略整个 `### Data Source` 小节**，不写占位符
   - **禁止写 `待绑定` 或任何占位文本**，平台会将其解析为 view ID 导致推送失败
8. `Mapped Field` 同理：无绑定时写 `-`，不写占位
9. Description 仅写稳定业务语义
10. **关系映射属性交叉验证**（生成关系文件后必须执行）：
    - 遍历所有 `relation_types/*.bkn`，对每个关系文件：
      1. 解析 Endpoint 中的 Source 对象 ID 和 Target 对象 ID
      2. 解析 Mapping Rules 中的 Source Property 和 Target Property
      3. 确认 Source Property 存在于 Source 对象 `object_types/{source_id}.bkn` 的 Data Properties 中
      4. 确认 Target Property 存在于 Target 对象 `object_types/{target_id}.bkn` 的 Data Properties 中
      5. 若任一属性不存在 → **修正映射规则中的属性名**使其与目标对象 Data Properties 中的实际属性名一致；若无法确定正确属性名，记录错误并提示
    - 对 `action_types/*.bkn` 同理：确认 Parameter Binding 中引用的属性存在于 Bound Object 的 Data Properties 中
11. 委托 `bkn-openbkn` 执行 `openbkn bkn validate {network_dir}/bkn/`
12. 对照校验清单复核
13. 用户复核

## 输出

- `.bkn` 文件目录（位于 `{network_dir}/bkn/`）
- validate 结果
- **root SKILL.md**（可选）：在 BKN 目录根添加 `{network_dir}/SKILL.md`（与 `network.bkn` 同级），包含简短概述 + 索引表（object | path | relation | path | action | path），方便 agent 定位 `.bkn` 文件

## 校验清单

生成后必须逐项检查：

- [ ] `network.bkn` 在根目录；frontmatter 符合规范（`type: knowledge_network`、`id`、`name`）
- [ ] 每个 `.bkn` 有有效的 YAML frontmatter（`type`、`id`、`name`）
- [ ] 文件放在与 `type` 匹配的文件夹下（`object_types/`、`relation_types/`、`action_types/`、`concept_groups/`）；文件名 = `{id}.bkn`
- [ ] Network Overview 列出**所有**定义 ID — 无遗漏/多余
- [ ] 关系/动作引用已存在的 object-type ID；概念分组只列出已存在的对象
- [ ] **关系映射属性存在性**：所有 relation_types 的 Mapping Rules 中 Source/Target Property 在对应对象 Data Properties 中存在
- [ ] **动作参数属性存在性**：所有 action_types 的 Parameter Binding 中引用的属性在 Bound Object 的 Data Properties 中存在
- [ ] Parameter binding 的 `Source` ∈ `property` | `input` | `const`；YAML 块（如 trigger）可解析
- [ ] 标题层级无跳跃（`#` → `##` → `###` → `####`）
- [ ] 业务规则只在允许位置（见 `references/SPECIFICATION.llm.md` 的"业务规则放置"章节）

## 数据类型选型

生成 Data Properties 时，Type 必须从规范合法类型中选择：

| 语义 | 选型 |
|------|------|
| 数量、金额、单价 | `decimal` |
| 计数、序号、版本号 | `integer` |
| 比率、百分比 | `float` |

**禁止使用 `number`**——平台不认识此类型，推送时报 `InvalidParameter`。

## Action Type 选型

Bound Object 表的 Action Type 列只允许三个值：

| 值 | 语义 |
|------|------|
| `add` | 新增（不可写 `create`，是后端保留字） |
| `modify` | 修改（不可写 `update`，是后端保留字） |
| `delete` | 删除 |

当前平台版本不支持 `query`。如有只读操作需求，暂用 `modify` 并在 Description 标注。

## 资源索引

| 类型 | 规范章节 | 模板 | 示例 |
|------|----------|------|------|
| Network | `knowledge_network` | `assets/templates/network_type.bkn.template` | `references/examples/k8s-network/network.bkn` |
| Object | `object_type` | `assets/templates/object_type.bkn.template` | `references/examples/k8s-network/object_types/pod.bkn` |
| Relation | `relation_type` | `assets/templates/relation_type.bkn.template` | `references/examples/k8s-network/relation_types/pod_belongs_node.bkn` |
| Action | `action_type` | `assets/templates/action_type.bkn.template` | `references/examples/k8s-network/action_types/restart_pod.bkn` |
| Concept group | `concept_group` | `assets/templates/concept_group.bkn.template` | `references/examples/k8s-network/concept_groups/k8s.bkn` |

完整规范：`references/SPECIFICATION.llm.md`。

## 约束

- 本 skill 不做建模决策，只做清单 → 文件的转换
- Description 不写映射猜测信息
- 不写任何占位符文本（"待绑定"/"待确认"/"TBD"等），占位值会被平台当作真实 ID 解析