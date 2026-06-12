# 事件通知投递协议（Push Notification）

> 版本：`v1`  
> 创建时间：2026-05-30  
> 生效时间：2026-05-30  
> 状态：正式生效  
> 适用范围：OCWS 第 3 阶段

## 1. 动机

当前 OCWS 的任务流转完全依赖各机器人的**心跳轮询（pull）**：

```
执行者完成 → done/ → 门禁者下次心跳发现 → 验收 → Planner 下次心跳发现 → 推进
```

问题：
- 每个环节延迟 = 心跳间隔（通常 5~30 分钟），链路串行后延迟叠加
- 执行者完成了但门禁者不知道，门禁者验收了但计划者不知道
- 项目负责人（人类）完全不知道进展，只能手动追问
- 之前出现"不问不干活"的根因在此

**目标**：在不替代心跳轮询的前提下，增加**事件驱动的主动通知（push）**，让关键状态变更立即触达下游。

## 2. 设计原则

| 原则 | 说明 |
|------|------|
| **push 补充 pull，不替代** | 心跳轮询仍然保留作为兜底；push 只加速关键路径 |
| **通知是指针，不是副本** | 事件通知仅携带事件元数据，不复制任务正文 |
| **收件箱统一** | 事件通知和任务通知共用同一 inbox/，统一处理 |
| **人类知情权** | 计划者是人类的代理，关键事件经计划者汇总后推送 TG |
| **幂等安全** | 重复投递不应造成副作用；通知可被心跳二次确认 |

## 3. 事件类型

### 3.1 机器人间事件

| 事件类型 | 触发方 | 触发时机 | 投递目标 |
|----------|--------|---------|---------|
| `task.completed` | 执行者 | notice → done/ 后 | 门禁者 inbox/ |
| `task.failed` | 执行者 | notice → failed/ 后 | 计划者 inbox/ |
| `task.verified` | 门禁者 | 验收通过，notice → done/ | 计划者 inbox/ |
| `task.returned` | 门禁者 | 验收不通过，退回 | 原执行者 inbox/（含退回原因） |
| `task.reassigned` | 计划者 | 故障接管 / 重分配 | 新执行者 inbox/ |
| `task.blocked` | 任意机器人 | 检测到阻塞（如外部资源不可用） | 计划者 inbox/ |

### 3.2 人类通知事件

| 事件类型 | 触发方 | 触发时机 | 通知方式 |
|----------|--------|---------|---------|
| `project.milestone` | 计划者 | 里程碑达成（如全部 P0 任务完成） | 计划者 → TG 消息 |
| `project.blocked` | 计划者 | 收到 task.blocked 或 task.failed，评估需要人工决策 | 计划者 → TG 消息 |
| `project.completed` | 计划者 | 全部任务归档 | 计划者 → TG 消息 |
| `task.escalation` | 计划者 | 执行者故障接管触发（超 4h 无进展） | 计划者 → TG 消息 |

## 4. 事件通知文件格式

事件通知文件命名：

```text
EVENT-<event_type>-<task_id>.md
```

示例：

```text
EVENT-task.completed-TASK-022-fix-brand-header.md
```

### 4.1 必填字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `event_id` | string | 事件唯一标识，格式 `EVENT-{type}-{task_id}` |
| `event_type` | enum | 见 §3 事件类型 |
| `source_robot` | string | 发起事件的机器人目录名 |
| `target_robot` | string | 接收事件的机器人目录名 |
| `task_id` | string | 关联任务 ID |
| `project_id` | string | 关联项目 ID |
| `timestamp` | ISO 8601 | 事件发生时间 |
| `status` | string | 始终为 `unread`（由目标机器人处理时更新） |

### 4.2 可选字段（按事件类型）

| 事件类型 | 附加字段 |
|----------|---------|
| `task.returned` | `return_reason`（验收不通过原因） |
| `task.failed` | `error_summary`（失败摘要） |
| `task.reassigned` | `reassign_reason`（重分配原因）、`previous_assignee` |
| `task.blocked` | `block_reason`（阻塞原因） |

## 5. 投递规则

### 5.1 投递动作

