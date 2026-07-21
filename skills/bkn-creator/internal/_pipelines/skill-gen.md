# 能力补齐流程（Skill Generate）

分析技能包能力缺口，产出复用方案或新 skill 草案。用户主动发起。

注意：业务规则 Skill 沉淀由 create pipeline 阶段五自动触发，不走本流程。

## Skill 路径索引

| skill | 路径（相对本文件） |
|-------|-------------------|
| bkn-skillgen | `../bkn-skillgen/SKILL.md` |
| bkn-report | `../bkn-report/SKILL.md` |
| 公约 | `../_shared/contract.md` |

## 流程

```
bkn-skillgen → 确认 → bkn-report
```

## 阶段

| # | 步骤 | 读取 | 说明 |
|---|------|------|------|
| 1 | 分析能力缺口 | `../bkn-skillgen/SKILL.md` | 现有 skill 覆盖比对 |
| 2 | 方案确认 | 用户确认 | 复用已有 or 生成新 skill |
| 3 | 生成草案 | `../bkn-skillgen/SKILL.md` | Spec + SKILL.md + 触发测试 |
| 4 | 报告 | `../bkn-report/SKILL.md` | — |

## 两种模式对比

| 维度 | 本流程（capability_gap） | create 阶段五（business_rules） |
|------|--------------------------|--------------------------------|
| 触发 | 用户主动 | pipeline 必执行 |
| 输入 | 用户描述的能力诉求 | PRD/对话/规则/绑定 |
| 输出 | 缺口分析 + skill 草案 | 业务规则 Skill + 网络锚定 |
