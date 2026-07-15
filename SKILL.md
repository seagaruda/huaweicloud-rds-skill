---
name: huaweicloud-rds
description: Use when the user wants to manage Huawei Cloud RDS (Relational Database Service) instances via natural language — create, query, modify, delete instances; manage backups and restores (PITR); manage database accounts and permissions; query slow/error logs; modify parameters. Covers MySQL, PostgreSQL, and SQL Server engines via RDS v3 API.
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [huaweicloud, rds, database, mysql, postgresql, sqlserver, cloud, api]
    related_skills: []
---

# 华为云 RDS 技能

## Overview

本 Skill 让 AI Agent 通过自然语言指令，使用华为云 RDS v3 REST API 完成关系型数据库的全生命周期管理。支持 MySQL、PostgreSQL、SQL Server 三种引擎，涵盖实例管理、备份恢复、账号权限、日志查询、参数配置等所有核心操作。

API 基础路径：`https://rds.{region}.myhuaweicloud.com/v3/{project_id}`  
认证方式：请求头携带 `X-Auth-Token`（IAM Token）或使用 AK/SK 签名。

## When to Use

- 用户说"帮我创建一个 RDS 实例"、"查一下我的数据库列表"
- 用户说"备份这个实例"、"把数据库恢复到昨天下午三点"
- 用户说"给 RDS 创建一个只读账号"、"查查最近的慢 SQL"
- 用户说"把实例的 max_connections 改成 500"、"重启数据库"
- 用户说"删掉那个测试实例"、"扩容磁盘到 500G"

不适用于：
- 在数据库内执行 SQL 语句（需直接连接数据库，本 Skill 不涉及）
- 华为云 GaussDB / DDS / DCS 等其他数据库产品

## 环境准备

在执行任何操作前，需要以下信息（向用户确认或从环境变量读取）：

| 变量 | 说明 | 示例 |
|------|------|------|
| `HW_TOKEN` | IAM Token（24h 有效） | `MIIEowIBAA...` |
| `HW_REGION` | 区域 ID | `cn-north-4` |
| `HW_PROJECT_ID` | 项目 ID | `0a9b2c3d4e5f...` |
| `RDS_ENDPOINT` | RDS 接入点（可从区域推导） | `rds.cn-north-4.myhuaweicloud.com` |

**获取 IAM Token：**

```bash
TOKEN=$(curl -s -X POST \
  https://iam.{region}.myhuaweicloud.com/v3/auth/tokens \
  -H "Content-Type: application/json" \
  -d '{
    "auth": {
      "identity": {
        "methods": ["password"],
        "password": {
          "user": {
            "name": "YOUR_IAM_USERNAME",
            "password": "YOUR_PASSWORD",
            "domain": { "name": "YOUR_ACCOUNT_NAME" }
          }
        }
      },
      "scope": { "project": { "name": "YOUR_REGION" } }
    }
  }' -D - -o /dev/null | grep -i x-subject-token | awk '{print $2}' | tr -d '\r')
echo "Token: $TOKEN"
```

**获取 Project ID（从控制台或 API）：**

```bash
curl -s https://iam.{region}.myhuaweicloud.com/v3/auth/projects \
  -H "X-Auth-Token: $TOKEN" | python3 -c "
import sys, json
data = json.load(sys.stdin)
for p in data['projects']:
    print(p['id'], p['name'])
"
```

---

## 实例管理

### 创建实例

**触发词：** "创建 RDS"、"新建数据库实例"、"建一个 MySQL"

向用户收集以下必填信息：实例名称、数据库类型（MySQL/PostgreSQL/SQLServer）、版本、规格、磁盘大小、VPC/子网/安全组 ID、区域/可用区、管理员密码。

```bash
curl -s -X POST \
  https://${RDS_ENDPOINT}/v3/${HW_PROJECT_ID}/instances \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: ${HW_TOKEN}" \
  -d '{
    "name": "my-rds-mysql",
    "datastore": {
      "type": "MySQL",
      "version": "8.0"
    },
    "ha": {
      "mode": "Ha",
      "replication_mode": "semisync"
    },
    "password": "Test@12345678",
    "flavor_ref": "rds.mysql.c2.large.ha",
    "volume": {
      "type": "ULTRAHIGH",
      "size": 100
    },
    "region": "cn-north-4",
    "availability_zone": "cn-north-4a,cn-north-4b",
    "vpc_id": "YOUR_VPC_ID",
    "subnet_id": "YOUR_SUBNET_ID",
    "security_group": { "id": "YOUR_SG_ID" }
  }'
```

