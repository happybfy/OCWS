# 通知模板

> 用途：在 `20-robots/<robot-name>/inbox/NOTICE-TASK-xxx-name.md` 中使用。  
> 要求：通知文件是任务分发指针，不复制完整任务正文；字段必须符合 `../rules/notice-schema.md`。

# 通知：`NOTICE-TASK-xxx-name`

## 基本信息

- notice_id：`NOTICE-TASK-xxx-name`
- task_id：`TASK-xxx-name`
- project_id：`PRJ-xxx-name`
- assignee_robot：`robot-name`
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
- 机器人领取后，应将文件移动到 `working/` 并同步更新 `status`
- 任务完成后，应将文件移动到 `done/` 并补齐 `finished_at` 与 `summary`
- 任务失败后，应将文件移动到 `failed/` 并补齐 `error_summary` 与 `next_action`
- 项目目录中的 `task.md` 仍然是最终事实源
