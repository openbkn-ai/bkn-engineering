# 巡检标准（Patrol Standard）

定义知识网络定期巡检的分析指标、异常判定、输出格式。

**插件依赖**：巡检功能完全依赖 `bkn-test` 插件。当 `plugin_availability.test == unavailable`（开源版删除 `_plugins/` 目录）时，整个巡检模块不可用，定时任务不创建，pipeline 阶段八跳过巡检配置和巡检任务创建。

## 巡检目标

知识网络已推送上线后，定期验证：
1. **测试集回归**：网络结构和规则是否仍通过创建时的测试集
2. **问答准确性**：基于知识网络的 Agent 回答是否准确、可用
3. **趋势追踪**：与历史巡检结果对比，发现退化趋势

## 指标体系

### 维度一：结构完整性（测试集回归）

通过 `../_plugins/bkn-test/SKILL.md` model_review 模式的测试用例定期回放：

| 指标 | 来源 | 合格阈值 | 异常判定 |
|------|------|---------|---------|
| L1 对象/属性存在率 | smoke 测试 | 100% | < 100% → 对象/属性被破坏或缺失 |
| L2 关系连通率 | smoke 测试 | ≥ 90% | < 90% → 关系断裂或新增对象未连通 |
| L3 规则回答准确率 | rules 测试 | ≥ 85% | < 85% → 规则失效或描述不准 |
| 反例拦截率 | rules 测试（反例） | ≥ 80% | < 80% → 规则约束力不足 |
| 绑定验证通过率 | binding 测试 | ≥ 90% | < 90% → 视图绑定异常 |

### 维度二：问答准确性（真实对话分析）

分析 Agent 对话日志中与知识网络相关的会话：

| 指标 | 计算方式 | 合格阈值 | 异常判定 |
|------|---------|---------|---------|
| 回答准确率 | 有效回答数 / 总会话数 | ≥ 80% | < 80% → 网络知识覆盖不足 |
| 检索命中率 | 检索到内容的会话 / 总会话 | ≥ 90% | < 90% → 语义搜索配置异常 |
| 平均检索得分 | 检索得分均值 | ≥ 0.6 | < 0.6 → 描述模糊或数据质量差 |
| 用户纠正率 | 用户明确纠正的会话 / 总会话 | ≤ 15% | > 15% → 规则描述或数据有误 |
| 用户追问率 | 追问 2+ 轮的会话 / 总会话 | ≤ 20% | > 20% → 首轮回答不完整 |
| 平均会话轮次 | 总会话轮次 / 总会话数 | — | 趋势性上升 → 回答质量下降 |

### 维度三：趋势分析

对比连续巡检结果：

| 指标 | 异常判定 |
|------|---------|
| 较上次新增测试失败项 | 任何新增失败 → 网络退化 |
| L3 通过率连续下降 2 次 | 规则质量持续退化 |
| 问答准确率下降 > 10% | 期内回答质量显著变差 |
| 用户纠正率上升 > 5% | 用户反馈持续变差 |

### 维度四：推理能力（需人工确认）

L4 推理测试无法自动执行（`bkn-test` 要求 L4 最终判定必须用户确认），巡检时作为建议项输出：

| 指标 | 来源 | 说明 |
|------|------|------|
| L4 推理正确率 | risk 测试（需用户确认） | 仅记录上次 qa_verify 结果，不自动判定 |
| 建议巡检项 | `patrol_result.trend.degradation_alert` | 如 L4 上次不通过，建议用户手动复测 |

## 异常信号定义

与 `feedback_brief` 信号对齐，用于将巡检结果转为可修复的 issue：

| 信号 | 含义 | 判定规则 |
|------|------|---------|
| 无法回答 | 网络完全无响应或返回空 | 会话中 Agent 未提供任何有效信息 |
| 检索得分低 | 有结果但相关性差 | 检索得分 < 0.5 或命中对象 < 3 |
| 用户追问 | 首轮回答不完整 | 同一会话中用户连续追问 2+ 轮 |
| 用户纠正 | 回答错误，用户明确指出 | 用户说"不对""错了""应该是…"等纠正词 |
| 会话极短 | 用户 1 轮即离开 | 会话轮次 = 1 且无有效回答 |