关键参数说明：
- `ha.mode`：`Ha`（主备）| `Single`（单机）| `Replica`（只读，需指定 `replica_of_id`）
- `ha.replication_mode`：MySQL → `semisync`；PostgreSQL → `async`；SQLServer → `sync`
- `flavor_ref`：通过「查询规格」接口获取，格式如 `rds.mysql.c2.large.ha`
- `volume.type`：`ULTRAHIGH`（SSD）| `LOCALSSD`（本地 SSD）| `CLOUDSSD`（云 SSD）
- 主备实例 `availability_zone` 填两个 AZ，逗号分隔

创建后返回 `job_id`，用「查询任务」接口轮询进度直到 `status: Completed`。

### 查询任务进度

```bash
curl -s "https://${RDS_ENDPOINT}/v3/${HW_PROJECT_ID}/jobs?id={job_id}" \
  -H "X-Auth-Token: ${HW_TOKEN}"
```

### 查询实例列表

**触发词：** "列出所有 RDS"、"查我的数据库实例"

```bash
# 查询所有实例
curl -s "https://${RDS_ENDPOINT}/v3/${HW_PROJECT_ID}/instances" \
  -H "X-Auth-Token: ${HW_TOKEN}" | python3 -m json.tool

# 按类型/引擎过滤
curl -s "https://${RDS_ENDPOINT}/v3/${HW_PROJECT_ID}/instances?datastore_type=MySQL&type=Ha" \
  -H "X-Auth-Token: ${HW_TOKEN}"

# 分页（limit 最大 100）
curl -s "https://${RDS_ENDPOINT}/v3/${HW_PROJECT_ID}/instances?offset=0&limit=20" \
  -H "X-Auth-Token: ${HW_TOKEN}"
```

`type` 可选：`Single` | `Ha` | `Replica` | `Enterprise`

### 查询实例详情

```bash
curl -s "https://${RDS_ENDPOINT}/v3/${HW_PROJECT_ID}/instances/{instance_id}" \
  -H "X-Auth-Token: ${HW_TOKEN}" | python3 -m json.tool
```

重要字段：`status`（ACTIVE/BUILD/FAILED/STORAGE FULL）、`private_ips`（内网连接地址）、`port`

### 重启实例

**触发词：** "重启数据库"、"重启 RDS 实例"

```bash
curl -s -X POST \
  https://${RDS_ENDPOINT}/v3/${HW_PROJECT_ID}/instances/{instance_id}/action \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: ${HW_TOKEN}" \
  -d '{"restart": {}}'
```

⚠️ 重启会造成业务中断，执行前必须向用户二次确认。

### 变更规格（纵向扩容）

**触发词：** "升级配置"、"扩容 CPU/内存"

```bash
curl -s -X POST \
  https://${RDS_ENDPOINT}/v3/${HW_PROJECT_ID}/instances/{instance_id}/action \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: ${HW_TOKEN}" \
  -d '{
    "resize_flavor": {
      "spec_code": "rds.mysql.c2.xlarge.ha"
    }
  }'
```

新规格码通过「查询数据库规格」接口获取，变更期间实例重启，有短暂中断。

### 扩容磁盘

**触发词：** "扩容磁盘"、"增加存储空间"

```bash
curl -s -X POST \
  https://${RDS_ENDPOINT}/v3/${HW_PROJECT_ID}/instances/{instance_id}/action \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: ${HW_TOKEN}" \
  -d '{
    "enlarge_volume": {
      "size": 200
    }
  }'
```

磁盘只能扩大不能缩小，`size` 需大于当前值，单位 GB，且为 10 的整数倍。

### 删除实例

**触发词：** "删除实例"、"销毁数据库"

```bash
curl -s -X DELETE \
  https://${RDS_ENDPOINT}/v3/${HW_PROJECT_ID}/instances/{instance_id} \
  -H "X-Auth-Token: ${HW_TOKEN}"
```

⚠️ 删除操作不可逆，会同时删除所有自动备份。执行前**必须**向用户明确确认实例名称，并提醒备份情况。

### 查询可用规格

