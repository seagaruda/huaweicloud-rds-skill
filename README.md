# huaweicloud-rds-skill

华为云 RDS（云数据库）Agent Skill，让 AI Agent 通过自然语言完成 RDS 全生命周期管理。

## 什么是 Agent Skill？

这是一个面向 [Hermes Agent](https://hermes-agent.nousresearch.com) 的技能文件（SKILL.md）。当用户用自然语言发出数据库相关指令时，Agent 自动加载本 Skill，获取 API 调用方式、参数格式、注意事项，从而完成操作。

**无需手写代码，无需查文档** —— 直接对 Agent 说：
- "帮我在华北四区创建一个 MySQL 8.0 主备实例，4核8G"
- "把生产库恢复到昨天晚上10点"
- "查一下最近有哪些慢 SQL"
- "给 app_user 授权访问 myapp_db"

## 覆盖操作

| 模块 | 支持的操作 |
|------|-----------|
| 实例管理 | 创建、查询列表、查询详情、重启、变更规格、扩容磁盘、删除 |
| 备份恢复 | 设置自动备份策略、创建手动备份、查询备份、PITR 恢复、按备份恢复、删除备份 |
| 账号管理 | 创建账号、授权/撤权、查询账号、重置密码、删除账号 |
| 参数管理 | 查询参数、修改参数 |
| 日志查询 | 慢日志查询、错误日志查询 |
| 安全配置 | SSL 开关、端口修改 |

支持引擎：**MySQL**、**PostgreSQL**、**SQL Server**

## 使用方式

将 `SKILL.md` 放入 Hermes Agent 的 skills 目录：

```bash
mkdir -p ~/.hermes/skills/cloud/huaweicloud-rds
cp SKILL.md ~/.hermes/skills/cloud/huaweicloud-rds/
```

然后在 Agent 对话中直接用自然语言描述 RDS 操作即可。

## 官方文档

- [RDS 产品介绍](https://support.huaweicloud.com/productdesc-rds/rds_01_0001.html)
- [RDS API 参考](https://support.huaweicloud.com/api-rds/rds_01_0001.html)
- [RDS 快速入门](https://support.huaweicloud.com/qs-rds/rds_02_0008.html)
