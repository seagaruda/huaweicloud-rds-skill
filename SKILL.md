
# 华为云业务上云Skill

## Overview

帮助用户将单体或分布式应用部署到华为云，覆盖完整的基础设施链路：

- **安全组**：创建、添加规则、常用端口速查
- **ECS**：创建云服务器、批量启停、查询状态
- **EIP**：申请弹性公网 IP、绑定到 ECS 或 ELB、加入共享带宽包
- **共享带宽**：创建共享流量包、添加/移除 EIP
- **OBS**：创建 Bucket、上传私有镜像或制品（AK/SK 签名）
- **ECS ↔ RDS 连接**：安全组打通 + 连接字符串配置
- **ELB**：创建负载均衡器、监听器、后端服务器组、添加 ECS 节点、健康检查
- **RDS**：创建实例、备份恢复、账号管理、慢日志、参数配置

详细 API 参考见 `references/api-endpoints.md`。

## When to Use

- 用户说"帮我部署应用到华为云"、"建个 ECS"、"配一下安全组"
- 用户说"申请一个公网 IP"、"给 ECS 配负载均衡"、"多节点部署"
- 用户说"上传镜像到 OBS"、"ECS 怎么连 RDS"
- 不适用于：GaussDB、DCS、DDS 等其他数据库产品；K8s/CCE 容器部署

## 前置变量

| 变量 | 说明 | Endpoint 推导 |
|------|------|---------------|
| `TOKEN` | IAM Token（24h 有效） | `POST iam.{REGION}.myhuaweicloud.com/v3/auth/tokens` |
| `REGION` | 区域 ID，如 `cn-north-4` | - |
| `PROJECT_ID` | 项目 ID | `GET iam.{REGION}.myhuaweicloud.com/v3/auth/projects` |

各服务 Endpoint：`{service}.{REGION}.myhuaweicloud.com`，service = ecs / vpc / elb / rds / obs

## 典型架构

```
单体：  Internet → EIP → ECS → RDS

分布式：Internet → EIP → ELB → ECS-1 ┐
                               ECS-2 ├→ RDS（主备）
                               ECS-N ┘
```

**完整部署步骤（分布式，按序）：**
1. 创建安全组 + 添加端口规则
2. 创建 N 台 ECS，关联安全组
3. 申请 EIP（或加入共享带宽包）
4. 上传私有镜像到 OBS（可选）
5. 创建 RDS 实例，安全组放通 ECS→RDS 端口
6. 创建 ELB → 监听器 → 后端服务器组 → 添加 ECS → 健康检查
7. 将 EIP 绑定到 ELB

---

## 一、安全组

```bash
# 创建安全组
curl -s -X POST https://vpc.{REGION}.myhuaweicloud.com/v2.0/security-groups \
  -H "Content-Type: application/json" -H "X-Auth-Token: {TOKEN}" \
  -d '{"security_group": {"name": "sg-web-app", "description": "Web 应用安全组"}}'

# 添加入方向规则（开放 HTTP 80）
curl -s -X POST https://vpc.{REGION}.myhuaweicloud.com/v2.0/security-group-rules \
  -H "Content-Type: application/json" -H "X-Auth-Token: {TOKEN}" \
  -d '{
    "security_group_rule": {
      "security_group_id": "{SG_ID}", "direction": "ingress",
      "protocol": "tcp", "port_range_min": 80, "port_range_max": 80,
      "remote_ip_prefix": "0.0.0.0/0"
    }
  }'
```

**常用端口速查：** HTTP:80, HTTPS:443, SSH:22, RDP:3389, MySQL:3306, PG:5432, 自定义:8080/8443

ECS→RDS 打通：`direction=ingress`，`remote_group_id={ECS_SG_ID}`（用安全组互通，比 CIDR 更安全）

---

## 二、ECS 云服务器