```bash
curl -s "https://${RDS_ENDPOINT}/v3/${HW_PROJECT_ID}/flavors/{db_type}?version_name={version}" \
  -H "X-Auth-Token: ${HW_TOKEN}"
# 示例：db_type=MySQL，version_name=8.0
```

---

## 备份与恢复

### 设置自动备份策略

**触发词：** "开启自动备份"、"设置备份保留天数"

```bash
curl -s -X PUT \
  https://${RDS_ENDPOINT}/v3/${HW_PROJECT_ID}/instances/{instance_id}/backups/policy \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: ${HW_TOKEN}" \
  -d '{
    "backup_policy": {
      "keep_days": 7,
      "start_time": "01:00-02:00",
      "period": "1,2,3,4,5,6,7"
    }
  }'
```

`period` 为每周备份天数（1=周一...7=周日），`keep_days` 范围 1-732 天。

### 创建手动备份

**触发词：** "立即备份"、"创建快照"

```bash
curl -s -X POST \
  https://${RDS_ENDPOINT}/v3/${HW_PROJECT_ID}/backups \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: ${HW_TOKEN}" \
  -d '{
    "instance_id": "{instance_id}",
    "name": "manual-backup-$(date +%Y%m%d%H%M)",
    "description": "手动备份"
  }'
```

### 查询备份列表

```bash
curl -s "https://${RDS_ENDPOINT}/v3/${HW_PROJECT_ID}/backups?instance_id={instance_id}" \
  -H "X-Auth-Token: ${HW_TOKEN}" | python3 -m json.tool
```

`status`：`BUILDING`（备份中）| `COMPLETED`（完成）| `FAILED`（失败）

### 查询可恢复时间段（PITR 前必查）

```bash
curl -s "https://${RDS_ENDPOINT}/v3/${HW_PROJECT_ID}/instances/{instance_id}/restore-time?date=2024-01-01" \
  -H "X-Auth-Token: ${HW_TOKEN}"
# 返回 start_time/end_time 为毫秒级 Unix 时间戳
```

### 恢复到新实例（按时间点 PITR）

**触发词：** "恢复到昨天下午三点"、"PITR 恢复"

```bash
curl -s -X POST \
  https://${RDS_ENDPOINT}/v3/${HW_PROJECT_ID}/instances \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: ${HW_TOKEN}" \
  -d '{
    "name": "restored-instance",
    "source": {
      "instance_id": "{source_instance_id}",
      "type": "timestamp",
      "restore_time": 1704067200000
    },
    "target": {
      "flavor_ref": "rds.mysql.c2.large",
      "volume": { "type": "ULTRAHIGH", "size": 100 },
      "availability_zone": "cn-north-4a",
      "vpc_id": "YOUR_VPC_ID",
      "subnet_id": "YOUR_SUBNET_ID",
      "security_group": { "id": "YOUR_SG_ID" }
    }
  }'
```

将用户提供的时间（如"昨天下午3点"）转换为毫秒时间戳：`date -d "yesterday 15:00" +%s%3N`

### 按备份文件恢复到新实例

```bash
curl -s -X POST \
  https://${RDS_ENDPOINT}/v3/${HW_PROJECT_ID}/instances \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: ${HW_TOKEN}" \
  -d '{
    "name": "restored-from-backup",
    "source": {
      "instance_id": "{source_instance_id}",
      "type": "backup",
      "backup_id": "{backup_id}"
    },
    "target": {
      "flavor_ref": "rds.mysql.c2.large",
      "volume": { "type": "ULTRAHIGH", "size": 100 },
      "availability_zone": "cn-north-4a",
      "vpc_id": "YOUR_VPC_ID",
      "subnet_id": "YOUR_SUBNET_ID",
      "security_group": { "id": "YOUR_SG_ID" }
    }
  }'
```

### 删除手动备份

```bash
curl -s -X DELETE \
  https://${RDS_ENDPOINT}/v3/${HW_PROJECT_ID}/backups/{backup_id} \
  -H "X-Auth-Token: ${HW_TOKEN}"
```

---

## 数据库账号管理

### 创建账号

**触发词：** "建一个数据库用户"、"创建只读账号"

**MySQL / SQL Server：**

```bash
curl -s -X POST \
  https://${RDS_ENDPOINT}/v3/${HW_PROJECT_ID}/instances/{instance_id}/db_user \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: ${HW_TOKEN}" \
  -d '{
    "name": "app_user",
    "password": "App@12345678",
    "comment": "应用只读账号"
  }'
```