## 巡检输出格式

```yaml
patrol_result:
  network_id: ""
  network_name: ""
  period: "YYYY-MM-DD ~ YYYY-MM-DD"
  executed_at: ""

  # 测试集回归（L1-L3 自动，L4 需人工）
  test_regression:
    l1_pass_rate: 0.95
    l2_pass_rate: 0.88
    l3_pass_rate: 0.82
    l4_pass_rate: 0.65  # 仅记录上次 qa_verify 结果，非自动测试
    counterexample_pass_rate: 0.75
    binding_pass_rate: 0.92
    total_cases: 120
    passed: 98
    failed: 22
    l4_requires_manual_review: true  # L4 始终需人工确认

  # 问答准确性
  qa_accuracy:
    total_sessions: 150
    answer_accuracy: 0.78
    retrieval_hit_rate: 0.85
    avg_retrieval_score: 0.62
    correction_rate: 0.12
    followup_rate: 0.22
    avg_session_turns: 3.2

  # 趋势对比（与上一次巡检）
  trend:
    l3_pass_rate_delta: -0.05
    qa_accuracy_delta: -0.08
    correction_rate_delta: +0.04
    new_failures: 3
    degradation_alert: true  # 任一关键指标超阈值时为 true

  # 问题清单（自动转为 feedback_brief）
  issues:
    - issue_id: "P-001"
      signal: "用户纠正"
      question: "BOM 层级最多几层？"
      frequency: 5
      affected_objects: ["bom_structure"]
      suggestion: "层级规则描述需补充最大层数限制"
    - issue_id: "P-002"
      signal: "检索得分低"
      question: "供应商交货周期多久？"
      frequency: 8
      affected_objects: ["supplier"]
      suggestion: "supplier 对象的 Description 字段可能过于简略"
    - issue_id: "P-003"
      signal: "无法回答"
      question: "MRP 运算逻辑是什么？"
      frequency: 3
      affected_objects: ["mrp_calculation"]
      suggestion: "mrp_calculation 对象可能缺少 Description 或关联关系"

  # 转换后的 feedback_brief（供 pipeline 直接使用）
  feedback_brief:
    network_id: ""
    network_name: ""
    period: "YYYY-MM-DD ~ YYYY-MM-DD"
    source: scheduled_task
    issues:
      - issue_id: "F-001"
        signal: "用户纠正"
        question: "..."
        frequency: 5
        affected_objects: []
        suggestion: ""
```

## 巡检频率

| 网络规模 | 推荐频率 | 说明 |
|---------|---------|------|
| ≤ 10 对象 | 每周 | 小型网络变化较慢 |
| 11–30 对象 | 每 3 天 | 中等网络需较频繁验证 |
| > 30 对象 | 每日 | 大型网络需持续监控 |

## 触发动作

| 条件 | 动作 |
|------|------|
| `degradation_alert: false` | 生成 `patrol_result` 记录到 `{network_dir}/patrol/patrol_log/`，不触发修复 |
| `degradation_alert: true` 且 issues ≤ 3 | 生成 `feedback_brief.yaml` 到 `{network_dir}/patrol/patrol_log/`，自动触发 feedback pipeline 修复 |
| `degradation_alert: true` 且 issues > 3 | 生成 `PATROL_ALERT.md` 到 `{network_dir}/patrol/`，标记人工介入，不自动触发修复 |
| 连续 2 次巡检 degradation | 标记"网络质量告警"，建议进入 bkn-doctor 诊断 |

## 数据源

| 数据 | 获取方式 |
|------|---------|
| 测试集用例 | `{network_dir}/tests/testset.yaml`（由 `bkn-test` model_review 生成） |
| 测试集结果 | 回放 `testset.yaml` 中的用例，执行 `bkn-test` 验证流程 |
| 对话日志 | Agent 对话 trace / 会话记录 API |
| 检索得分 | `openbkn bkn search` 返回的检索分数 |
| 网络结构 | `openbkn bkn get <kn_id> --stats` |
| 历史巡检 | `{network_dir}/patrol/patrol_log/` 下的历史 `patrol_result` |