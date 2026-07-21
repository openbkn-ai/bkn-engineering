---
name: bkn-skillgen
description: 分析技能包能力缺口，判断是否需要新 skill，生成草案。
---

# 能力缺口分析

公约：`../_shared/contract.md`

## 做什么

分析当前 skill 包的能力缺口，判断复用还是新建，需要时生成 skill 草案。

## 输入

- `user_input`：用户描述的能力诉求
- `current_skill_inventory`（可选）：已有 skill 清单，缺省时主动读取
- `generation_goal`：analyze_only / generate_draft / write_files

## 流程

1. 建立 `current_skill_inventory`
2. 按 `references/gap-checklist.md` 拆解需求
3. 覆盖比对 → 已覆盖 / 部分覆盖 / 缺失
4. 无显著缺口 → 输出 `reuse_plan`，停止
5. 有缺口 → 聚类为候选 skill → 产出 Spec + SKILL.md + 触发测试

## 输出

```yaml
capability_gap_analysis: {user_goal, need_new_skill}
reuse_plan: {recommended_existing_skills, why_not_enough}
skill_generation_plan:
  candidate_skills:
    - skill_name: ""
      role: ""
      boundaries: {does, does_not}
generated_skill_artifacts:
  - skill_name: ""
    skill_md_draft: ""
    trigger_test_set: {should_trigger, should_not_trigger}
```

## 约束

- 先判断是否真需要新 skill，现有覆盖 80%+ 优先建议优化
- 每个新 skill 必须写清"何时用/不用" + 至少 1 负例
- 不改写外部能力（bkn-openbkn / data-semantic）

## 保留名称（禁止新建同名）

`bkn-router`、`bkn-domain`、`bkn-extract`、`bkn-doctor`、`bkn-rules`、`bkn-draft`、`bkn-env`、`bkn-bind`、`bkn-map`、`bkn-backfill`、`bkn-test`、`bkn-review`、`bkn-anchor`、`bkn-report`、`bkn-skillgen`、`bkn-distribute`、`bkn-openbkn`、`bkn-archive`

## 参考

- `references/gap-checklist.md`