**PostgreSQL：**

```bash
curl -s -X POST \
  https://${RDS_ENDPOINT}/v3/${HW_PROJECT_ID}/instances/{instance_id}/db_user \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: ${HW_TOKEN}" \
  -d '{
    "name": "app_user",
    "password": "App@12345678",
    "comment": "应用账号"
  }'
```

密码规则：8-32 位，含大小写字母、数字、特殊字符（`!@#$%^*-_=+?,`）中至少 3 类。

### 授权账号访问数据库（MySQL）

**触发词：** "给用户授权"、"让 app_user 能访问 mydb"

```bash
curl -s -X POST \
  https://${RDS_ENDPOINT}/v3/${HW_PROJECT_ID}/instances/{instance_id}/db_user/privilege \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: ${HW_TOKEN}" \
  -d '{
    "user_name": "app_user",
    "databases": [
      { "name": "myapp_db", "readonly": false }
    ]
  }'
```

`readonly: true` 为只读，`readonly: false` 为读写。

### 查询账号列表

```bash
curl -s "https://${RDS_ENDPOINT}/v3/${HW_PROJECT_ID}/instances/{instance_id}/db_user/detail?page=1&limit=100" \
  -H "X-Auth-Token: ${HW_TOKEN}" | python3 -m json.tool
```

### 重置账号密码

```bash
curl -s -X POST \
  https://${RDS_ENDPOINT}/v3/${HW_PROJECT_ID}/instances/{instance_id}/db_user/resetpwd \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: ${HW_TOKEN}" \
  -d '{
    "name": "app_user",
    "password": "NewPass@12345"
  }'
```

### 删除账号

```bash
curl -s -X DELETE \
  https://${RDS_ENDPOINT}/v3/${HW_PROJECT_ID}/instances/{instance_id}/db_user/{user_name} \
  -H "X-Auth-Token: ${HW_TOKEN}"
```

---

## 参数管理

### 查询实例当前参数

**触发词：** "查看参数配置"、"max_connections 现在是多少"

```bash
curl -s "https://${RDS_ENDPOINT}/v3/${HW_PROJECT_ID}/instances/{instance_id}/configurations" \
  -H "X-Auth-Token: ${HW_TOKEN}" | python3 -m json.tool
```

### 修改实例参数

**触发词：** "把 max_connections 改成 500"、"调整慢查询阈值"

```bash
curl -s -X PUT \
  https://${RDS_ENDPOINT}/v3/${HW_PROJECT_ID}/instances/{instance_id}/configurations \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: ${HW_TOKEN}" \
  -d '{
    "values": {
      "max_connections": "500",
      "long_query_time": "1"
    }
  }'
```

常用参数（MySQL）：

| 参数 | 说明 | 默认值 |
|------|------|--------|
| `max_connections` | 最大连接数 | 151 |
| `long_query_time` | 慢查询阈值（秒） | 10 |
| `innodb_buffer_pool_size` | InnoDB 缓冲池大小 | 取决于规格 |
| `character_set_server` | 服务器字符集 | utf8mb4 |
| `time_zone` | 时区 | +08:00 |

部分参数修改后需要重启实例才能生效，接口响应中 `restart_required: true` 时需告知用户。

---

## 日志查询

### 查询慢日志

**触发词：** "查慢 SQL"、"有哪些慢查询"、"最近的性能问题"

```bash
curl -s "https://${RDS_ENDPOINT}/v3/${HW_PROJECT_ID}/instances/{instance_id}/slowlog?\
start_date=2024-01-01T00:00:00%2B0800\
&end_date=2024-01-02T00:00:00%2B0800\
&offset=1&limit=20&type=SELECT" \
  -H "X-Auth-Token: ${HW_TOKEN}" | python3 -m json.tool
```

参数：
- `start_date` / `end_date`：格式 `yyyy-MM-ddTHH:mm:ss+0800`（URL encode `+` 为 `%2B`）
- `type`：`SELECT` | `INSERT` | `UPDATE` | `DELETE` | `CREATE`（不填则返回全部）
- 时间范围最长 30 天

关键响应字段：`time`（执行耗时）、`rows_examined`（扫描行数）、`query_sample`（SQL 示例）

