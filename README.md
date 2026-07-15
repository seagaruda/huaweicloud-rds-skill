# 华为云 RDS Agent Skill

一个面向所有 AI 平台的华为云 RDS 操作技能文件，让任何 AI Agent 通过自然语言完成 RDS 数据库的全生命周期管理。

## 支持的 AI 平台

| 平台 | 使用方式 |
|------|---------|
| **Claude Projects** | 将 `SKILL.md` 内容粘贴到 Project Instructions |
| **Cursor** | 保存为 `.cursor/rules/huaweicloud-rds.mdc` |
| **GitHub Copilot** | 保存为 `.github/copilot-instructions.md` |
| **ChatGPT / Custom GPT** | 粘贴到 System Prompt 或 Instructions |
| **Workbuddy / Dify / FastGPT** | 作为系统提示词或知识库文档导入 |
| **Hermes Agent** | 放入 `~/.hermes/skills/cloud/huaweicloud-rds/SKILL.md` |
| **任意平台** | 将 `SKILL.md` 全文粘贴为 System Prompt 即可 |

## 覆盖功能

| 模块 | 操作 |
|------|------|
| **实例管理** | 创建、查询列表/详情、重启、变更规格、扩容磁盘、删除、查询任务进度 |
| **备份恢复** | 自动备份策略、手动备份、查询备份、PITR 按时间点恢复、按备份恢复、删除备份 |
| **账号管理** | 创建账号、授权数据库、查询账号、重置密码、删除账号 |
| **参数管理** | 查询参数、修改参数（含常用参数速查表） |
| **日志查询** | 慢日志查询与分析、错误日志查询 |
| **安全配置** | SSL 开关、端口修改 |

支持引擎：**MySQL**、**PostgreSQL**、**SQL Server**  
API 版本：**RDS v3**（华为云推荐版本）

## 快速使用

### Claude Projects

1. 打开 [Claude Projects](https://claude.ai/projects)
2. 进入项目设置 → Instructions
3. 复制 `SKILL.md` 全文粘贴进去
4. 对话示例：
   > "帮我在 cn-north-4 创建一个 MySQL 8.0 主备实例，4核8G，100G SSD"

### Cursor

```bash
# 在项目根目录执行
mkdir -p .cursor/rules
curl -o .cursor/rules/huaweicloud-rds.mdc \
  https://raw.githubusercontent.com/seagaruda/huaweicloud-rds-skill/main/SKILL.md
```

### Hermes Agent

```bash
mkdir -p ~/.hermes/skills/cloud/huaweicloud-rds
curl -o ~/.hermes/skills/cloud/huaweicloud-rds/SKILL.md \
  https://raw.githubusercontent.com/seagaruda/huaweicloud-rds-skill/main/SKILL.md
```

## 对话示例

```
用户：帮我查一下所有 MySQL 实例的状态
用户：把生产库恢复到今天上午10点
用户：给 app_user 创建一个只读账号，只能访问 orders 数据库
用户：最近数据库很慢，帮我查一下慢 SQL
用户：把 max_connections 改成 800
用户：帮我备份一下，待会儿要做变更
```

## 官方文档

- [RDS 产品介绍](https://support.huaweicloud.com/productdesc-rds/rds_01_0001.html)
- [RDS API 参考](https://support.huaweicloud.com/api-rds/rds_01_0001.html)