```bash
# 创建 ECS
curl -s -X POST https://ecs.{REGION}.myhuaweicloud.com/v1/{PROJECT_ID}/cloudservers \
  -H "Content-Type: application/json" -H "X-Auth-Token: {TOKEN}" \
  -d '{
    "server": {
      "name": "app-server-01", "imageRef": "{IMAGE_ID}", "flavorRef": "c3.xlarge.2",
      "availability_zone": "{AZ}", "vpcid": "{VPC_ID}",
      "nics": [{"subnet_id": "{SUBNET_ID}"}],
      "security_groups": [{"id": "{SG_ID}"}],
      "root_volume": {"volumetype": "SSD", "size": 50},
      "adminPass": "{PASSWORD}"
    }
  }'
# 返回 job_id，轮询 GET /v1/{PROJECT_ID}/jobs/{JOB_ID} 直到 status=SUCCESS

# 批量启动 / 关机 / 重启
curl -s -X POST https://ecs.{REGION}.myhuaweicloud.com/v1/{PROJECT_ID}/cloudservers/action \
  -H "Content-Type: application/json" -H "X-Auth-Token: {TOKEN}" \
  -d '{"os-start": {"servers": [{"id": "{SERVER_ID}"}]}}'
# os-stop: {"type":"SOFT","servers":[...]}, reboot: {"type":"SOFT","servers":[...]}

# 查询列表
curl -s "https://ecs.{REGION}.myhuaweicloud.com/v1/{PROJECT_ID}/cloudservers/detail?limit=20&offset=1" \
  -H "X-Auth-Token: {TOKEN}"
```

状态：`ACTIVE`运行中 / `SHUTOFF`关机 / `BUILD`创建中 / `ERROR`异常

规格命名：`s6.large.2`=2C4G通用，`c3.xlarge.2`=4C8G计算型，`m3.xlarge.8`=4C32G内存型

---

## 三、弹性公网 IP（EIP）

```bash
# 申请独享 EIP（按带宽计费，5Mbps）
curl -s -X POST https://vpc.{REGION}.myhuaweicloud.com/v1/{PROJECT_ID}/publicips \
  -H "Content-Type: application/json" -H "X-Auth-Token: {TOKEN}" \
  -d '{
    "publicip": {"type": "5_bgp"},
    "bandwidth": {"name": "bw-app", "size": 5, "share_type": "PER", "charge_mode": "bandwidth"}
  }'
# 返回 publicip.id 和 public_ip_address

# 绑定 EIP 到 ECS 网卡端口
curl -s -X POST \
  https://vpc.{REGION}.myhuaweicloud.com/v1/{PROJECT_ID}/publicips/{EIP_ID}/action \
  -H "Content-Type: application/json" -H "X-Auth-Token: {TOKEN}" \
  -d '{"publicip": {"port_id": "{PORT_ID}"}}'
# PORT_ID = ECS 网卡的 port，从 ECS 详情的 addresses 中获取

# 解绑 EIP
curl -s -X POST \
  https://vpc.{REGION}.myhuaweicloud.com/v1/{PROJECT_ID}/publicips/{EIP_ID}/action \
  -H "Content-Type: application/json" -H "X-Auth-Token: {TOKEN}" \
  -d '{"publicip": {"port_id": null}}'
```

---

## 四、共享带宽包

```bash
# 创建共享带宽
curl -s -X POST https://vpc.{REGION}.myhuaweicloud.com/v2.0/{PROJECT_ID}/bandwidths \
  -H "Content-Type: application/json" -H "X-Auth-Token: {TOKEN}" \
  -d '{"bandwidth": {"name": "shared-bw-prod", "size": 50}}'
# 返回 bandwidth.id

# 将 EIP 加入共享带宽
curl -s -X POST \
  https://vpc.{REGION}.myhuaweicloud.com/v2.0/{PROJECT_ID}/bandwidths/{BW_ID}/insert \
  -H "Content-Type: application/json" -H "X-Auth-Token: {TOKEN}" \
  -d '{"bandwidth": {"publicip_info": [{"publicip_id": "{EIP_ID}", "publicip_type": "5_bgp"}]}}'

# 从共享带宽移除 EIP（移除后需指定该 EIP 新的独享带宽大小）
curl -s -X POST \
  https://vpc.{REGION}.myhuaweicloud.com/v2.0/{PROJECT_ID}/bandwidths/{BW_ID}/remove \
  -H "Content-Type: application/json" -H "X-Auth-Token: {TOKEN}" \
  -d '{"bandwidth": {"publicip_info": [{"publicip_id": "{EIP_ID}", "publicip_type": "5_bgp"}], "size": 5, "charge_mode": "bandwidth"}}'
```

