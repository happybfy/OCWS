# 事件通知模板

> 用途：机器人间投递事件通知时使用。  
> 要求：仅携带事件元数据，不复制任务正文；字段符合 `../rules/push-notification.md` 规范。

# 事件通知：`EVENT-{event_type}-{task_id}`

## 基本信息

- event_id：`EVENT-{event_type}-{task_id}`
- event_type：`task.completed | task.failed | task.verified | task.returned | task.reassigned | task.blocked`
- source_robot：`robot-{name}`
- target_robot：`robot-{name}`
- task_id：`TASK-{xxx}-{name}`
- project_id：`PRJ-{xxx}-{name}`
- timestamp：`YYYY-MM-DDTHH:MM:SSZ`
- status：`unread`

## 路径引用

- project_path：`10-projects/PRJ-{xxx}-{name}`
- task_path：`10-projects/PRJ-{xxx}-{name}/tasks/TASK-{xxx}-{name}`

## 事件载荷（按类型选填）

- return_reason：（仅 task.returned）
- error_summary：（仅 task.failed）
- reassign_reason：（仅 task.reassigned）
- previous_assignee：（仅 task.reassigned）
- block_reason：（仅 task.blocked）

## 处理说明

- 目标机器人心跳或实时扫描时处理此文件
- 处理完成后移入 `cache/` 或删除（视机器人策略）
- event_id 用于去重，同一 event_id 不重复处理
