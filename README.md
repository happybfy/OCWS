# OCWS — Open Claw Workspace Specification

> **目录 + Markdown + 通知文件 = 多机器人协作框架**
>
> 不需要 API、消息队列或数据库。NFS 共享 + 约定即协议。

[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Version](https://img.shields.io/badge/version-v1.0-green.svg)](SPEC.md)

---

## 这是什么？

OCWS 是一套**轻量级多机器人协作工作空间规范**。它定义了一个共享目录结构、一套命名约定、任务状态流转规则和通知文件格式，让多个 AI Agent（机器人）能够在同一个文件系统上自主协作，无需中心调度服务。

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
| **Gatekeeper** | 门禁者 | 巡检 working/、验收 done/、决定归档/退回 | 不修改产出、不代执行 |

## 任务状态流转

```
new → assigned → working → done → archived
                      ↓         ↑
                   failed ─────┘
                      ↓
              assigned（重分配）或 archived（终止）
```

## 实战验证

OCWS 已在 60 个真实项目中运行，涵盖：

- 网站部署（WordPress + OSS）
- 内容站点开发（10+ 页面、多模板）
- ERPNext 定制开发（自定义 App + 前后端 + Bug修复）
- 批量运维操作（bench console 仓库转移）

累计 150+ 个任务通过 OCWS 自动分发执行，未出现任务丢失或事实源不一致。

## 文档导航

| 文档 | 内容 |
|------|------|
| [SPEC.md](SPEC.md) | 完整规范：目录结构、角色、流程 |
| [rules/](rules/) | 命名、任务流转、通知格式、项目结构等规则 |
| [templates/](templates/) | 项目、任务、通知、事件的标准模板 |
| [lessons-learned.md](lessons-learned.md) | 实战经验与踩坑记录 |

## 依赖

- 共享文件系统（NFS / SMB / 本地目录均可）
- 支持 `mv` 原子操作（任务领取互斥的基础）
- 各机器人能读写共享目录

## 许可证

MIT
