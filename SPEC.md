# OCWS 规范 v1.0

> Open Claw Workspace Specification  
> 轻量级多机器人协作工作空间协议

---

## 目录

1. [概述](#1-概述)
2. [目录结构](#2-目录结构)
3. [角色体系](#3-角色体系)
4. [任务状态流转](#4-任务状态流转)
5. [通知文件规范](#5-通知文件规范)
6. [事件通知（Push）](#6-事件通知push)
7. [命名规范](#7-命名规范)
8. [协作流程](#8-协作流程)
9. [任务派发模式](#9-任务派发模式)
10. [Sponsor 唤醒 Planner](#10-sponsor-唤醒-planner)

---

## 1. 概述

### 1.1 设计目标

OCWS 解决的核心问题：**多个 AI Agent（机器人）如何在同一套项目文件上自主协作，无需中心调度服务。**

### 1.2 核心原则

| 原则 | 说明 |
|------|------|
| **目录即协议** | 约定好的目录结构就是 API，`mv` 就是操作 |
| **单一事实源** | `project.md` / `task.md` 是唯一真相，通知只是指针 |
| **通知是指针** | 通知文件只含元数据（ID/路径/状态/分配对象），不复制正文 |
| **原子操作为锁** | NFS `mv` 原子操作保证**同一机器人多进程**的领取互斥（不是任务只执行一次的保证） |
| **规则先于工具** | 行为规范写在文档中，不依赖定制化软件 |

### 1.3 最小依赖

- 共享文件系统（NFS / SMB / 本地目录）
- 支持原子 `mv` 操作
- 各机器人可读写共享目录

---

## 2. 目录结构

```
workspace/
├── 00-system/                     # 静态规范层
│   ├── rules/                     # 规则文档（命名、流转、格式）
│   │   ├── naming.md
│   │   ├── task-lifecycle.md
│   │   ├── notice-schema.md
│   │   ├── project-schema.md
│   │   ├── push-notification.md
│   │   ├── verification-rules.md
│   │   ├── authorization-boundary.md
│   │   └── content-registry.md
│   └── templates/                 # 标准模板
│       ├── project-template.md
│       ├── task-template.md
│       ├── notice-template.md
│       ├── event-notice-template.md
│       ├── infra-template.md
│       └── result-template.md
│
├── 10-projects/                   # 事实源：项目实体
│   └── PRJ-{编号}-{短名}/
│       ├── project.md             # ← 项目唯一事实源
│       ├── infra.md               # ← 基础设施信息（含敏感凭证）
│       ├── shared/                # 项目共享资料
│       │   ├── input/
│       │   ├── output/
│       │   └── refs/
│       ├── tasks/                 # 任务实体
│       │   └── TASK-{编号}-{短名}/
│       │       ├── task.md        # ← 任务唯一事实源
│       │       ├── input/         # 任务输入
│       │       ├── workspace/     # 执行中间产物
│       │       ├── output/        # 最终交付物
│       │       └── logs/          # 过程记录
│       └── archive/               # 已归档任务或历史资料
│
└── 20-robots/                     # 机器人工作入口
    └── robot-{短名}/
        ├── profile.md             # 机器人自述
        ├── inbox/                 # 待领取通知
        ├── working/               # 执行中
        ├── done/                  # 已完成
        ├── failed/                # 执行失败
        └── cache/                 # 临时缓存
```

### 2.1 目录职责

| 目录 | 职责 | 谁写 |
|------|------|------|
| `00-system/` | 规则与模板 | 所有参与者（改规则需共识） |
| `10-projects/` | 项目事实源 | Sponsor + Planner |
| `20-robots/` | 机器人流转 | 各机器人自己维护 |

---

## 3. 角色体系

### 3.1 角色定义

| 角色 | 职责 | 产出 | 不越界 |
|------|------|------|--------|
| **Sponsor** | 项目创建、任务定义、基础设施保障 | `project.md`、`task.md`、`infra.md` | 不分发、不执行 |
| **Planner** | 读 inbox、拆任务、生成通知、分发、重分配 failed | 通知文件、状态更新 | 不执行、不验收 |
| **Executor** | 原子领取（mv）、执行、产出、回写 task.md | 交付物、执行报告 | 不规划、不验收 |
| **Gatekeeper** | 巡检 working/、验收 done/、出具归档/退回**结论** | 验收报告、EVENT 文件 | 不修改产出、不代执行、不自行归档 |

### 3.2 角色交互流

```
Sponsor 创建项目/任务 → Planner 读取分发 → Executor 领取执行 
    → Gatekeeper 验收 → Planner 汇总 → Sponsor 确认
```

---

## 4. 任务状态流转

### 4.1 六种状态

```
new → assigned → working → done → archived
                     │          │
                     │          └─→ assigned（验收退回，新建批次）
                     ↓
                  failed
                     ├─→ assigned（补齐后重分配）
                     └─→ archived（终止）
```

### 4.2 状态含义

| 状态 | 含义 | 通知位置 |
|------|------|---------|
| `new` | 已创建，未分配 | — |
| `assigned` | 已分配，待领取 | `inbox/` |
| `working` | 执行中 | `working/` |
| `done` | 已完成，待验收 | `done/` |
| `failed` | 执行失败 | `failed/` |
| `archived` | 已闭环（须记 `closure_reason`：`verified` / `cancelled` / `terminated`） | 项目 `archive/` |

### 4.3 非法流转（禁止）

- `new → done/working`（跳过分配）
- `assigned → done`（跳过执行）
- `done → working`（回退）
- `archived → working/assigned`（归档复活）

合法但易混淆的两条：

- `done → assigned`：**验收退回**（已交付但不合格），计划者重分配并开启新批次
- `failed → assigned`：**执行失败后重分配**（补齐输入或修复阻塞）

两者都必须保留前一批次的记录与产物。

### 4.4 单一事实源

- 任务状态以 `task.md` 为准
- 通知文件状态是流程视图，与 task.md 冲突时以 task.md 为准
- 通知位置（inbox/working/done/failed）必须与文件内状态字段一致（反馈类通知除外，见 `rules/notice-schema.md`）
- `task.md` 只记录当前有效结论，历史批次结论与证据不得被覆盖或改写
- 证据冲突时的核对顺序见 `rules/authorization-boundary.md`

---

## 5. 通知文件规范

### 5.1 设计原则

通知文件是**任务分发指针**，不是任务副本。

### 5.2 命名

```
NOTICE-{project_id}-{task_id}-{attempt_id}.md
```

如 `NOTICE-PRJ-001-demo-TASK-002-implement-a1.md`。必须含 `project_id`（`task_id` 仅项目内唯一）与 `attempt_id`（执行批次）。

### 5.3 必填字段

| 字段 | 说明 | 示例 |
|------|------|------|
| `notice_id` | 通知唯一标识 | `NOTICE-PRJ-001-demo-TASK-002-implement-a1` |
| `notice_type` | dispatch / verify_result / return / alert / takeover_request | `dispatch` |
| `task_id` | 目标任务 ID（仅项目内唯一） | `TASK-002-implement` |
| `attempt_id` | 执行批次号，从 `a1` 递增 | `a1` |
| `sender_robot` | 发出通知的机器人 ID | `robot-planner` |
| `project_id` | 所属项目 ID | `PRJ-001-demo` |
| `project_path` | 项目相对路径 | `10-projects/PRJ-001-demo` |
| `task_path` | 任务相对路径 | `…/tasks/TASK-002-implement` |
| `result_path` | 交付物输出路径 | `…/tasks/TASK-002-implement/output` |
| `assignee_robot` | 被分配机器人 ID | `robot-coder` |
| `priority` | P0/P1/P2/P3 | `P1` |
| `status` | assigned/working/done/failed | `assigned` |
| `created_at` | ISO 8601 | `2026-05-23T10:30:00Z` |

### 5.4 禁止

- ❌ 粘贴完整任务正文替代路径引用
- ❌ 通知状态与所在子目录不一致（如 inbox/ 中通知状态为 done）
- ❌ 把机器人私有中间信息写成项目正式结论

---

## 6. 事件通知（Push）

### 6.1 动机

心跳轮询（pull）有延迟（分钟级），关键状态变更需要即时触达（push）。

### 6.2 事件类型

| 事件 | 触发方 | 投递目标 |
|------|--------|---------|
| `task.completed` | Executor | Gatekeeper inbox |
| `task.verified` | Gatekeeper | Planner inbox |
| `task.returned` | Gatekeeper | Executor inbox |
| `task.failed` | Executor | Planner inbox |
| `task.reassigned` | Planner | 新 Executor inbox |
| `task.blocked` | 任意 | Planner inbox |

### 6.3 格式

```
EVENT-{event_type}-{project_id}-{task_id}-{attempt_id}.md
```

如 `EVENT-task.completed-PRJ-001-demo-TASK-022-fix-brand-header-a1.md`。

事件通知仅携带元数据，不复制任务正文。详见 `rules/push-notification.md`。

---

## 7. 命名规范

| 对象 | 格式 | 示例 | 唯一性范围 |
|------|------|------|-----------|
| 项目 | `PRJ-{编号}-{短名}/` | `PRJ-001-pilot/` | 全局 |
| 任务 | `TASK-{编号}-{短名}/` | `TASK-001-hello/` | 项目内 |
| 通知 | `NOTICE-{project}-{task}-{attempt}.md` | `NOTICE-PRJ-001-pilot-TASK-001-hello-a1.md` | 机器人 inbox 内 |
| 事件 | `EVENT-{类型}-{project}-{task}-{attempt}.md` | `EVENT-task.completed-PRJ-001-pilot-TASK-001-hello-a1.md` | 机器人 inbox 内 |
| 机器人 | `robot-{短名}/` | `robot-coder/` | 全局 |

原则：先唯一再可读、先稳定再美观、统一 ASCII。详见 `rules/naming.md`。

---

## 8. 协作流程

### 8.1 项目初始化

```
Sponsor:
1. 创建 PRJ-xxx-name/ 目录结构
2. 编写 project.md（背景、目标、范围、验收标准）
3. 编写 infra.md（服务器、凭证等敏感信息）
4. 根据需要创建 TASK-xxx/ 和 task.md
5. 在 Planner inbox 投递项目级通知
6. 唤醒 Planner
```

### 8.2 任务分发

```
Planner（心跳或唤醒）:
1. 扫描自身 inbox/
2. 读取 OCWS 项目文档
3. 检查依赖（前置任务是否已归档）
4. 生成 NOTICE-*.md → Executor inbox/
5. 更新 task.md 状态 → assigned
```

### 8.3 任务执行

```
Executor（心跳或唤醒）:
1. 扫描自身 inbox/
2. mv NOTICE-*.md → working/（原子领取）
3. 读取 task.md + infra.md，了解任务上下文
4. 在 workspace/ 中执行
5. 产出写入 output/
6. 回写 task.md → done
7. 移动通知 → done/
8. 投递 EVENT-task.completed → Gatekeeper inbox
```

### 8.4 任务验收

```
Gatekeeper（心跳或唤醒）:
1. 扫描自身 inbox/ 中 EVENT-task.completed
2. 检查 done/ 中产出物，对照 task.md 验收标准（类型化标准见 verification-rules）
3. 出具结论并写验收报告 → {task}/logs/verification-report-{日期}.md
4. 通过：投递 EVENT-task.verified → Planner inbox/
   （归档由 Planner 执行：task.md → archived，closure_reason=verified）
5. 退回：投递 EVENT-task.returned（含 return_reason）→ Executor inbox/
   （Planner 据此走 done → assigned，重分配并递增 attempt_id）

Gatekeeper 不得自行把 task.md 标记为 archived，也不得移动项目 archive/。
```

---

## 9. 任务派发模式

### 9.1 默认：定向派发

Planner 选择目标 Executor，把通知投递到**该 Executor 自己的 inbox/**。

- Executor 只领取自己 inbox 中的通知，**不领取**其他 Executor 的通知
- Assistant-Executor 承接 Planner 独立分配给它的任务，不主动争抢另一执行者的通知
- 第一版默认每个 Executor 同时只处理一个任务，后续按真实需要调整

### 9.2 分发策略

| 策略 | 说明 |
|------|------|
| **负载均衡** | 轮流分配（round-robin） |
| **优先级路由** | P0/P1 优先分配给空闲者 |
| **亲和性** | 同项目后续任务优先给第一个执行者 |
| **故障接管** | 一方超时（如 >4h）→ 告警 + 人工批准 → Planner 重分配（见 task-lifecycle「超时与接管」） |

### 9.3 原子 `mv` 的正确用途

「原子抢锁」只解决**同一个机器人多个进程**同时领取同一份通知的问题：

- NFS 上 `mv` 是原子操作，同一文件不会被两个 `mv` 同时成功
- 通知文件移动到 `working/` 后即视为被领取
- 任何机器人在移动前须检查文件仍存在于 inbox/

它**不是**任务执行保证，也**不应该**被扩大解释为「多个机器人抢同一任务池」或「整个业务操作只执行一次」。

### 9.4 执行授权

- 同一任务同一时刻最多只有一个有效执行授权
- 重分配、退回重交、接管都必须显式撤销旧授权，并递增 `attempt_id`
- 同一任务的历史批次记录必须保留，不得覆盖

---

## 10. Sponsor 唤醒 Planner

### 10.1 双通道机制

| 通道 | 方式 | 延迟 |
|------|------|------|
| **轮询（pull）** | Planner cron 定时扫描 inbox/ | 分钟级 |
| **事件（push）** | Sponsor SSH 触发 Planner agent | 秒级 |

### 10.2 唤醒流程

```
Sponsor:
1. 更新 OCWS 文档（project.md / task.md）
2. 在 Planner inbox 投递简短通知
3. SSH 到 Planner 宿主机，执行:
   openclaw agent --agent main --message "来活了"
4. Planner 自行读取 OCWS 文档并分发
```

原则：**一句「来活了」足够**，不要口述任务细节。一切以 OCWS 文档为准。

---

## 版本历史

| 版本 | 日期 | 变更 |
|------|------|------|
| v1.0 | 2026-05-23 | 初始版本，6 项目实战验证 |
| v1.1 | 2026-09-22 | 消除规则歧义：默认定向派发、归档责任唯一化（Planner 归档）、新增 `done → assigned` 退回路径与 `closure_reason`、通知含 `project_id` + `attempt_id`、通知类型 `notice_type`、执行批次与超时接管、新增 `rules/authorization-boundary.md`（授权边界 + 证据分层） |

## 相关规则

| 文件 | 定义什么 |
|------|---------|
| `rules/naming.md` | 命名与唯一性键 |
| `rules/task-lifecycle.md` | 状态流转、批次、接管、恢复、归档责任 |
| `rules/notice-schema.md` | 通知字段与类型 |
| `rules/push-notification.md` | 事件通知与人类通知分级 |
| `rules/verification-rules.md` | 验收标准与红线 |
| `rules/authorization-boundary.md` | 授权边界、高影响操作确认、证据分层 |
| `rules/project-schema.md` | 项目文件与 `infra.md` |
| `rules/content-registry.md` | 内容发布追踪 |

## 许可证

MIT
