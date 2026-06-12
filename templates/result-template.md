# 结果回填模板

> 用途：用于任务完成后在 `task.md` 中回填结果，或生成任务输出目录下的结果说明文档。  
> 要求：结果描述必须可追溯到实际输出物，不能只写“已完成”。

## 基本信息

- task_id：`TASK-xxx-name`
- project_id：`PRJ-xxx-name`
- completed_by：`robot-name`
- completed_at：`YYYY-MM-DDTHH:MM:SSZ`
- final_status：`done | failed`

## 完成摘要

简要说明本次执行完成了什么，控制在可快速阅读的粒度。

## 输出物清单

列出本次执行产生的关键输出，并写清路径。

- 输出物 1：
- 输出物路径：
- 输出物说明：

- 输出物 2：
- 输出物路径：
- 输出物说明：

## 处理过程摘要

说明做了哪些关键动作，不要求写成流水账，但应覆盖关键决策或关键步骤。

## 与原目标对比

- 已完成部分：
- 未完成部分：
- 偏差说明：

## 风险与遗留问题

- 风险 1：
- 风险 2：

## 建议下一步

- 下一步 1：
- 下一步 2：

## 验收建议

说明验收人需要重点关注什么。

- 验收关注点 1：
- 验收关注点 2：

## 关联路径

- 任务目录：`10-projects/PRJ-xxx-name/tasks/TASK-xxx-name`
- 输出目录：`10-projects/PRJ-xxx-name/tasks/TASK-xxx-name/output`
- 日志目录：`10-projects/PRJ-xxx-name/tasks/TASK-xxx-name/logs`