---

## 五、OBS 上传私有镜像/制品

OBS 使用 AK/SK 签名，不用 IAM Token。需要先在控制台创建访问密钥（AK/SK）。

```bash
# 创建 Bucket（仅第一次）
curl -s -X PUT https://{BUCKET_NAME}.obs.{REGION}.myhuaweicloud.com \
  -H "Authorization: OBS {AK}:{SIGNATURE}" \
  -H "Date: {GMT_DATE}" \
  -H "x-obs-acl: private"

# 上传文件（PUT Object）
curl -s -X PUT \
  https://{BUCKET_NAME}.obs.{REGION}.myhuaweicloud.com/{OBJECT_KEY} \
  -H "Authorization: OBS {AK}:{SIGNATURE}" \
  -H "Date: {GMT_DATE}" \
  -H "Content-Type: application/octet-stream" \
  --data-binary @/path/to/your/image.tar.gz
```

**实际推荐用法**：用华为云官方 `obsutil` CLI 工具，避免手动计算签名：
```bash
# 安装 obsutil
wget https://obs-community.obs.cn-north-1.myhuaweicloud.com/obsutil/current/linux_amd64/obsutil_linux_amd64.tar.gz
tar -xzf obsutil_linux_amd64.tar.gz && chmod +x obsutil

# 配置 AK/SK
./obsutil config -i={AK} -k={SK} -e=obs.{REGION}.myhuaweicloud.com

# 上传
./obsutil cp /local/image.tar.gz obs://{BUCKET_NAME}/images/image.tar.gz
```

---

## 六、ECS ↔ RDS 连接

1. **安全组打通**：在 RDS 实例的安全组中，添加入方向规则，`remote_group_id` 设为 ECS 所在安全组 ID，端口为数据库端口（MySQL:3306，PG:5432）。
2. **获取 RDS 内网 IP**：从 RDS 实例详情 `private_ips` 字段获取，仅同 VPC 内可达。
3. **连接字符串**：
   - MySQL：`mysql -h {RDS_PRIVATE_IP} -P 3306 -u {DB_USER} -p {DB_NAME}`
   - PG：`psql -h {RDS_PRIVATE_IP} -p 5432 -U {DB_USER} -d {DB_NAME}`
   - 应用配置：`jdbc:mysql://{RDS_PRIVATE_IP}:3306/{DB_NAME}?useSSL=true`

**注意**：ECS 和 RDS 必须在同一 VPC 内；跨 VPC 需配置对等连接。

---

## 七、ELB 负载均衡（多节点）

完整步骤：创建 LB → 创建监听器 → 创建后端服务器组 → 添加 ECS 成员 → 配置健康检查 → 绑 EIP

