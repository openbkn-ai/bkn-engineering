---
name: bkn-distribute
description: 将业务规则 Skill 分发到多个 AI 平台本地目录，用户按需选择目标平台。
---

# 多平台分发

公约：`../../_shared/contract.md`

## 做什么

将 `bkn-rules` 生成的业务规则 Skill 文件，安装到用户选择的一个或多个 AI 平台本地目录中，使各平台的 Agent 能加载并使用这些规则。

## 输入

- `skill_artifact_path`：`bkn-rules` 输出的 Skill 文件路径列表
- `network_id`：网络唯一标识
- `network_name`：网络名称
- `skill_version`（可选）：Skill 版本号，来自 `bkn-rules` 输出

## 已知平台

| 平台 | 默认路径 | 目录标识 |
|------|---------|---------|
| Cursor | `.cursor/skills/{network_id}-rules/` | `.cursor` |
| Claude | `.claude/skills/{network_id}-rules/` | `.claude` |
| OpenClaw | `.openclaw/workspace/skills/{network_id}-rules/` | `.openclaw` |

安装目录标识为工作区根目录下的平台根文件夹（如 `.cursor`、`.claude`、`.openclaw`）。若某平台根目录不存在，仍可询问用户是否创建。
OpenClaw 的 skills 子目录位于 `.openclaw/workspace/` 下，而非 `.openclaw/` 根目录。

## 平台目录结构

各平台统一使用 `skills/{network_id}-rules/` 子目录，与 `.cursor/skills/` 结构一致：

```
{platform_root}/
└── skills/
    └── {network_id}-rules/
        └── {skill_name}.md   # SKILL.md 格式
```

## 流程

### 1. 检测已存在的平台

扫描工作区根目录，识别已知平台目录标识是否存在：

- `.cursor/` → Cursor
- `.claude/` → Claude
- `.openclaw/workspace/` → OpenClaw

输出 `detected_platforms` 列表（包含平台名、路径、是否已存在）。

### 2. 呈现平台选择菜单

向用户展示以下信息：

```
业务规则 Skill 已生成，请选择安装到哪些 AI 平台：

已检测到平台：
  [x] Cursor (.cursor/skills/)     ← 已存在
  [ ] Claude (.claude/skills/)     ← 未检测到

其他已知平台：
  [ ] OpenClaw (.openclaw/workspace/skills/)

请输入选项（多选，用逗号分隔，如 A,C），或输入 N 跳过：
  A. Cursor
  B. Claude（将创建 .claude/skills/ 目录）
  C. OpenClaw（将创建 .openclaw/workspace/skills/ 目录）
  N. 跳过，不安装到任何本地平台
```

- 已存在的平台默认选中（用户可取消）
- 未检测到的平台不默认选中，选择后需确认将创建目录
- 用户可手动输入平台路径（如 `windsurf/.windsurf/skills/`）

### 3. 执行分发

对每个选中的平台：

1. 若平台目录不存在，创建目录结构
   - OpenClaw 特殊路径：`.openclaw/workspace/skills/{network_id}-rules/`
   - 其他平台：`{platform_root}/skills/{network_id}-rules/`
2. 将 `skill_artifact_path` 中的每个 Skill 文件复制到目标路径
   - 若目标文件已存在，覆盖更新
3. 保持源文件名不变

### 4. 回读校验

对每个平台：

- 确认目标目录下每个 Skill 文件存在且内容非空
- 记录 `files_copied` 列表和 `status`

### 5. 输出分发结果

```yaml
distribution_result:
  platforms:
    - platform: cursor
      platform_root: ".cursor"
      target_path: ".cursor/skills/{network_id}-rules/"
      status: success | failed | skipped
      files_copied:
        - source: "{network_dir}/skills/{network_id}-rules.md"
          target: ".cursor/skills/{network_id}-rules/{network_id}-rules.md"
          verified: true
    - platform: claude
      platform_root: ".claude"
      target_path: ".claude/skills/{network_id}-rules/"
      status: success | failed | skipped
      files_copied: []
  overall_status: all_success | partial | failed | skipped
```

## 约束

- 不修改 Skill 文件内容，仅复制
- 各平台结构一致，无需格式转换
- 平台目录下直接覆盖，不归档（归档由 `bkn-rules` 在源头 `{network_dir}/skills/_archived/` 统一管理）
- 跳过（选 N）不阻断后续流程
- 分发失败不阻断平台发布（`openbkn skill register`）
- 若 `skill_artifact_path` 为空（无规则可分发），输出 `skipped: no_skills` 并停止
