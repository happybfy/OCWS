# 项目文件规范

## 必选文件

每个项目必须包含以下文件：

| 文件 | 用途 |
|------|------|
| `project.md` | 项目说明（背景、目标、范围、干系人、风险、验收标准） |
| `infra.md` | 基础设施信息（服务器、OSS、域名、代理、路径等敏感/关键信息） |

## infra.md 规范

**目的：** 将项目相关的服务器 IP、密码、OSS 凭证、域名等关键信息集中存放，机器人执行任务时直接读取，避免反复解析 `project.md`，减少 token 消耗。

**位置：** `10-projects/PRJ-xxx-name/infra.md`

**格式要求：**

- 使用 INI-style section 分组
- key = value，一行一条
- 机器人可通过 `grep` 或 section 名快速定位

**模板：** `templates/infra-template.md`

**示例结构：**

```ini
## 服务器

[server-dev]
ip = 192.168.1.100
user = admin
password = <secret>
role = 开发服务器

## OSS

[oss-primary]
bucket = my-bucket
endpoint = oss-cn-example.aliyuncs.com
access_key_id = <key>
access_key_secret = <secret>

## 域名

[domain]
primary = example.com

## 路径

[paths]
project_root = /opt/project
```

**约束：**

- `infra.md` 包含敏感信息，**不在通知文件中引用其内容**
- 机器人执行时直接读取 `infra.md`，不复制凭证到其他地方
- 项目创建时由 Sponsor 或 Planner 同步创建
- 不应提交到公开代码仓库