```bash
# 步骤1：创建负载均衡器
curl -s -X POST https://elb.{REGION}.myhuaweicloud.com/v3/{PROJECT_ID}/elb/loadbalancers \
  -H "Content-Type: application/json" -H "X-Auth-Token: {TOKEN}" \
  -d '{
    "loadbalancer": {
      "name": "lb-prod", "vpc_id": "{VPC_ID}",
      "vip_subnet_cidr_id": "{SUBNET_ID}",
      "availability_zone_list": ["{AZ}"],
      "guaranteed": false
    }
  }'
# guaranteed=false 为共享型（免费额度），true 为独享型（需指定 l4_flavor_id 或 l7_flavor_id）
# 返回 loadbalancer.id

# 步骤2：创建监听器
curl -s -X POST https://elb.{REGION}.myhuaweicloud.com/v3/{PROJECT_ID}/elb/listeners \
  -H "Content-Type: application/json" -H "X-Auth-Token: {TOKEN}" \
  -d '{
    "listener": {
      "name": "listener-http-80", "loadbalancer_id": "{LB_ID}",
      "protocol": "HTTP", "protocol_port": 80
    }
  }'
# protocol 可选: TCP / UDP / HTTP / HTTPS / TERMINATED_HTTPS
# 返回 listener.id

# 步骤3：创建后端服务器组
curl -s -X POST https://elb.{REGION}.myhuaweicloud.com/v3/{PROJECT_ID}/elb/pools \
  -H "Content-Type: application/json" -H "X-Auth-Token: {TOKEN}" \
  -d '{
    "pool": {
      "name": "pool-app", "protocol": "HTTP",
      "lb_algorithm": "ROUND_ROBIN", "listener_id": "{LISTENER_ID}"
    }
  }'
# lb_algorithm: ROUND_ROBIN / LEAST_CONNECTIONS / SOURCE_IP
# 返回 pool.id

# 步骤4：添加 ECS 节点为后端服务器
curl -s -X POST https://elb.{REGION}.myhuaweicloud.com/v3/{PROJECT_ID}/elb/pools/{POOL_ID}/members \
  -H "Content-Type: application/json" -H "X-Auth-Token: {TOKEN}" \
  -d '{
    "member": {
      "name": "member-ecs-01", "address": "{ECS_PRIVATE_IP}",
      "protocol_port": 8080, "subnet_cidr_id": "{SUBNET_ID}",
      "weight": 1
    }
  }'
# address = ECS 内网 IP，protocol_port = 应用实际监听端口（非 LB 端口）

# 步骤5：配置健康检查
curl -s -X POST https://elb.{REGION}.myhuaweicloud.com/v3/{PROJECT_ID}/elb/healthmonitors \
  -H "Content-Type: application/json" -H "X-Auth-Token: {TOKEN}" \
  -d '{
    "healthmonitor": {
      "pool_id": "{POOL_ID}", "type": "HTTP",
      "monitor_port": 8080, "url_path": "/health",
      "delay": 5, "timeout": 3, "max_retries": 3
    }
  }'
# type: TCP / HTTP / HTTPS; url_path 仅 HTTP/HTTPS 有效

# 步骤6：将 EIP 绑定到 ELB（通过更新 LB 的 vip_address 关联的 port 来绑 EIP）
# 先查 LB 的 vip_port_id，再绑 EIP
curl -s "https://elb.{REGION}.myhuaweicloud.com/v3/{PROJECT_ID}/elb/loadbalancers/{LB_ID}" \
  -H "X-Auth-Token: {TOKEN}"
# 取 loadbalancer.vip_port_id，然后用 EIP 绑定接口绑到此 port_id
```

---

## 八、RDS 数据库

```bash
# 创建 MySQL 8.0 主备实例
curl -s -X POST https://rds.{REGION}.myhuaweicloud.com/v3/{PROJECT_ID}/instances \
  -H "Content-Type: application/json" -H "X-Auth-Token: {TOKEN}" \
  -d '{
    "name": "rds-prod", "datastore": {"type": "MySQL", "version": "8.0"},
    "ha": {"mode": "Ha", "replication_mode": "semisync"},
    "password": "{ADMIN_PASSWORD}", "flavor_ref": "rds.mysql.c2.large.ha",
    "volume": {"type": "ULTRAHIGH", "size": 100},
    "region": "{REGION}", "availability_zone": "{AZ1},{AZ2}",
    "vpc_id": "{VPC_ID}", "subnet_id": "{SUBNET_ID}",
    "security_group": {"id": "{RDS_SG_ID}"}
  }'

# 其他常用操作（均为 /v3/{PROJECT_ID}/...）：
# 查询实例列表：GET /instances
# 创建手动备份：POST /backups
# PITR 恢复：POST /instances（source.type=timestamp）
# 创建账号：POST /instances/{id}/db_user
# 查询慢日志：GET /instances/{id}/slowlog
# 修改参数：PUT /instances/{id}/configurations
```

