# 需求发现 Agentic Harness 协议

## 定位

本文件定义 `bkn-requirement` 的平台无关 Harness 执行协议。它不绑定 Claude、Codex、Cursor 或其他运行时；如果环境支持多个 Agent，可启用独立评审；如果不支持，则由当前 Agent 按同一协议顺序执行。

Harness 的目标不是增加形式化流程，而是把需求发现拆成候选生成、独立检查、定向修订和最终门禁，降低“生成即通过”的风险。

## Skill 边界

本 Harness 只服务 `bkn-requirement`：

- 解决：需求整理、PRD 迭代、质量评估、验收用例、`BKN_Creator` 交接摘要；
- 不解决：生成 `.bkn`、数据绑定、平台推送、工具执行、线上 Action 操作；
- 输入：业务材料、会议纪要、上一版 PRD、调研大纲、客户补充材料；
- 输出：业务可读 PRD / 迭代摘要 / 评审报告 / handoff；
- 成功标准：业务专家可评审，下游 `bkn-creator` 可理解，Verifier 无 hard stop。

## 总流程

```text
Archive / Context
→ Generator
→ Verifier
→ Reviser（如有必要）
→ Final Gate
→ Final Output
```

## 结构化中间产物

Harness 内部按阶段形成轻量交接产物：

| 阶段 | 中间产物 | 用途 |
|---|---|---|
| Archive / Context | source manifest、输入摘要、约束说明 | 证明输入来源和上下文边界 |
| Generator | candidate output | 作为 Verifier 检查对象 |
| Verifier | `verification_findings` | 说明 hard stop、major、minor 和修订建议 |
| Reviser | revision summary | 说明按哪些 findings 修改了什么 |
| Final Gate | final gate result | 判定 pass / warn / fail |

这些中间产物默认不原样写入 PRD 正文。若本轮会在项目目录落盘正式 PRD、PRD 迭代或 handoff，必须同时落盘同名 `*-verification-findings.md`，用于证明 Verifier 和 Reviser 已执行；若只在对话中回答，可只输出摘要。

## 适用范围

必须完整执行 Harness 的模式：

- `prd_mode`
- `prd_iteration_mode`
- `handoff_mode`

可执行轻量 Harness 的模式：

- `research_plan_mode`
- `meeting_digest_mode`
- `review_mode`

`intake_mode` 只需输出最小澄清问题，不强制生成完整验证结果。

## 角色职责

### Generator

Generator 负责生成候选产物，不直接声明最终通过。

输入：

- 本轮输入材料和 source manifest；
- 用户目标和输出模式；
- 已有 PRD 或历史版本；
- 与本模式相关的模板和参考资料。

输出：

- candidate output；
- 候选假设、待确认点和证据不足处；
- 可供 Verifier 检查的场景、规则、验收和 handoff 内容。

工作方法：

- 先判断输出模式，再选择对应模板；
- 先按业务场景组织，再做 BKN 线索收敛；
- 已确认、候选、待确认必须分开；
- 业务正文不输出机器 schema；
- 不把对象、关系、指标、算子、行动写成业务用户必须理解的前置概念；
- PRD 正文的场景收敛优先写“业务对象 / 业务联系 / 业务计算与判断 / 责任权限与留痕”，不要用“算子 — ...”作为正文标签。
- 第 15 节 `BKN_Creator` 交接摘要必须按每个已承诺 P0 / P1 场景使用固定五层表格，不能压缩为 `对象：A、B；关系：A-B；逻辑：X`。
- 第 15 节必须比第 6 节场景收敛更完整，不能更抽象；概念模型层必须说明对象边界。

按模式生成：

- 调研提纲；
- 会议纪要整理；
- 标准 PRD 候选版；
- PRD 迭代候选版；
- 评审报告候选版；
- `BKN_Creator` 中文交接摘要候选版。

生成 PRD 或 handoff 时，必须完成：

- 场景识别；
- 业务规则与决策梳理；
- 系统、表单与数据来源梳理；
- 业务验收用例；
- 场景收敛；
- 需求成熟度与下一步建议。

