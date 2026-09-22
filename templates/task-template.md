# 任务说明模板

> 用途：在 `10-projects/<project>/tasks/TASK-xxx-name/task.md` 中使用。  
> 要求：任务进入 `assigned` 前，至少应补齐“基本信息、任务背景、任务目标、输入信息、输出要求、验收标准”。

## 基本信息

- 任务 ID：`TASK-xxx-name`
- 所属项目：`PRJ-xxx-name`
- 当前状态：`new | assigned | working | done | failed | archived`
- 执行批次：`a1`（重分配/退回重交时递增）
- 分配类型：`前端 | 后端 | 混合 | 文档`
- 优先级：`P0 | P1 | P2 | P3`
- 责任机器人：
- 创建时间：`YYYY-MM-DDTHH:MM:SSZ`
- 最近更新时间：`YYYY-MM-DDTHH:MM:SSZ`

> 归档时必须补写 `closure_reason`：`verified | cancelled | terminated`。

## 依赖

列出本任务所依赖的前置任务 ID。计划者（Planner）分发前需检查依赖是否全部归档。

- 依赖任务 ID：无 | TASK-xxx-name
- 依赖任务 ID：无 | TASK-xxx-name

> 无依赖时填「无」或留空。严禁在依赖未全部归档时生成通知。

## 任务背景

说明这个任务为什么存在，它解决项目中的什么问题。

## 任务目标

清楚说明任务完成后应达到的结果。

## 输入信息

列出执行任务所依赖的输入，建议引用当前任务目录或项目共享目录中的实际路径。

- 输入 1：
- 输入 2：

## 输出要求

明确产物应该放在哪里，以及输出形式是什么。

- 输出路径：`output/`
- 输出物说明：

## 执行约束

列出执行过程中的限制条件。

- 时间要求：
- 技术约束：
- 安全或权限约束：
- 不允许做的事情：

## 验收标准

列出验收依据，必须可判断是否达成。

- 验收项 1：
- 验收项 2：

## 执行记录

- 分配给：`robot-name`
- 分配时间：
- 开始时间：
- 完成时间：

## 当前结论

当任务处于不同状态时，这里应至少记录：

- `assigned`：为什么分配给当前机器人、本批次的 `attempt_id`
- `working`：当前进展和阻塞点
- `done`：完成摘要和交付位置（指向 `output/att-{attempt_id}/`）
- `failed`：失败原因和建议下一步
- `archived`：`closure_reason` + 归档人 + 归档时间

> `task.md` 只记录**当前有效结论**。历史批次的结论、产物与证据不得因重试或退回而被覆盖或改写。

## 执行批次记录

每次实际执行追加一行，不得删改历史行：

| attempt_id | 执行者 | 开始 | 结束 | 结果 | 证据路径 |
|------------|--------|------|------|------|---------|
| a1 | `robot-name` | | | done/failed | `logs/att-a1/` |

## 关联路径

- 输入目录：`input/`
- 中间产物目录：`workspace/att-{attempt_id}/`
- 输出目录：`output/att-{attempt_id}/`
- 过程日志目录：`logs/att-{attempt_id}/`

## 变更记录

- `YYYY-MM-DD`：初始化任务说明