详见 `references/rds-api.md`。

---

## 九、完整端到端场景：3节点 Web 应用 + MySQL 从零部署

**目标架构**：
```
Internet → EIP → ELB(HTTP:80) → app-01(8080)
                              → app-02(8080)  → RDS MySQL 8.0（主备）
                              → app-03(8080)
```

所有操作均依赖 TOKEN、REGION、PROJECT_ID，假设 VPC 和子网已存在。

### 阶段一：安全组

```bash
# 1a. 创建 Web 层安全组（给 ECS 用）
SG_WEB=$(curl -s -X POST https://vpc.${REGION}.myhuaweicloud.com/v2.0/security-groups \
  -H "Content-Type: application/json" -H "X-Auth-Token: ${TOKEN}" \
  -d '{"security_group":{"name":"sg-web","description":"Web ECS"}}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['security_group']['id'])")

# 1b. 创建 DB 层安全组（给 RDS 用）
SG_DB=$(curl -s -X POST https://vpc.${REGION}.myhuaweicloud.com/v2.0/security-groups \
  -H "Content-Type: application/json" -H "X-Auth-Token: ${TOKEN}" \
  -d '{"security_group":{"name":"sg-db","description":"RDS MySQL"}}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['security_group']['id'])")

# 1c. sg-web 开放：SSH(22)、应用端口(8080)、ELB健康检查(8080 from 100.125.0.0/16)
for PORT in 22 8080; do
  curl -s -X POST https://vpc.${REGION}.myhuaweicloud.com/v2.0/security-group-rules \
    -H "Content-Type: application/json" -H "X-Auth-Token: ${TOKEN}" \
    -d "{\"security_group_rule\":{\"security_group_id\":\"${SG_WEB}\",\"direction\":\"ingress\",
         \"protocol\":\"tcp\",\"port_range_min\":${PORT},\"port_range_max\":${PORT},
         \"remote_ip_prefix\":\"0.0.0.0/0\"}}"
done

# 1d. sg-db 开放：仅允许 sg-web 访问 MySQL 3306（安全组互通）
curl -s -X POST https://vpc.${REGION}.myhuaweicloud.com/v2.0/security-group-rules \
  -H "Content-Type: application/json" -H "X-Auth-Token: ${TOKEN}" \
  -d "{\"security_group_rule\":{\"security_group_id\":\"${SG_DB}\",\"direction\":\"ingress\",
       \"protocol\":\"tcp\",\"port_range_min\":3306,\"port_range_max\":3306,
       \"remote_group_id\":\"${SG_WEB}\"}}"
```

### 阶段二：创建 3 台 ECS

```bash
# 循环创建 app-01 / app-02 / app-03
for i in 01 02 03; do
  JOB=$(curl -s -X POST https://ecs.${REGION}.myhuaweicloud.com/v1/${PROJECT_ID}/cloudservers \
    -H "Content-Type: application/json" -H "X-Auth-Token: ${TOKEN}" \
    -d "{
      \"server\": {
        \"name\": \"app-${i}\", \"imageRef\": \"${IMAGE_ID}\",
        \"flavorRef\": \"s6.large.2\",
        \"availability_zone\": \"${AZ}\", \"vpcid\": \"${VPC_ID}\",
        \"nics\": [{\"subnet_id\": \"${SUBNET_ID}\"}],
        \"security_groups\": [{\"id\": \"${SG_WEB}\"}],
        \"root_volume\": {\"volumetype\": \"SSD\", \"size\": 50},
        \"adminPass\": \"${PASSWORD}\"
      }
    }" | python3 -c "import sys,json; print(json.load(sys.stdin)['job_id'])")
  echo "app-${i} job_id=${JOB}"
  # 轮询直到 SUCCESS
  while true; do
    STATUS=$(curl -s "https://ecs.${REGION}.myhuaweicloud.com/v1/${PROJECT_ID}/jobs/${JOB}" \
      -H "X-Auth-Token: ${TOKEN}" | python3 -c "import sys,json; print(json.load(sys.stdin)['status'])")
    [ "$STATUS" = "SUCCESS" ] && break
    echo "  waiting... $STATUS"; sleep 10
  done
done
```

