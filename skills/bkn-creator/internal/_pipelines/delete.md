# 删除流程（Delete）

安全删除（整网或局部），删除前充分暴露影响面。

## Skill 路径索引

| skill | 路径（相对本文件） |
|-------|-------------------|
| bkn-report | `../bkn-report/SKILL.md` |
| 公约 | `../_shared/contract.md` |

外部 skill `bkn-openbkn` 通过相对路径读取（`../bkn-openbkn/SKILL.md`）。

## 流程

```
bkn-openbkn(影响预检) → 确认 → bkn-openbkn(删除) → 验证 → bkn-report
```

## 阶段

| # | 步骤 | 行为 |
|---|------|------|
| 1 | 定位删除目标 | kn_id / 对象类 / 关系类 |
| 2 | 影响预检 | 委托 bkn-openbkn：级联影响、失效关系、孤悬对象 |
| 3 | 用户确认 | 必须明确"确认删除"，含糊不算 |
| 4 | 执行删除 | 委托 bkn-openbkn |
| 5 | 验证 | 目标不存在 + 无残留引用 |
| 6 | 报告 | `../bkn-report/SKILL.md` |

## 删除前必须展示

- 删除目标清单（名称 + ID）
- 级联影响：将失效的关系、可能孤悬的对象、受影响的 Action/绑定
- 风险提示：不可恢复
