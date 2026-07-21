# 插件检测协议

Pipeline 启动时执行此协议，判定进阶能力可用性。

## 检测时机

在 `bkn-env` 执行阶段，作为环境就绪检查的一部分。

## 检测方法

按插件组检测文件存在性：

| 组 | 检测文件 | 存在判定 |
|----|---------|---------|
| `rules` | `../_plugins/bkn-rules/SKILL.md` | 文件存在且非空 |
| `test` | `../_plugins/bkn-test/SKILL.md` | 文件存在且非空 |

**检测路径**：相对于 `_shared/` 目录，即 `../_plugins/{plugin_name}/SKILL.md`

## 输出字段

写入 `pipeline_state.yaml` 和信封：

```yaml
plugin_availability:
  rules: available | unavailable
  test: available | unavailable
plugin_mode: full | limited
```

**判定规则**：

| plugin_availability.rules | plugin_availability.test | plugin_mode |
|---------------------------|--------------------------|-------------|
| available | available | full |
| available | unavailable | limited |
| unavailable | available | limited |
| unavailable | unavailable | limited |

## 降级行为映射

| plugin_mode | 受影响阶段 | 降级行为 |
|-------------|-----------|---------|
| `limited` + rules unavailable | create 阶段五、update 阶段 5、feedback 阶段三 | 跳过 bkn-rules/anchor/distribute；评分规则覆盖率降级 |
| `limited` + test unavailable | create 阶段六/八、update 阶段 7、feedback 阶段四/五 | 跳过 bkn-test；评分测试通过率标记 N/A；无 qa_verify |

## Pipeline 调用协议

Pipeline 调用涉及插件的 skill 前，必须：

1. 读取 `pipeline_state.yaml` 中的 `plugin_availability`
2. 若对应插件 `unavailable`，执行降级分支
3. 降级分支必须在 pipeline 中明确定义，不可隐式跳过

**示例**：

```markdown
## 阶段五：业务规则沉淀

**前置检测**：读取 `pipeline_state.yaml.plugin_availability.rules`

|| plugin_availability.rules | 执行路径 |
||---------------------------|---------|
|| available | 读取 `../_plugins/bkn-rules/SKILL.md` → 执行完整流程 |
|| unavailable | 跳过本阶段，在 `pipeline_state.yaml.completed_stages` 记录 `stage5: skipped(plugin_unavailable)` |
```

## 状态持久化

跳过的阶段必须记录原因：

```yaml
completed_stages:
  - stage: "stage5_rules"
    status: skipped
    reason: plugin_unavailable  # 或 user_skipped
    timestamp: "..."
```

**区分两种跳过原因**：
- `plugin_unavailable`：插件未安装，自动降级
- `user_skipped`：用户主动要求跳过

## 约束

- 检测只做文件存在性判断，不做内容校验
- 插件文件损坏时，pipeline 执行会自然报错，不在此层拦截
- `plugin_mode` 只影响 pipeline 分支，不影响 skill 内部逻辑