### 阶段三：申请 EIP + 共享带宽

```bash
# 3a. 创建共享带宽（50Mbps，多 EIP 共享）
BW_ID=$(curl -s -X POST https://vpc.${REGION}.myhuaweicloud.com/v2.0/${PROJECT_ID}/bandwidths \
  -H "Content-Type: application/json" -H "X-Auth-Token: ${TOKEN}" \
  -d '{"bandwidth":{"name":"shared-bw-prod","size":50}}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['bandwidth']['id'])")

# 3b. 申请 EIP（后续绑到 ELB，先用独享）
EIP_ID=$(curl -s -X POST https://vpc.${REGION}.myhuaweicloud.com/v1/${PROJECT_ID}/publicips \
  -H "Content-Type: application/json" -H "X-Auth-Token: ${TOKEN}" \
  -d '{"publicip":{"type":"5_bgp"},"bandwidth":{"name":"bw-elb","size":50,"share_type":"PER","charge_mode":"bandwidth"}}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['publicip']['id'])")
```

### 阶段四：创建 RDS MySQL 主备实例

```bash
RDS_JOB=$(curl -s -X POST https://rds.${REGION}.myhuaweicloud.com/v3/${PROJECT_ID}/instances \
  -H "Content-Type: application/json" -H "X-Auth-Token: ${TOKEN}" \
  -d "{
    \"name\": \"rds-prod\",
    \"datastore\": {\"type\": \"MySQL\", \"version\": \"8.0\"},
    \"ha\": {\"mode\": \"Ha\", \"replication_mode\": \"semisync\"},
    \"password\": \"${DB_ADMIN_PASS}\",
    \"flavor_ref\": \"rds.mysql.c2.large.ha\",
    \"volume\": {\"type\": \"ULTRAHIGH\", \"size\": 100},
    \"region\": \"${REGION}\",
    \"availability_zone\": \"${AZ1},${AZ2}\",
    \"vpc_id\": \"${VPC_ID}\",
    \"subnet_id\": \"${SUBNET_ID}\",
    \"security_group\": {\"id\": \"${SG_DB}\"}
  }" | python3 -c "import sys,json; d=json.load(sys.stdin); print(d['instance']['id'])")
echo "RDS instance_id=${RDS_JOB}"

# 等待 RDS 状态变为 available（轮询约5-10分钟）
while true; do
  S=$(curl -s "https://rds.${REGION}.myhuaweicloud.com/v3/${PROJECT_ID}/instances/${RDS_JOB}" \
    -H "X-Auth-Token: ${TOKEN}" | python3 -c "import sys,json; print(json.load(sys.stdin)['instance']['status'])")
  [ "$S" = "available" ] && break
  echo "  RDS status=$S, waiting..."; sleep 30
done

# 创建业务账号
curl -s -X POST https://rds.${REGION}.myhuaweicloud.com/v3/${PROJECT_ID}/instances/${RDS_JOB}/db_user \
  -H "Content-Type: application/json" -H "X-Auth-Token: ${TOKEN}" \
  -d "{\"name\":\"appuser\",\"password\":\"${DB_USER_PASS}\",
       \"databases\":[{\"name\":\"appdb\",\"readonly\":false}]}"
```

### 阶段五：ELB 配置（全流程）