### Verifier

Verifier 负责独立检查候选产物，不直接改文件。

输入：

- Generator 候选产物；
- source manifest；
- 本轮模式和用户目标；
- Verifier 必读参考。

输出：

- `verification_findings`；
- final gate 初步建议；
- 必须修订项和可接受风险。

Verifier 必须读取：

- `references/requirement-verifier.md`
- `references/anti-drift-checklist.md`
- `references/bkn-requirement-ontology-discovery.md`
- `references/bkn-methodology.md`

Verifier 的检查分两类：

1. 需求规范检查：依据 `SKILL.md` 的输出模式、项目归档要求、模板、`references/bkn-requirement-method.md`、`references/quality-scoring.md` 和 `references/anti-drift-checklist.md`。
2. 方法论检查：依据 `references/bkn-methodology.md` 和 `references/bkn-requirement-ontology-discovery.md`，检查对象、事实属性、关系、指标、算子、行动、治理和 handoff 边界。

Verifier 输出 findings，供 Reviser 使用。除非用户明确要求，不把完整 findings 原样写入 PRD 正文。

Verifier 必须特别检查指标、算子、行动和工具的边界：

- 指标是业务度量，应说明业务口径和适用对象；
- 算子是逻辑判断、计算、推理、推荐或模拟能力，应说明输入、规则、输出和边界；
- 行动是会改变对象、影响对象或触发外部系统的受控业务动作，应说明影响对象、前置条件、治理和审计；
- 工具 / API / MCP 是执行层绑定，不得在需求发现产物中替代业务行动定义。

Verifier 的原则：

- 只检查，不直接重写；
- 每个问题必须给出证据和影响；
- 方法论检查只判断边界是否清楚，不把 PRD 改成 `.bkn` 草案；
- 不确定的内容标为候选或待下游建模判定，不替业务专家确认。

### Reviser

Reviser 只根据 Verifier findings 定向修订，不重新发散生成整篇文档。

输入：

- candidate output；
- `verification_findings`；
- 用户明确补充或更正的信息。

输出：

- revised output；
- revision summary；
- 需要再次验证的问题。

修订规则：

- 先修 hard stop，再修 major；
- 不删除已确认业务事实，除非 Verifier 指出确认状态错误；
- 不把候选项改写成已确认内容；
- 不把 PRD 改成机器 schema；
- 修订摘要必须能追溯到 Verifier finding 或用户补充；
- 修订后必须再次执行 Verifier；
- 自动修订最多 2 轮。

### Final Gate

Final Gate 判定最终交付状态：

| 状态 | 条件 | 输出 |
|---|---|---|
| `pass` | 无 hard stop，major 已处理或可解释 | 最终 PRD / handoff |
| `warn` | 无 hard stop，但仍有关键待确认 | Draft / Candidate，附下一步 |
| `fail` | hard stop 未清除 | 不声明 Ready，输出未通过原因和追问清单 |

## 多 Agent 增强协议

如果运行环境支持多个 Agent，可将 Verifier 拆成独立评审角色。多 Agent 是增强能力，不是基础运行前提。

推荐角色：

| 角色 | 职责 | 是否改文件 |
|---|---|---|
| Requirement Verifier | 检查业务场景、流程、角色、规则、输入输出、验收和待确认问题 | 否 |
| BKN Methodology Verifier | 检查对象、事实属性、关系、指标、算子、行动、治理和 handoff 边界 | 否 |
| Reviser | 汇总 findings，定向修订产物 | 是 |

执行约束：

- Verifier 只能输出 findings，不直接修改 PRD；
- 主 Agent 或 Reviser 负责合并 findings；
- 两个 Verifier 结论冲突时，按更保守的质量门禁处理；
- 多 Agent 不可用时，当前 Agent 必须按同一 Verifier 清单自检。

## 迭代上限

```text
max_revision_rounds: 2
```

超过上限仍未通过时，不继续自动修订；输出：

- 未通过原因；
- 剩余 hard stop / major findings；
- 下一轮调研问题；
- 不应进入 Ready 或 BKN_Creator 的原因。
