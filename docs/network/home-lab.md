# 家庭网络总览

2026-08-29 全节点实测核实（ROS + AR 逐台登录核对），与规则管线的衔接见 [ROS 规则管线](ros-rules-pipeline.md)。

## 出口与拓扑

```
互联网 ── 电信 PPPoE (Dialer1) ─┐
                                ├─ AR6140 (192.168.1.1, 华为 VRP) ── LAN 192.168.1.0/24
互联网 ── 移动 PPPoE (Dialer2) ─┘        │
                                        └─ ROS x86 (192.168.1.2, RouterOS 7.x, LAN 侧 ether6)
                                             ├─ 默认路由 → 192.168.1.1
                                             ├─ wg-home ──→ 日本 VPS 202.144.194.248:7138
                                             ├─ sstp-jp ──→ 日本侧（192.168.20.0 网段）
                                             └─ wg-cn ───→ 腾讯云 VPS 43.136.80.90 (192.168.40.1)
```

- 双拨：Dialer1 = 电信，Dialer2 = 移动（CMCC），均 `nat outbound 2001`、`adjust-mss 1200`
- AR 的 DNS 为 PPPoE 动态下发（电信 202.103.224.68/225.68 + 移动 211.138.240.100/245.180），无静态配置
- ROS 主 DNS 上游 = 192.168.1.1（AR）
- wg-port 调度器每分钟对 WireGuard 做端口跳跃

## BGP（loopback eBGP ×4，AS 64523 ↔ 64527）

| 邻居 | 本端 loopback | 路由表 | 宣告 network | AR PrefRcv (08-29) |
|---|---|---|---|---|
| lo-CT (192.168.0.1) | .2 (LoopBack0) | LO-CT | CT | 3084 |
| lo-CM (192.168.0.5) | .6 (LoopBack1) | LO-CM | CM | 1517 |
| lo-wg (192.168.0.9) | .10 (LoopBack2) | lo | blacklist | 1361 |
| lo-sstp (192.168.0.13) | .14 (LoopBack3) | lo | blacklist | 1361 |

- 全部 `network-blackhole=yes`，AR 侧入向 CT/CM/WG/SSTP_IMPORT、出向 DENY_ALL，ebgp-max-hop 2
- AR 路由表共 5989 前缀；blacklist 网段经 BGP 进 AR 后决定电信/移动双线选择

## DNS 实况

- 代理域名 FWD 静态条目 4085 条：**全部无 address-list，统一 `forward-to=DNS`**（命名转发器 = 192.168.20.2 / 192.168.30.2，日本隧道对端 DNS），`match-subdomain=yes`
- `blacklist` 地址表 1361 条中 **1357 条为动态条目**——由 DNS 侧（疑为日本节点上的 OxiDNS）观察解析结果后按 TTL 租约注入
- 分流入口在 mangle：从 ether6 进入、`dst-address-list=blacklist` 的新连接按 PCC（both-addresses-and-ports）2/0 → `jp-wg`、2/1 → `jp-sstp` 双隧道负载均衡

## 地址表现状（ROS 实测）

| 列表 | 条数 | 说明 |
|---|---|---|
| CN | 4638 | 中国大陆（当前为旧数据，待管线版本导入） |
| CT / CM | 3084 / 644 | 电信 / 移动 |
| blacklist | 1361 | 动态 1357 + 静态手工 IP |
| CU / CC | — | 尚未导入（管线已生成 cn-unicom/cn-cernet.rsc） |

## 已知问题与待办

- [ ] `proxy-domain.rsc`（5197 条单文件双层版）尚未导入，ROS 上还是旧手工 gfw.rsc 的 4085 条
- [ ] cn-unicom.rsc / cn-cernet.rsc 未导入（CU/CC 空缺）
- [ ] **jp-wg 路由表默认路由（gateway=1.1.1.1 via wg-home）INACTIVE**，fallback sstp-jp 生效中——双隧道负载均衡当前实际全走 sstp，wg-home 握手正常，需排查该路由的 check-gateway 写法
- [ ] AR6140 / ROS 完整配置导出归档