```bash
# 5a. 创建负载均衡器
LB_ID=$(curl -s -X POST https://elb.${REGION}.myhuaweicloud.com/v3/${PROJECT_ID}/elb/loadbalancers \
  -H "Content-Type: application/json" -H "X-Auth-Token: ${TOKEN}" \
  -d "{\"loadbalancer\":{\"name\":\"lb-prod\",\"vpc_id\":\"${VPC_ID}\",
       \"vip_subnet_cidr_id\":\"${SUBNET_ID}\",
       \"availability_zone_list\":[\"${AZ1}\"],\"guaranteed\":false}}" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['loadbalancer']['id'])")

# 5b. 创建 HTTP 监听器（80）
LISTENER_ID=$(curl -s -X POST https://elb.${REGION}.myhuaweicloud.com/v3/${PROJECT_ID}/elb/listeners \
  -H "Content-Type: application/json" -H "X-Auth-Token: ${TOKEN}" \
  -d "{\"listener\":{\"name\":\"listener-http-80\",\"loadbalancer_id\":\"${LB_ID}\",
       \"protocol\":\"HTTP\",\"protocol_port\":80}}" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['listener']['id'])")

# 5c. 创建后端服务器组（轮询）
POOL_ID=$(curl -s -X POST https://elb.${REGION}.myhuaweicloud.com/v3/${PROJECT_ID}/elb/pools \
  -H "Content-Type: application/json" -H "X-Auth-Token: ${TOKEN}" \
  -d "{\"pool\":{\"name\":\"pool-app\",\"protocol\":\"HTTP\",
       \"lb_algorithm\":\"ROUND_ROBIN\",\"listener_id\":\"${LISTENER_ID}\"}}" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['pool']['id'])")

# 5d. 将 3 台 ECS 加入后端（替换 ECS_IP_01/02/03 为实际内网 IP）
for ECS_IP in ${ECS_IP_01} ${ECS_IP_02} ${ECS_IP_03}; do
  curl -s -X POST https://elb.${REGION}.myhuaweicloud.com/v3/${PROJECT_ID}/elb/pools/${POOL_ID}/members \
    -H "Content-Type: application/json" -H "X-Auth-Token: ${TOKEN}" \
    -d "{\"member\":{\"address\":\"${ECS_IP}\",\"protocol_port\":8080,
         \"subnet_cidr_id\":\"${SUBNET_ID}\",\"weight\":1}}"
done

# 5e. 配置健康检查（HTTP GET /health）
curl -s -X POST https://elb.${REGION}.myhuaweicloud.com/v3/${PROJECT_ID}/elb/healthmonitors \
  -H "Content-Type: application/json" -H "X-Auth-Token: ${TOKEN}" \
  -d "{\"healthmonitor\":{\"pool_id\":\"${POOL_ID}\",\"type\":\"HTTP\",
       \"monitor_port\":8080,\"url_path\":\"/health\",
       \"delay\":5,\"timeout\":3,\"max_retries\":3}}"

# 5f. 获取 ELB VIP 的 port_id，然后将 EIP 绑上去
VIP_PORT=$(curl -s "https://elb.${REGION}.myhuaweicloud.com/v3/${PROJECT_ID}/elb/loadbalancers/${LB_ID}" \
  -H "X-Auth-Token: ${TOKEN}" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['loadbalancer']['vip_port_id'])")

curl -s -X PUT https://vpc.${REGION}.myhuaweicloud.com/v1/${PROJECT_ID}/publicips/${EIP_ID} \
  -H "Content-Type: application/json" -H "X-Auth-Token: ${TOKEN}" \
  -d "{\"publicip\":{\"port_id\":\"${VIP_PORT}\"}}"
```

### 阶段六：部署应用 + 配置数据库连接

