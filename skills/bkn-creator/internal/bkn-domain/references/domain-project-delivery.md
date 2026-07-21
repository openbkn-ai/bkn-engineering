# 项目交付领域提取参考

## 典型对象类（canonical）

项目(`project`)、里程碑(`milestone`)、任务(`task`)、交付物(`deliverable`)、
风险(`risk`)、问题(`issue`)、资源(`resource`)、工时记录(`timesheet`)

## 典型关系类

- 项目 `包含` 里程碑（1:N）
- 里程碑 `包含` 任务（1:N）
- 任务 `产出` 交付物（1:N）
- 任务 `依赖` 任务（N:N）
- 项目 `登记` 风险（1:N）
- 项目 `跟踪` 问题（1:N）
- 资源 `执行` 任务（N:N）
- 资源 `提交` 工时记录（1:N）

## 领域闭环补全

| 命中对象 | 缺失时补全 |
|---------|-----------|
| milestone | task |
| task + 交付/验收语义 | deliverable |
| 阻塞/延期/风险语义 | risk 或 issue |
| 资源投入/工时语义 | resource 或 timesheet |

## 关系识别提示

- "延期、阻塞、依赖、排期冲突" → 任务依赖与资源分配
- "验收、交付、阶段完成" → 里程碑与交付物

## 主键候选

- 编码类：project_code、milestone_id、task_id、deliverable_id
- 单据类：risk_no、issue_no
- 人员/资源类：resource_id、timesheet_id
