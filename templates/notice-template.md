# 通知模板

> 用途：在 `20-robots/<robot-name>/inbox/NOTICE-<project_id>-<task_id>-<attempt_id>.md` 中使用。  
> 要求：通知文件是任务分发指针，不复制完整任务正文；字段必须符合 `../rules/notice-schema.md`。

# 通知：`NOTICE-PRJ-xxx-name-TASK-xxx-name-a1`

## 基本信息

- notice_id：`NOTICE-PRJ-xxx-name-TASK-xxx-name-a1`
- notice_type：`dispatch | verify_result | return | alert | takeover_request`
- task_id：`TASK-xxx-name`
- project_id：`PRJ-xxx-name`
- attempt_id：`a1`
- assignee_robot：`robot-name`
- sender_robot：`robot-planner`
- priority：`P0 | P1 | P2 | P3`
- status：`assigned | working | done | failed`

## 路径信息

- project_path：`10-projects/PRJ-xxx-name`
- task_path：`10-projects/PRJ-xxx-name/tasks/TASK-xxx-name`
- result_path：`10-projects/PRJ-xxx-name/tasks/TASK-xxx-name/output`

## 时间信息

- created_at：`YYYY-MM-DDTHH:MM:SSZ`
- assigned_at：`YYYY-MM-DDTHH:MM:SSZ`
- started_at：
- finished_at：

## 执行说明

- assigned_by：
- summary：
- next_action：
- error_summary：

## 使用说明

- 本文件默认创建于目标机器人 `inbox/`
- 文件名必须含 `project_id` 与 `attempt_id`：`task_id` 仅项目内唯一，批次号用于区分重试
- 机器人领取后，应将文件移动到 `working/` 并同步更新 `status`
- 任务完成后，应将文件移动到 `done/` 并补齐 `finished_at` 与 `summary`
- 任务失败后，应将文件移动到 `failed/` 并补齐 `error_summary` 与 `next_action`
- 项目目录中的 `task.md` 仍然是最终事实源
- 历史批次的通知文件不得删除或覆盖（保留在 `done/`、`failed/` 或 `cache/`）
- `verify_result` / `alert` / `takeover_request` 类通知的接收方不是任务执行者，处理后移入 `cache/`
