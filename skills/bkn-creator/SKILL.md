---
name: bkn-creator
description: >-
  BKN 全生命周期编排入口。按需读取 OpenBKN SDK / CLI 操作参考。
  凡涉及创建 BKN、从 PRD/文档提取对象关系、
  生成或更新 `.bkn`、做数据视图绑定、环境检查、测试集生成、校验与推送时，
  优先使用 bkn-creator 进行流程路由、阶段门禁、子 skill 编排与结果回执。
  适用于 BKN 的 create/read/update/delete/extract/copy/validate 场景，
  技能包能力补齐/skill 草案生成场景，
  以及使用反馈巡检与改进场景（定时任务触发、Agent 对话质量异常、
  feedback_brief 传入、知识网络持续优化）。
  不应用于纯数据语义查询，该场景应交由 data-semantic 处理。
---

# BKN 创建器

本文件是 BKN Creator 的桥接入口。实际能力定义在 `internal/` 目录中。

## 路由表

识别用户意图后，读取对应 pipeline 文件并按其指令执行。

| 意图 | 触发词 | 读取文件 |
|------|--------|----------|
| `create` | 创建知识网络、新建 BKN、根据 PRD 建模 | `internal/_pipelines/create.md` |
| `extract` | 提取对象类、从文档抽取实体关系 | `internal/_pipelines/extract.md` |
| `read` | 查询、搜索、列出、查看 | `internal/_pipelines/read.md` |
| `update` | 修改、编辑、更新、重绑定 | `internal/_pipelines/update.md` |
| `delete` | 删除、清理、移除 | `internal/_pipelines/delete.md` |
| `copy` | 复制、克隆、基于现有生成新版本 | `internal/_pipelines/copy.md` |
| `validate` | 检查网络、评估质量、诊断问题、验证完整性 | `internal/_pipelines/validate.md` |
| `skill_generate` | 补 skill、抽成独立 skill、补齐能力 | `internal/_pipelines/skill-gen.md` |
| `feedback_review` | 巡检反馈、使用问题、定时任务触发、改进摘要、trace 分析结果 | `internal/_pipelines/feedback.md` |

## 执行协议

1. 识别意图 → 输出：识别意图 + 路由目标 + 输入摘要 + 确认请求
2. 路由确认阶段只展示高层信息，不展示对象/关系明细
3. 用户确认后 → **读取**上表中的 pipeline 文件，按文件内指令逐步执行
4. pipeline 中引用的每个 skill，按 pipeline 内标注的相对路径读取其 SKILL.md

## 共享公约

执行前先读取：`internal/_shared/contract.md`

## 约束

- 未确认时只重复"路由识别 + 确认请求"
- 跨流程切换必须重新路由确认