### 查询错误日志

**触发词：** "查错误日志"、"数据库有没有报错"

```bash
curl -s "https://${RDS_ENDPOINT}/v3/${HW_PROJECT_ID}/instances/{instance_id}/errorlog?\
start_date=2024-01-01T00:00:00%2B0800\
&end_date=2024-01-02T00:00:00%2B0800\
&offset=1&limit=20&level=WARNING" \
  -H "X-Auth-Token: ${HW_TOKEN}" | python3 -m json.tool
```

`level`：`ALL` | `INFO` | `WARNING` | `ERROR` | `FATAL`

---

## 安全配置

### 开启/关闭 SSL

```bash
# 开启 SSL
curl -s -X PUT \
  https://${RDS_ENDPOINT}/v3/${HW_PROJECT_ID}/instances/{instance_id}/ssl \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: ${HW_TOKEN}" \
  -d '{"ssl_enable": true}'
```

### 修改实例访问端口

```bash
curl -s -X PUT \
  https://${RDS_ENDPOINT}/v3/${HW_PROJECT_ID}/instances/{instance_id}/port \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: ${HW_TOKEN}" \
  -d '{"port": 3307}'
```

---

## 完整操作流程示例

### 场景：从零创建生产级 MySQL 实例

```
用户：帮我在 cn-north-4 创建一个 MySQL 8.0 主备实例，4核8G，100G SSD，放在我的默认 VPC
Agent 执行步骤：
1. 调用「查询数据库规格」找到 4核8G MySQL 8.0 主备的 flavor_ref
2. 向用户确认 VPC/子网/安全组 ID
3. 调用「创建实例」API
4. 轮询 job_id 直到完成
5. 调用「查询实例详情」获取连接地址，返回给用户
```

### 场景：数据恢复

```
用户：把生产库恢复到昨天晚上10点
Agent 执行步骤：
1. 调用「查询实例列表」确认实例 ID
2. 调用「查询可恢复时间段」确认昨晚 22:00 在可恢复范围内
3. 将"昨晚 10 点"转换为毫秒时间戳
4. 向用户确认：将恢复到新实例（不覆盖现有实例）
5. 调用「恢复到新实例」API
6. 轮询 job_id，完成后返回新实例连接信息
```

---

## Common Pitfalls

1. **Token 过期**：IAM Token 有效期 24 小时。报 401 错误时需重新获取 Token，不要直接重试。

2. **可用区格式**：主备实例 `availability_zone` 必须填两个 AZ（如 `cn-north-4a,cn-north-4b`），单机实例只填一个。填错会报参数错误。

3. **恢复时间戳越界**：PITR 的 `restore_time` 必须在「查询可恢复时间段」返回的范围内，否则报 400。先查再填，不要猜测。

4. **磁盘只能扩不能缩**：`enlarge_volume` 的 `size` 必须大于当前值，且为 10 的整数倍。

5. **删除操作不可逆**：删除实例会同时删除所有自动备份。务必先让用户确认，并建议先手动备份。

6. **参数修改需重启**：修改某些参数（如 `innodb_buffer_pool_size`）后需重启实例，响应中会有 `restart_required: true` 提示，需告知用户。

7. **跨区域 Endpoint**：每个区域有独立的 Endpoint，不能混用。`cn-north-4` 实例必须用 `rds.cn-north-4.myhuaweicloud.com`。

8. **密码特殊字符**：管理员/账号密码必须包含 3 类字符，且不能含有账号名，否则报密码强度不足错误。

9. **分页 limit 上限**：查询接口 `limit` 最大 100，超过会报参数校验失败。超 100 条数据需循环分页。

10. **只读实例创建**：创建只读实例时需携带 `replica_of_id` 字段指定主实例 ID，且 `ha.mode` 设为 `Replica`。

## Verification Checklist

- [ ] 确认已获取有效的 IAM Token（未过期）
- [ ] 确认 HW_REGION、HW_PROJECT_ID 与目标实例一致
- [ ] 破坏性操作（删除、覆盖恢复、重启）已向用户二次确认
- [ ] 创建/变更操作后通过 job_id 轮询确认任务完成
- [ ] 返回给用户的连接信息（IP、端口）来自查询接口的实际响应，非假设值
- [ ] PITR 操作前已查询可恢复时间段，确认目标时间点在范围内
