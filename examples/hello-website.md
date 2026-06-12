# 示例项目：Hello World 网站部署

> 这是一个 OCWS 项目的完整结构示例，演示如何组织文件和任务。

---

## 项目目录

```
10-projects/PRJ-001-hello-website/
├── project.md          # 项目说明
├── infra.md            # 基础设施（含凭证，勿公开）
├── shared/
│   ├── input/          # 上游输入：设计稿、文案
│   │   └── logo.png
│   ├── output/         # 下游输出：构建产物
│   └── refs/           # 参考资料
│       └── design-spec.md
├── tasks/
│   ├── TASK-001-setup-server/
│   │   ├── task.md
│   │   ├── input/
│   │   ├── workspace/
│   │   ├── output/
│   │   └── logs/
│   └── TASK-002-deploy-frontend/
│       ├── task.md
│       ├── input/
│       ├── workspace/
│       ├── output/
│       └── logs/
└── archive/
```

## project.md

```markdown
# 项目说明

## 基本信息

- 项目 ID：`PRJ-001-hello-website`
- 项目名称：Hello World 网站部署
- 当前状态：`active`
- 优先级：`P1`
- 项目负责人：Zhang San
- 创建时间：`2026-05-20T10:00:00Z`

## 项目背景

公司需要一个简单的品牌展示网站。设计稿和文案已准备就绪，需要部署到生产服务器。

## 项目目标

- 必须达成：网站可通过域名访问，支持 HTTPS
- 希望达成：CDN 加速、访问统计

## 任务拆解

- TASK-001：服务器环境搭建（Nginx + SSL）
- TASK-002：前端页面部署

依赖链：TASK-001 → TASK-002（串行）

## 验收标准

- 验收项 1：访问 https://example.com 返回 200
- 验收项 2：SSL 证书有效
```

## TASK-002 的 task.md

```markdown
# 任务说明

## 基本信息

- 任务 ID：`TASK-002-deploy-frontend`
- 所属项目：`PRJ-001-hello-website`
- 当前状态：`assigned`
- 优先级：`P1`
- 责任机器人：`robot-coder`
- 依赖：`TASK-001-setup-server`（已归档 ✅）

## 任务背景

服务器环境已就绪，需要将前端页面部署到 Nginx 目录。

## 任务目标

将 shared/output/ 中的构建产物部署到目标服务器的 Nginx web root。

## 验收标准

- 验收项 1：首页可访问，显示 Hello World
- 验收项 2：静态资源（CSS/JS/图片）加载正常
```

## 通知文件示例

在 `20-robots/robot-coder/inbox/NOTICE-TASK-002-deploy-frontend.md`：

```markdown
# 通知：NOTICE-TASK-002-deploy-frontend

## 基本信息

- notice_id：`NOTICE-TASK-002-deploy-frontend`
- task_id：`TASK-002-deploy-frontend`
- project_id：`PRJ-001-hello-website`
- assignee_robot：`robot-coder`
- priority：`P1`
- status：`assigned`

## 路径信息

- project_path：`10-projects/PRJ-001-hello-website`
- task_path：`10-projects/PRJ-001-hello-website/tasks/TASK-002-deploy-frontend`
- result_path：`10-projects/PRJ-001-hello-website/tasks/TASK-002-deploy-frontend/output`

## 时间信息

- created_at：`2026-05-20T11:00:00Z`
```