```bash
# 在每台 ECS 上执行（以 app-01 为例，通过 SSH）
ssh root@${ECS_IP_01} << 'EOF'
# 从 OBS 下载应用包
./obsutil cp obs://my-bucket/releases/app-v1.0.tar.gz /opt/app/ -i=${AK} -k=${SK}

# 解压部署
tar -xzf /opt/app/app-v1.0.tar.gz -C /opt/app/
cat > /opt/app/config.env << CONF
DB_HOST=${RDS_PRIVATE_IP}
DB_PORT=3306
DB_NAME=appdb
DB_USER=appuser
DB_PASS=${DB_USER_PASS}
APP_PORT=8080
CONF

# 启动（示例：Java 应用）
nohup java -jar /opt/app/app.jar --spring.config.location=/opt/app/config.env > /var/log/app.log 2>&1 &
echo "App started, PID=$!"
EOF
```

### 验证清单

```bash
# 1. 检查 ELB 后端节点健康状态（期望 ONLINE）
curl -s "https://elb.${REGION}.myhuaweicloud.com/v3/${PROJECT_ID}/elb/pools/${POOL_ID}/members" \
  -H "X-Auth-Token: ${TOKEN}" | python3 -c \
  "import sys,json; [print(m['address'], m['operating_status']) for m in json.load(sys.stdin)['members']]"

# 2. 通过 EIP 访问应用
curl -I http://${EIP_ADDRESS}/health
# 期望: HTTP/1.1 200 OK

# 3. 检查 RDS 可连接性（在 ECS 上执行）
mysql -h ${RDS_PRIVATE_IP} -P 3306 -u appuser -p${DB_USER_PASS} appdb -e "SELECT 1;"
```

---

## Common Pitfalls

1. **上下文过载导致工具调用参数截断**：本 Skill 的文档收集涉及多个并行 web 请求，消耗大量上下文。收集完文档后，若继续在同一会话中写大文件（SKILL.md、README 等），工具调用的 `content` 参数会被静默截断，导致写入失败。**解决方案**：用 `delegate_task` 将"写文件 + git push"委托给子 Agent，隔离上下文。子 Agent 超时（600s）时，改为手动分段写入。

2. **EIP 绑定需要 port_id 而非 server_id**：绑定 EIP 时需提供 ECS 网卡的 `port_id`，不是 server ID。从 ECS 详情的 `addresses` 字段中取 `OS-EXT-IPS:port_id`。

3. **ELB 后端服务器地址是 ECS 内网 IP，端口是应用端口**：`member.address` 填 ECS 内网 IP，`protocol_port` 填应用实际监听端口（如 8080），不是 LB 监听端口（80）。

4. **共享带宽移除 EIP 时需指定新独享带宽大小**：移除时 `size` 和 `charge_mode` 是必填项，指定该 EIP 离开共享带宽后的独享配置。

5. **OBS 不用 IAM Token**：OBS 使用 AK/SK 签名（HMAC-SHA1），与其他服务的 Token 认证不同。强烈推荐用 `obsutil` CLI 而非手写签名。

6. **ECS 创建返回 job_id 而非直接返回 server_id**：必须轮询 `GET /v1/{PROJECT_ID}/jobs/{JOB_ID}` 直到 `status=SUCCESS`，再从 `entities.server_id` 取 ID。

7. **RDS 和 ECS 必须在同一 VPC**：跨 VPC 访问 RDS 需先建 VPC 对等连接，不是直接连。

8. **ELB 绑 EIP 通过 vip_port_id**：ELB 不直接提供"绑 EIP"接口，需先查 LB 详情取 `vip_port_id`，再用 EIP 绑定接口把该 port 绑上去。

## Verification Checklist

- [ ] TOKEN 已获取且未过期（24h 有效）
- [ ] ECS、RDS 在同一 VPC 和同一区域
- [ ] RDS 安全组的入方向包含 ECS 安全组 ID（不是 CIDR）
- [ ] ELB member 的 `address` = ECS 内网 IP，`protocol_port` = 应用端口
- [ ] ECS 创建后通过 job_id 轮询确认 SUCCESS 再取 server_id
- [ ] OBS 上传使用 AK/SK，不是 IAM Token
- [ ] EIP 绑定使用 port_id，不是 server_id
