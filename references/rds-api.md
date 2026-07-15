# RDS v3 API 快速参考

Base: `https://rds.{REGION}.myhuaweicloud.com/v3/{PROJECT_ID}`
Auth: `X-Auth-Token: {TOKEN}`

## 实例管理

| 操作 | Method | Path |
|------|--------|------|
| 创建实例 | POST | `/instances` |
| 查询列表 | GET | `/instances?datastore_type=MySQL&type=Ha` |
| 查询详情 | GET | `/instances/{id}` |
| 重启 | POST | `/instances/{id}/action` body: `{"restart":{}}` |
| 变更规格 | POST | `/instances/{id}/action` body: `{"resize_flavor":{"spec_code":"..."}}` |
| 扩容磁盘 | POST | `/instances/{id}/action` body: `{"enlarge_volume":{"size":200}}` |
| 删除 | DELETE | `/instances/{id}` |

实例 `ha.mode`：`Ha`主备 / `Single`单机 / `Replica`只读
复制模式：MySQL=`semisync`，PG=`async`，SQLServer=`sync`

## 备份恢复

| 操作 | Method | Path |
|------|--------|------|
| 设置自动备份 | PUT | `/instances/{id}/backups/policy` |
| 创建手动备份 | POST | `/backups` body: `{"instance_id":"...","name":"..."}` |
| 查询备份列表 | GET | `/backups?instance_id={id}` |
| 查询可恢复时间段 | GET | `/instances/{id}/restore-time?date=YYYY-MM-DD` |
| PITR 恢复（新实例） | POST | `/instances` body 含 `source.type=timestamp` |
| 按备份恢复（新实例）| POST | `/instances` body 含 `source.type=backup` |
| 删除手动备份 | DELETE | `/backups/{backup_id}` |

PITR restore_time 单位为毫秒时间戳：`date -d "yesterday 22:00" +%s%3N`

## 账号管理（MySQL）

| 操作 | Method | Path |
|------|--------|------|
| 创建账号 | POST | `/instances/{id}/db_user` |
| 查询账号列表 | GET | `/instances/{id}/db_user/detail?page=1&limit=100` |
| 授权数据库 | POST | `/instances/{id}/db_user/privilege` body: `{"user_name":"...","databases":[{"name":"...","readonly":false}]}` |
| 重置密码 | POST | `/instances/{id}/db_user/resetpwd` |
| 删除账号 | DELETE | `/instances/{id}/db_user/{user_name}` |

密码规则：8-32位，含大小写字母/数字/特殊字符(`!@#$%^*-_=+?,`)至少3类，不含账号名。

## 日志与参数

| 操作 | Method | Path |
|------|--------|------|
| 查询慢日志 | GET | `/instances/{id}/slowlog?start_date=...&end_date=...&offset=1&limit=20` |
| 查询错误日志 | GET | `/instances/{id}/errorlog?level=WARNING&...` |
| 查询参数 | GET | `/instances/{id}/configurations` |
| 修改参数 | PUT | `/instances/{id}/configurations` body: `{"values":{"max_connections":"500"}}` |

慢日志 `start_date` 格式：`yyyy-MM-ddTHH:mm:ss+0800`（`+` URL encode 为 `%2B`），最长查30天。
修改参数响应含 `restart_required:true` 时需重启实例生效。

## 常用 MySQL 参数

| 参数 | 说明 | 推荐值 |
|------|------|--------|
| `max_connections` | 最大连接数 | 内存1G→150，4G→500，8G→800 |
| `long_query_time` | 慢查询阈值（秒） | 生产环境1-2 |
| `innodb_buffer_pool_size` | InnoDB缓冲池（字节） | 可用内存的70% |
| `character_set_server` | 字符集 | utf8mb4 |
| `time_zone` | 时区 | +08:00 |
