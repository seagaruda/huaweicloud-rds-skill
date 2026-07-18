# ELB API Quick Reference

Base: `https://elb.{REGION}.myhuaweicloud.com/v3/{PROJECT_ID}`
Auth: `X-Auth-Token: {TOKEN}`

> **API version note**: This reference uses **v3** (the current recommended API).
> All resource paths are prefixed with `/elb/` under the v3 base URL.
> (Legacy v2 paths like `/v2/{PROJECT_ID}/loadbalancers` are deprecated.)

## Load Balancer Management

| Operation | Method | Path |
|-----------|--------|------|
| Create LB | POST | `/elb/loadbalancers` |
| List LBs | GET | `/elb/loadbalancers?limit=1000` |
| Get LB details | GET | `/elb/loadbalancers/{loadbalancer_id}` |
| Update LB | PUT | `/elb/loadbalancers/{loadbalancer_id}` |
| Delete LB | DELETE | `/elb/loadbalancers/{loadbalancer_id}` |

Create request body:
```json
{
  "loadbalancer": {
    "name": "elb-web-01",
    "vip_subnet_cidr": "{subnet_id}",
    "vip_address": "192.168.1.100",
    "listeners": [
      {
        "name": "listener-http",
        "protocol": "HTTP",
        "protocol_port": 80,
        "server_groups": [
          {
            "name": "sg-web",
            "algorithm": "ROUND_ROBIN"
          }
        ]
      }
    ],
    "flavor": "elb.t1.micro",
    "availability_zone": ["cn-north-4a", "cn-north-4b"]
  }
}
```

## Listener Management

| Operation | Method | Path |
|-----------|--------|------|
| Create listener | POST | `/elb/listeners` |
| List listeners | GET | `/elb/listeners?loadbalancer_id={lb_id}` |
| Get listener details | GET | `/elb/listeners/{listener_id}` |
| Update listener | PUT | `/elb/listeners/{listener_id}` |
| Delete listener | DELETE | `/elb/listeners/{listener_id}` |

### Listener Protocols

| Protocol | Port Range | Use Case |
|----------|-----------|----------|
| `HTTP` | 1–65535 | Web 应用（7 层） |
| `HTTPS` | 1–65535 | 加密 Web（7 层） |
| `TCP` | 1–65535 | 通用 TCP（4 层） |
| `UDP` | 1–65535 | DNS、视频流（4 层） |

## Pool / Server Group Management

| Operation | Method | Path |
|-----------|--------|------|
| Create pool | POST | `/elb/pools` |
| List pools | GET | `/elb/pools?listener_id={listener_id}` |
| Get pool details | GET | `/elb/pools/{pool_id}` |
| Update pool | PUT | `/elb/pools/{pool_id}` |
| Delete pool | DELETE | `/elb/pools/{pool_id}` |

### Load Balancing Algorithms

| Algorithm | Description |
|-----------|-------------|
| `ROUND_ROBIN` | 轮询（默认） |
| `LEAST_CONNECTIONS` | 最少连接数 |
| `SOURCE_IP` | 源 IP 哈希（会话保持） |

## Member Management

| Operation | Method | Path |
|-----------|--------|------|
| Add member | POST | `/elb/pools/{pool_id}/members` |
| List members | GET | `/elb/pools/{pool_id}/members` |
| Get member details | GET | `/elb/pools/{pool_id}/members/{member_id}` |
| Update member | PUT | `/elb/pools/{pool_id}/members/{member_id}` |
| Delete member | DELETE | `/elb/pools/{pool_id}/members/{member_id}` |

Add member body:
```json
{
  "member": {
    "name": "member-ecs-01",
    "protocol_port": 8080,
    "address": "192.168.1.50",
    "weight": 1,
    "subnet_id": "{subnet_id}"
  }
}
```

## Health Check Management

| Operation | Method | Path |
|-----------|--------|------|
| Create health check | POST | `/elb/healthmonitors` |
| List health checks | GET | `/elb/healthmonitors?pool_id={pool_id}` |
| Update health check | PUT | `/elb/healthmonitors/{healthmonitor_id}` |
| Delete health check | DELETE | `/elb/healthmonitors/{healthmonitor_id}` |

Health check body:
```json
{
  "healthmonitor": {
    "delay": 5,
    "timeout": 3,
    "max_retries": 3,
    "type": "HTTP",
    "url_path": "/health",
    "expected_codes": "200",
    "http_method": "GET"
  }
}
```

### Health Check Types

| Type | Description |
|------|-------------|
| `TCP` | TCP 连接检测 |
| `HTTP` | HTTP GET 请求 |
| `HTTPS` | HTTPS 请求 |

## L7 Policies and Rules (HTTP/HTTPS)

| Operation | Method | Path |
|-----------|--------|------|
| Create L7 policy | POST | `/elb/l7policies` |
| Create L7 rule | POST | `/elb/l7policies/{policy_id}/rules` |
| List L7 policies | GET | `/elb/l7policies?listener_id={listener_id}` |

## Certificate Management (HTTPS)

| Operation | Method | Path |
|-----------|--------|------|
| Upload certificate | POST | `/elb/certificates` |
| List certificates | GET | `/elb/certificates` |
| Get certificate | GET | `/elb/certificates/{certificate_id}` |
| Delete certificate | DELETE | `/elb/certificates/{certificate_id}` |

## Common Scenarios

| Scenario | Configuration |
|----------|--------------|
| Web 负载均衡 | HTTPS listener → pool (ROUND_ROBIN) → HTTP health check `/health` |
| 数据库读写分离 | TCP listener → read pool + write pool → L7 policy 路由 |
| 灰度发布 | 创建 L7 policy，按 header/cookie 路由到新版本 pool |
| 跨可用区容灾 | LB flavor 配置多 AZ，member 分布在不同 AZ |

## ELB Flavors

| Flavor | Max Connections | Bandwidth (Mbps) | Use Case |
|--------|----------------|-------------------|----------|
| `elb.t1.micro` | 5000 | 500 | 小型业务 |
| `elb.t1.small` | 10000 | 1000 | 中型业务 |
| `elb.t1.medium` | 50000 | 5000 | 大型业务 |
| `elb.t1.large` | 100000 | 10000 | 超大型业务 |
