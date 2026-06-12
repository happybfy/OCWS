# 基础设施信息

> 本文件供机器人执行任务时查阅，避免反复解析 project.md。
> ⚠️ 含敏感凭证，不应提交到公开仓库。
> 格式：key = value / INI-style section

## 服务器

```ini
[server-{role}]
ip = 
user = 
password = 
role = 用途说明
```

## OSS / 对象存储

```ini
[oss-primary]
bucket = 
endpoint = 
access_key_id = 
access_key_secret = 
region = 
```

## 域名 / CDN

```ini
[domain]
primary = 
dns_provider = 
ssl = 
```

## 代理 / 网络

```ini
[proxy]
name = url
```

## 项目路径

```ini
[paths]
project_root = 
build_output = 
deploy_script = 
```
