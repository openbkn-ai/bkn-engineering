# OpenBKN Engineering

中文 | [English](README.md)

OpenBKN Engineering 提供用于构建业务知识网络（BKN）的 Agent Skills 与方法论，帮助团队用可复用的工程流程完成从业务材料到可运行知识网络的交付。

它面向 FDE、AI 工程师、产品团队和交付团队，覆盖从业务输入到已验证、可运营知识网络的完整路径：

```text
业务材料 / 访谈 / PRD 草稿
  -> bkn-requirement
  -> 场景中心 PRD + BKN Creator 交接摘要
  -> bkn-ontology-builder
  -> 业务可评审的本体建模方案
  -> bkn-creator
  -> BKN 建模 / 绑定 / 测试 / 校验 / 发布
  -> 反馈巡检
  -> 交付归档
```

本仓库的目标是让 Agent 工作可追踪、可评审、可验证、可复用。每个阶段都有清晰的输入、输出、交接边界和验证门禁。

## Skills

本仓库发布四个同级 Skill。

| Skill | 定位 | 主要产物 | 不负责 |
|---|---|---|---|
| `bkn-requirement` | 需求发现与 PRD 整理 | 调研提纲、会议 digest、场景中心 PRD、验收用例、BKN Creator 交接摘要 | 不创建 `.bkn` 文件，不绑定数据，不发布平台资源 |
| `bkn-ontology-builder` | 本体建模方案生成与修订 | 业务可评审的本体建模方案、Verifier findings、Final Gate、实现反馈修订说明 | 不创建 `.bkn` 文件，不绑定数据，不发布平台资源 |
| `bkn-methodology` | BKN 建模方法与评审规则 | 对象、关系、事实、指标、行动、治理边界的判断方法 | 不单独执行项目流程 |
| `bkn-creator` | BKN 生命周期编排 | BKN 创建、抽取、更新、复制、校验、绑定、报告、反馈巡检、交付归档 | 不用于纯数据语义查询 |

## 推荐工作流

### 1. 发现需求

当你需要把客户背景、访谈、会议纪要、PRD、BRD、流程说明、系统材料或数据材料整理成业务可读的 PRD 时，使用 `bkn-requirement`。

示例：

```text
使用 $bkn-requirement，把这些访谈纪要整理成场景中心 PRD，并生成 BKN Creator 交接摘要。
```

### 2. 设计本体

当你需要在正式创建 BKN 之前生成业务可评审的本体建模方案时，使用 `bkn-ontology-builder`。

示例：

```text
使用 $bkn-ontology-builder，基于这份 PRD 和交接摘要生成本体建模方案。
```

### 3. 应用方法论

当你需要判断对象边界、关系语义、事实、指标、算子、行动、治理点以及 Skill / Agent 责任边界时，使用 `bkn-methodology`。

示例：

```text
使用 $bkn-methodology，评审这些候选对象和关系是否符合 BKN 建模规则。
```

### 4. 创建并验证 BKN

当工作进入 BKN 创建、抽取、更新、绑定、校验、反馈巡检或交付归档阶段时，使用 `bkn-creator`。

示例：

```text
使用 $bkn-creator，基于这份本体建模方案创建 BKN。先展示路由识别和执行预览，等我确认后继续。
```

## 安装

使用 `npx skills` 安装指定 Skill：

```bash
npx skills add https://github.com/openbkn-ai/bkn-engineering --skill bkn-requirement
npx skills add https://github.com/openbkn-ai/bkn-engineering --skill bkn-ontology-builder
npx skills add https://github.com/openbkn-ai/bkn-engineering --skill bkn-methodology
npx skills add https://github.com/openbkn-ai/bkn-engineering --skill bkn-creator
```

安装完成后，重启你的 Agent 会话，让 Skill 列表刷新。

认证、校验、push / pull、Trace、Eval、admin 等平台操作由 `@openbkn/bkn-sdk` 提供：

```bash
npm install -g @openbkn/bkn-sdk
openbkn --help
```

## 仓库结构

```text
bkn-engineering/
  README.md
  README.zh.md
  LICENSE
  NOTICE
  skills/
    bkn-requirement/
      SKILL.md
      agents/
      assets/
      references/
    bkn-ontology-builder/
      SKILL.md
      agents/
      assets/
      references/
    bkn-methodology/
      SKILL.md
      references/
    bkn-creator/
      SKILL.md
      internal/
        _pipelines/
        _plugins/
        _shared/
        bkn-archive/
        bkn-backfill/
        bkn-bind/
        bkn-doctor/
        bkn-domain/
        bkn-draft/
        bkn-env/
        bkn-extract/
        bkn-map/
        bkn-openbkn/
        bkn-relation-bind/
        bkn-report/
        bkn-review/
        bkn-skillgen/
        references/
```

每个发布 Skill 都必须自包含。Skill 运行时应只引用自身目录内的文件。

## BKN Creator Pipelines

`bkn-creator` 会根据用户意图路由到生命周期 pipeline。

| 意图 | Pipeline |
|---|---|
| 创建 BKN | `internal/_pipelines/create.md` |
| 从文档抽取 BKN 候选元素 | `internal/_pipelines/extract.md` |
| 读取或检查 BKN 资产 | `internal/_pipelines/read.md` |
| 更新或重绑定 BKN | `internal/_pipelines/update.md` |
| 删除 BKN 资产 | `internal/_pipelines/delete.md` |
| 复制或克隆 BKN | `internal/_pipelines/copy.md` |
| 校验和诊断 BKN | `internal/_pipelines/validate.md` |
| 生成 Skill 草案 | `internal/_pipelines/skill-gen.md` |
| 评审反馈并改进 BKN | `internal/_pipelines/feedback.md` |

所有 pipeline 遵循同一组门禁：

```text
discovery -> preview -> confirm -> execute -> verify -> report
```

写操作只有在用户明确确认后才会执行。

## 项目归档约定

客户或项目交付时，建议把源材料和生成产物放在同一个项目目录中：

```text
projects/prj-<project-name>/
  inputs/
    round-01/
      source-manifest.md
  <project>-PRD-v0.1.md
  <project>-ontology-modeling-scheme-v0.1.md
  bkn/
    network.bkn
    object_types/
    relation_types/
    action_types/
  delivery/
    validation-report.md
    feedback-review.md
    final-archive.md
```

不要移动用户原始文件。使用外部源文件时，应将它们复制或登记到当前轮次目录，并记录到 `source-manifest.md`。

## 开发指南

- 对外文档要对用户友好：解释项目做什么、什么时候使用哪个 Skill、如何安装、会得到什么产物。
- 四个 Skill 保持同级，统一放在 `skills/` 下。
- 每个 Skill 保持自包含，支持独立安装。
- 平台操作委托给 `@openbkn/bkn-sdk`。
- 平台写操作前必须获得用户明确确认。
- 引入外部材料时，同步更新 `NOTICE`。

## 许可证

OpenBKN Engineering 采用 Apache License, Version 2.0。详见 [LICENSE](LICENSE) 与 [NOTICE](NOTICE)。
