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
9. [双执行者模式](#9-双执行者模式)
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
| **原子操作为锁** | NFS `mv` 原子操作保证任务领取互斥 |
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
│   │   └── push-notification.md
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
| **Gatekeeper** | 巡检 working/、验收 done/、决定归档/退回 | 验收报告、EVENT 文件 | 不修改产出、不代执行 |

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
                      ↓         ↑
                   failed ─────┘
                      ↓
              assigned（重分配）或 archived（终止）
```

### 4.2 状态含义

| 状态 | 含义 | 通知位置 |
|------|------|---------|
| `new` | 已创建，未分配 | — |
| `assigned` | 已分配，待领取 | `inbox/` |
| `working` | 执行中 | `working/` |
| `done` | 已完成，待验收 | `done/` |
| `failed` | 执行失败 | `failed/` |
| `archived` | 已闭环 | 项目 `archive/` |

### 4.3 非法流转（禁止）

- `new → done/working`（跳过分配）
- `assigned → done`（跳过执行）
- `done → working`（回退）
- `archived → working/assigned`（归档复活）

### 4.4 单一事实源

- 任务状态以 `task.md` 为准
- 通知文件状态是流程视图，与 task.md 冲突时以 task.md 为准
- 通知位置（inbox/working/done/failed）必须与文件内状态字段一致

---

## 5. 通知文件规范

### 5.1 设计原则

通知文件是**任务分发指针**，不是任务副本。

### 5.2 命名

```
NOTICE-TASK-{编号}-{短名}.md
```

### 5.3 必填字段

| 字段 | 说明 | 示例 |
|------|------|------|
| `notice_id` | 通知唯一标识 | `NOTICE-TASK-002-implement` |
| `task_id` | 目标任务 ID | `TASK-002-implement` |
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
EVENT-{event_type}-{task_id}.md
```

事件通知仅携带元数据，不复制任务正文。详见 `rules/push-notification.md`。

---

## 7. 命名规范

| 对象 | 格式 | 示例 | 唯一性范围 |
|------|------|------|-----------|
| 项目 | `PRJ-{编号}-{短名}/` | `PRJ-001-pilot/` | 全局 |
| 任务 | `TASK-{编号}-{短名}/` | `TASK-001-hello/` | 项目内 |
| 通知 | `NOTICE-{任务目录名}.md` | `NOTICE-TASK-001-hello.md` | 机器人 inbox 内 |
| 事件 | `EVENT-{类型}-{任务ID}.md` | `EVENT-task.completed-TASK-001.md` | 机器人 inbox 内 |
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
2. 检查 done/ 中产出物
3. 对照 task.md 验收标准
4. 通过：task.md → archived，移动通知至项目 archive/
5. 不通过：移动通知回 Executor inbox/，投递 EVENT-task.returned
```

---

## 9. 双执行者模式

### 9.1 原子抢锁

两个 Executor 共享 `20-robots/` 下的独立 inbox/，通过 `mv` 原子操作争夺通知文件。

### 9.2 分发策略

| 策略 | 说明 |
|------|------|
| **负载均衡** | 轮流分配（round-robin） |
| **优先级路由** | P0/P1 优先分配给空闲者 |
| **亲和性** | 同项目后续任务优先给第一个执行者 |
| **故障接管** | 一方超时（如 >4h）→ Planner 重分配 |

### 9.3 互斥保证

- NFS 上 `mv` 是原子操作，同一文件不会被两个 `mv` 同时成功
- 通知文件移动到 `working/` 后即视为被领取
- 任何机器人在移动前须检查文件仍存在于 inbox/

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

## 许可证

MIT
