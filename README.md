# OCWS — Open Claw Workspace Specification

> **目录 + Markdown + 通知文件 = 多机器人协作框架**
>
> 不需要 API、消息队列或数据库。NFS 共享 + 约定即协议。

[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Version](https://img.shields.io/badge/version-v1.1-green.svg)](SPEC.md)

---

## 这是什么？

OCWS 是一套**轻量级多机器人协作工作空间规范**。它定义了一个共享目录结构、一套命名约定、任务状态流转规则和通知文件格式，让多个 AI Agent（机器人）能够在同一个文件系统上自主协作，无需中心调度服务。

## 适用前提（边界声明）

OCWS 面向**一人公司 + 少量角色化机器人**的场景，明确不做以下承诺：

- 一个主要人类管理者，少量职责清晰的机器人
- 低并发、非实时调度，**允许人工处理异常**（不追求完全无人值守）
- 不以大规模协作、复杂多租户为目标
- 小规模不等于所有任务都低风险：高影响 / 不可逆操作另设执行前确认（见 `rules/authorization-boundary.md`）

这条边界是后续功能取舍的依据：遇到反复出现的具体问题再增加局部工具，不预先为低频异常建设平台。

## 核心设计哲学

| 原则 | 含义 |
|------|------|
| **简单 > 完备** | 不引入新协议，用目录和 Markdown 就够了 |
| **约定 > 工具** | 规则写在文档里，不依赖定制化软件 |
| **文档 > 消息** | `project.md` / `task.md` 是唯一事实源，通知只是指针 |

## 快速理解

```
workspace/
├── 00-system/          # 规范层：规则、模板
├── 10-projects/        # 事实源：项目上下文与任务实体
│   └── PRJ-001-demo/
│       ├── project.md  # ← 项目唯一事实源
│       ├── infra.md    # ← 基础设施信息
│       └── tasks/
│           └── TASK-001-hello/
│               ├── task.md    # ← 任务唯一事实源
│               ├── input/ output/ workspace/ logs/
└── 20-robots/          # 工作入口：各机器人 inbox/working/done/failed
    └── robot-coder/
        ├── profile.md
        ├── inbox/      # 待领取任务通知
        ├── working/    # 执行中
        ├── done/       # 已完成
        └── failed/     # 执行失败
```

## 角色体系

| 角色 | 代号 | 职责 | 不越界 |
|------|------|------|--------|
| **Sponsor** | 项目发起人 | 建项目、定任务、保障基础设施 | 不分发、不执行 |
| **Planner** | 计划者 | 读 inbox、拆任务、分发通知、重分配 failed | 不执行、不验收 |
| **Executor** | 执行者 | 原子领取、执行、产出、回写 task.md | 不规划、不验收 |
| **Gatekeeper** | 门禁者 | 巡检 working/、验收 done/、出具归档/退回**结论** | 不修改产出、不代执行、不自行归档 |

## 任务状态流转

```
new → assigned → working → done → archived
                      ↓         │
                   failed       └─→ assigned（验收退回，新建批次）
                      ↓
              assigned（重分配）或 archived（终止）
```

> `done → assigned` 是**验收退回**（已交付但不合格）；`failed → assigned` 是**执行失败后重分配**。两者都必须保留前一批次的记录与产物。

## 实战验证

OCWS 已在 60 个真实项目中运行，涵盖：

- 网站部署（WordPress + OSS）
- 内容站点开发（10+ 页面、多模板）
- ERPNext v15 部署（分离架构：DB + Redis + Web）
- ERPNext v15 应用部署（POS Awesome V15）
- ERPNext 定制开发（自定义 App + 前后端 + Bug修复）
- 公网映射服务frpc部署（服务端+客户端）
- 批量运维操作（bench console 仓库转移）
- WordPress二次开发
- 短视频自动化剪辑

累计 150+ 个任务通过 OCWS 分发执行。**注意**：早期运行中已发现过事实源被改写、批次证据缺失的案例（见 `lessons-learned.md`），因此本规范不再声称「从未出现事实源不一致」——记录可信度靠 `rules/authorization-boundary.md` 的证据分层与批次纪律保证，而不是靠结论。

## 文档导航

| 文档 | 内容 |
|------|------|
| [SPEC.md](SPEC.md) | 完整规范：目录结构、角色、流程 |
| [rules/](rules/) | 命名、任务流转、通知格式、验收规范、授权边界、项目结构等规则 |
| [templates/](templates/) | 项目、任务、通知、事件的标准模板 |
| [lessons-learned.md](lessons-learned.md) | 实战经验与踩坑记录 |

### 规则清单

| 文件 | 定义什么 |
|------|---------|
| [rules/naming.md](rules/naming.md) | 项目/任务/通知/机器人命名与唯一性键 |
| [rules/task-lifecycle.md](rules/task-lifecycle.md) | 状态流转、执行批次、超时接管、中断恢复、归档责任 |
| [rules/notice-schema.md](rules/notice-schema.md) | 通知字段、通知类型、位置一致性 |
| [rules/push-notification.md](rules/push-notification.md) | 事件通知与人类通知分级 |
| [rules/verification-rules.md](rules/verification-rules.md) | 门禁者验收标准与红线 |
| [rules/authorization-boundary.md](rules/authorization-boundary.md) | 授权边界、高影响操作确认、证据分层 |
| [rules/project-schema.md](rules/project-schema.md) | 项目文件与 `infra.md` 规范 |
| [rules/content-registry.md](rules/content-registry.md) | 内容发布状态追踪 |

## 依赖

- 共享文件系统（NFS / SMB / 本地目录均可）
- 支持 `mv` 原子操作（用于**同一机器人多进程**的领取互斥，不是任务只执行一次的保证）
- 各机器人能读写共享目录

## 许可证

MIT
