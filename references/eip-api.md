# EIP API Quick Reference

Base: `https://vpc.{REGION}.myhuaweicloud.com/v1/{PROJECT_ID}`
Auth: `X-Auth-Token: {TOKEN}`

## EIP Management

| Operation | Method | Path |
|-----------|--------|------|
| Create EIP | POST | `/publicips` |
| List EIPs | GET | `/publicips?limit=1000` |
| Get EIP details | GET | `/publicips/{publicip_id}` |
| Update EIP | PUT | `/publicips/{publicip_id}` |
| Delete EIP | DELETE | `/publicips/{publicip_id}` |

Create request body:
```json
{
  "publicip": {
    "type": "5_bgp",
    "ip_version": 4,
    "name": "eip-web-01",
    "description": "Web server EIP"
  },
  "bandwidth": {
    "name": "bw-web-01",
    "size": 5,
    "share_type": "PER",
    "charge_mode": "traffic"
  }
}
```

## EIP Types

| Type | Description | Use Case |
|------|-------------|----------|
| `5_bgp` | 动态 BGP 多线 | 推荐，国内+海外 |
| `5_sbgp` | 静态 BGP 多线 | 固定 IP，需审核 |
| `5_common` | 普通（单线） | 特定运营商 |

## Associate/Disassociate

EIP binding uses the `/action` endpoint with a JSON body specifying the
`port_id` to bind (or unbind). This is consistent with the EIP v1 API.

| Operation | Method | Path |
|-----------|--------|------|
| Associate (bind) | POST | `/publicips/{publicip_id}/action` |
| Disassociate (unbind) | POST | `/publicips/{publicip_id}/action` |

Associate (bind) body:
```json
{
  "publicip": {
    "port_id": "{port_id}"
  }
}
```

Disassociate (unbind) body:
```json
{
  "publicip": {
    "port_id": ""
  }
}
```

> `port_id` is the ECS VIF UUID, obtained from the ECS detail API
> (`public_ips[0].port_id`). To unbind, set `port_id` to an empty string.

## Bandwidth Management

| Operation | Method | Path |
|-----------|--------|------|
| List bandwidths | GET | `/vpc/bandwidths?limit=1000` |
| Get bandwidth details | GET | `/vpc/bandwidths/{bandwidth_id}` |
| Update bandwidth size | PUT | `/vpc/bandwidths/{bandwidth_id}` |
| Delete bandwidth | DELETE | `/vpc/bandwidths/{bandwidth_id}` |

Update bandwidth body:
```json
{
  "bandwidth": {
    "size": 10
  }
}
```

## Billing Modes

| `charge_mode` | Description | Notes |
|---------------|-------------|-------|
| `bandwidth` | 按带宽计费 | 固定月费，适合稳定流量 |
| `traffic` | 按流量计费 | 按 GB 收费，适合波动流量 |
| `bandwidth_metered` | 按带宽峰值计费 | 月末按峰值结算 |

## Common EIP Operations

| Scenario | Steps |
|----------|-------|
| 绑定到 ECS | 创建 EIP → 获取 ECS port_id → associate |
| 绑定到 ELB | 创建 EIP → 获取 ELB listener port_id → associate |
| 解绑并释放 | disassociate → 等待 5 分钟 → delete |
| 变更带宽大小 | 直接 PUT bandwidth size，无需解绑 |

## Query Filters

| Filter | Example |
|--------|---------|
| `publicip_id` | `/publicips?publicip_id={id}` |
| `ip` | `/publicips?ip=1.2.3.4` |
| `status` | `/publicips?status=DOWN` (未绑定) |
| `tags.key` | `/publicips?tags.key=env&tags.value=prod` |
