# 可插拔能力清单

本目录包含 bkn-creator 的进阶能力模块。开源版本可删除此目录，核心流程将自动降级。

## 插件列表

| 插件 | 功能 | 依赖关系 | 开源版降级行为 |
|------|------|---------|---------------|
| `bkn-rules` | 从 PRD/对话/建模提取业务规则，生成可复用的 Skill 文件 | 被 bkn-anchor、bkn-distribute、bkn-test(L3验证)、bkn-review(评分)依赖 | 阶段五跳过 → 无 Skill 文件 → 评分降级 → L3无法验证 |
| `bkn-test` | 生成测试集与验证用例（model_review / rules_verification / qa_verify 三模式） | 被 bkn-review(评分)、create/update/feedback pipeline依赖 | 阶段六/八跳过 → 评分降级 → 无 qa_verify |
| `bkn-anchor` | 将业务规则 Skill 锚定为 BKN 网络中的孤悬对象类 | 依赖 bkn-rules 输出 | 随 bkn-rules 一起跳过 |
| `bkn-distribute` | 将业务规则 Skill 分发到多个 AI 平台本地目录 | 依赖 bkn-rules 输出 | 随 bkn-rules 一起跳过 |

## 插件组

插件按功能域分组，必须整体启用或禁用：

| 组 | 包含插件 | 启用条件 |
|----|---------|---------|
| `rules` | bkn-rules + bkn-anchor + bkn-distribute | `internal/_plugins/bkn-rules/SKILL.md` 存在 |
| `test` | bkn-test | `internal/_plugins/bkn-test/SKILL.md` 存在 |

## 检测协议

Pipeline 启动时通过 `_shared/plugin-check.md` 检测插件可用性，输出 `plugin_availability` 字段。

详见 `_shared/plugin-check.md`。

## 安装/卸载

**安装（进阶版）**：
保留 `_plugins/` 目录即可，pipeline 自动检测并启用。

**卸载（开源版）**：
```bash
rm -rf internal/_plugins/
```
Pipeline 自动降级，不影响核心建模、绑定、推送功能。

## 版本兼容

- 插件与核心版本通过 `MANIFEST.md` 中的 `required_core_version` 声明兼容性
- 当前核心版本：`2.0.0`
- 插件版本变更需更新本文件