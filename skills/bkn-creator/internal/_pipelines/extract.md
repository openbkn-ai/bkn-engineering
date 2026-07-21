# 提取流程（Extract）

从业务文档中提取对象类与关系类候选清单，不落盘、不推送。

## Skill 路径索引

| skill | 路径（相对本文件） |
|-------|-------------------|
| bkn-domain | `../bkn-domain/SKILL.md` |
| bkn-extract | `../bkn-extract/SKILL.md` |
| bkn-doctor | `../bkn-doctor/SKILL.md` |
| bkn-report | `../bkn-report/SKILL.md` |
| 公约 | `../_shared/contract.md` |

## 流程

```
bkn-domain → bkn-extract → [bkn-doctor] → 确认 → bkn-report
```

## 阶段

| # | 步骤 | 读取 | 说明 |
|---|------|------|------|
| 1 | 领域识别 | `../bkn-domain/SKILL.md` | 评分式领域匹配；**无论是否命中都需要用户确认** |
| 2 | 对象关系提取 | `../bkn-extract/SKILL.md` | 四分组 + 关系 + 动作 |
| 3 | 质量检查 | pipeline 判定 | pending >= 3 / 冲突 → bkn-doctor |
| 4 | 建模收敛 | `../bkn-doctor/SKILL.md` | 仅质量不达标时 |
| 5 | 待确认对象处理 | 用户确认 | pending 逐项：纳入/移出/保留 |
| 6 | 报告 | `../bkn-report/SKILL.md` | 最终清单 |

## 校验规则

- 对象名归一化：复合表达（含 `→`/`:`/`/`）转入 rejected
- 分组：explicit / inferred / pending
- inferred 必须含 inference_reason
- 关系引用对象必须存在于对象总清单

## 与 create 衔接

提取结果可直接作为 create 的输入，跳过重复提取。
