# 通知文件字段规范

本文档定义机器人通知文件的最小字段集合和填写规则。通知文件用于把项目中的任务分发给指定机器人执行，但通知文件本身不是任务实体，只是任务的分发载体和流程视图。

## 适用范围

本规范适用于：

- `20-robots/<robot-name>/inbox/` 中的通知文件
- `working/`、`done/`、`failed/` 中保留的通知文件
- `00-system/templates/notice-template.md`

## 设计原则

1. 通知只做分发指针  
   通知文件必须指向任务实体，而不是复制完整任务正文。

2. 字段固定且最小化  
   第一版只保留分发必要字段，避免通知文件膨胀成第二份任务文档。

3. 人工可读，后续可解析  
   字段名称应稳定，便于 Markdown 人工阅读，也便于未来脚本解析。

## 文件命名规范

通知文件名必须符合以下格式：

```text
NOTICE-<任务目录名>.md
```

示例：

```text
NOTICE-TASK-002-implement.md
```

## 建议目录位置

通知文件在不同阶段建议位于以下目录：

- 待领取：`20-robots/<robot-name>/inbox/`
- 处理中：`20-robots/<robot-name>/working/`
- 已完成：`20-robots/<robot-name>/done/`
- 执行失败：`20-robots/<robot-name>/failed/`

## 必填字段

通知文件必须至少包含以下字段：

- `notice_id`
- `task_id`
- `project_id`
- `project_path`
- `task_path`
- `assignee_robot`
- `created_at`
- `priority`
- `status`
- `result_path`

## 字段定义

### `notice_id`

通知自身的唯一标识。第一版建议直接使用通知文件名去掉扩展名后的值，例如：

```text
NOTICE-TASK-002-implement
```

### `task_id`

目标任务标识，必须与任务目录名一致，例如：

```text
TASK-002-implement
```

### `project_id`

所属项目标识，必须与项目目录名一致，例如：

```text
PRJ-001-demo
```

### `project_path`

项目目录的相对路径，例如：

```text
10-projects/PRJ-001-demo
```

### `task_path`

任务目录的相对路径，例如：

```text
10-projects/PRJ-001-demo/tasks/TASK-002-implement
```

### `assignee_robot`

目标机器人目录名，例如：

```text
robot-coder
```

### `created_at`

通知创建时间，必须使用 ISO 8601 格式，例如：

```text
2026-04-16T10:30:00Z
```

### `priority`

任务优先级。第一版统一使用以下枚举值：

- `P0`
- `P1`
- `P2`
- `P3`

建议含义：

- `P0`：必须立即处理
- `P1`：高优先级，需优先安排
- `P2`：常规优先级
- `P3`：低优先级或可延期事项

### `status`

通知状态必须与任务状态体系保持一致，仅允许使用以下值：

- `assigned`
- `working`
- `done`
- `failed`

说明：

- 通知文件不使用 `new`，因为 `new` 状态下任务尚未分发
- 通知文件通常也不使用 `archived`，归档后应转入项目归档流程

### `result_path`

任务结果回写路径，建议指向任务目录中的 `output/`，例如：

```text
10-projects/PRJ-001-demo/tasks/TASK-002-implement/output
```

## 建议字段

第一版建议保留以下可选字段：

- `assigned_by`
- `assigned_at`
- `started_at`
- `finished_at`
- `summary`
- `error_summary`
- `next_action`

## 标准结构

通知文件建议使用固定章节顺序：

```markdown
# 通知标题

## 基本信息
- notice_id:
- task_id:
- project_id:
- assignee_robot:
- priority:
- status:

## 路径信息
- project_path:
- task_path:
- result_path:

## 时间信息
- created_at:
- assigned_at:
- started_at:
- finished_at:

## 执行说明
- summary:
- next_action:
- error_summary:
```

## 一致性要求

通知文件必须满足以下一致性要求：

- `task_id` 必须与 `task_path` 指向目录一致
- `project_id` 必须与 `project_path` 指向目录一致
- `assignee_robot` 必须与通知所在机器人目录一致
- `status` 必须与通知所在子目录语义一致

例如：

- 位于 `inbox/` 的通知状态应为 `assigned`
- 位于 `working/` 的通知状态应为 `working`
- 位于 `done/` 的通知状态应为 `done`
- 位于 `failed/` 的通知状态应为 `failed`

## 禁止事项

通知文件中禁止出现以下情况：

- 粘贴完整任务正文替代任务路径引用
- 使用自由格式字段名，导致脚本无法解析
- 把机器人私有中间信息写成项目正式结论
- 通知状态与项目任务状态长期不一致

## 与任务文档的关系

通知文件与 `task.md` 的关系如下：

- `task.md` 是任务事实源
- 通知文件是任务分发视图
- 执行结果最终应回写到任务目录
- 如果通知文件和任务文档冲突，以任务文档为准

## 版本说明

- 当前版本：`v1`
- 生效范围：OCWS 第一版通知文件