机器人完成自身阶段工作后，在移动 notice 的同时，**写入事件通知到目标机器人 inbox/**：

```text
# 执行者完成
1. notice 从 working/ → done/
2. 写入 EVENT-task.completed-<task_id>.md → 门禁者 inbox/

# 门禁者验收通过
1. notice 从 done/ → archived（项目 archive/）
2. 写入 EVENT-task.verified-<task_id>.md → 计划者 inbox/

# 门禁者退回
1. notice 从 done/ → 执行者 inbox/（重置为 assigned）
2. 写入 EVENT-task.returned-<task_id>.md → 执行者 inbox/
```

### 5.2 人类通知投递

计划者在处理自身 inbox/ 中的事件时，对需人类感知的事件，通过 Telegram 发送消息：

```
计划者心跳扫描 inbox/ → 发现 task.verified + 项目所有任务完成 
                      → 更新 project.md → TG 通知用户 "项目已完成"
```

## 6. 事件处理流程

### 6.1 机器人处理 inbox/ 事件

```
扫描 inbox/
├── 发现 NOTICE-*.md（任务通知）
│   └── 现有流程：mv → working/ → 执行
├── 发现 EVENT-*.md（事件通知）
│   ├── event_type = task.completed → 进入验收流程
│   ├── event_type = task.returned → 读取退回原因 → 修复 → 重新提交
│   ├── event_type = task.failed → 评估是否重分配
│   └── event_type = task.reassigned → 同新任务领取
└── 处理完成后删除或归档事件文件
```

### 6.2 计划者处理人类通知

```
计划者扫描 inbox/
├── 累积的事件按优先级排序
├── 同类型、同项目事件合并（如 3 个 task.verified → 一条 "3 个任务已验收"）
├── 需要人类决策的 → TG 消息带上下文
└── 纯信息通知 → 汇总到心跳报告，不单独推送
```

## 7. 与心跳轮询的关系

| 维度 | Push（事件通知） | Pull（心跳轮询） |
|------|-----------------|-----------------|
| 触发方式 | 事件驱动，即时 | 定时扫描 |
| 延迟 | 秒级 | 分钟级（心跳间隔） |
| 作用 | 加速关键路径 | 兜底 + 批量处理 |
| 冲突处理 | 以 task.md 为准 | 以 task.md 为准 |

**双通道兼容规则**：
- 心跳扫描到 done/ 中 notice 但未收到对应 EVENT → 视为异常，主动触发事件
- 收到 EVENT 但 notice 状态不匹配 → 以 task.md 为准，记录不一致日志
- 同事件多次投递 → 幂等处理（event_id 去重）

## 8. 人类通知分级

| 级别 | 触发条件 | 通知方式 | 示例 |
|------|---------|---------|------|
| 🔴 紧急 | 需立即决策 | TG 即时消息 | 项目阻塞、故障接管 |
| 🟡 重要 | 里程碑/阶段完成 | TG 即时消息 | 全部 P0 完成、部署上线 |
| 🟢 常规 | 任务级进展 | 心跳汇总报告 | "X 个任务已验收" |
| ⚪ 静默 | 纯流程 | 不通知 | 单任务 assigned → working |

## 9. 目录约定

```
00-system/
├── rules/
│   ├── push-notification.md     ← 本文件
│   └── ...
├── templates/
│   ├── event-notice-template.md ← 新增：事件通知模板
│   └── ...
└── human-inbox/                 ← 新增：人类通知暂存（可选，计划者直接写 TG）

20-robots/
└── robot-{name}/
    ├── inbox/       ← 接收任务通知 + 事件通知（统一入口）
    ├── working/
    ├── done/
    ├── failed/
    └── cache/       ← 新增建议：事件文件处理后可移入 cache/（非归档，可丢弃）
```

## 10. 落地计划

### 阶段 3a（最小可用）

1. 增加 `task.completed` 事件：执行者完成 → 写 EVENT 到门禁者 inbox
2. 增加 `task.verified` 事件：门禁者验收 → 写 EVENT 到计划者 inbox
3. 计划者收到 verified 事件后更新 task.md + project.md，关键事件转发 TG
4. 心跳保留，作为事件投递失败的兜底

### 阶段 3b（完整事件链）

5. 增加 `task.returned`（门禁者退回）、`task.failed`（执行失败）、`task.blocked`（阻塞上报）
6. 增加 `task.reassigned`（故障接管通知新执行者）
7. 人类通知分级制度生效

### 阶段 3c（可靠性与审计）

8. 事件去重、超时重试
9. 事件日志（谁在何时投递了什么事件）
10. 事件统计面板（可选，远期）

## 11. 待讨论

- [ ] 事件通知文件处理完毕后是删除还是移入 cache/？
- [ ] 人类通知是否需要"已读/静音"机制（避免夜间打扰）？
- [ ] EVENT 和 NOTICE 在同一 inbox/ 是否需要优先级排序？
- [ ] 是否需要事件确认机制（ACK）保证投递可靠性？

## 版本说明

- 当前版本：`v1`
- 前置依赖：`notice-schema.md` v1、`task-lifecycle.md` v1
- 下一版本：根据试用反馈调整事件类型和人类通知分级
