# 家庭网络总览

!!! abstract "家庭网络架构速查"
    出口、选线、隧道、DNS 的分层总览。细节见右侧各篇。

## 节点一览

| 节点 | 系统 | 角色 |
|---|---|---|
| AR6140 | V300R024C00SPC100,`192.168.1.1`,AS 64527 | 双 PPPoE 电信+移动出口、NAT、BGP 路由接收 |
| 家里 ROS | RouterOS 7.24 stable,`192.168.1.2`,AS 64523 | BGP 黑洞通告、mangle PBR 分流、DNS、隧道端点 |
| 日本 VPS | RouterOS CHR 7.24,`202.144.194.248` | sstp-jp / wg-home 隧道对端、公网出口 |
| 广州 VPS | Debian 13,`43.136.80.90` | wg-cn 管理隧道、规则生成管线 |

## 拓扑

```
互联网 ── 电信 PPPoE (Dialer1) ─┐
                                ├─ AR6140 (192.168.1.1) ── LAN 192.168.1.0/24
互联网 ── 移动 PPPoE (Dialer2) ─┘     │
                                      └─ ROS x86 (192.168.1.2, LAN 侧 ether6)
                                           ├─ 默认路由 → 192.168.1.1
                                           ├─ wg-home ──→ 日本 VPS 202.144.194.248:7138
                                           ├─ sstp-jp ──→ 日本侧（192.168.20.0 网段）
                                           └─ wg-cn ───→ 腾讯云 VPS 43.136.80.90 (192.168.40.1)
```

- 双拨:Dialer1 = 电信、Dialer2 = 移动(CMCC),均 `nat outbound 2001`、`adjust-mss 1200`
- AR 的 DNS 为 PPPoE 动态下发(电信 202.103.224.68/225.68 + 移动 211.138.240.100/245.180),无静态配置
- wg-port 调度器每分钟对 WireGuard 做端口跳跃

## 分流体系

- **选线**:电信/移动资源精确走对应线路,blacklist 回送 ROS 经隧道出公网([双线出口与分流](dual-wan-routing.md))
- **域名层**:ROS `/ip dns static` FWD,`match-subdomain=yes`,命中打标 `blacklist`([DNS 防污染](dns-architecture.md))
- **IP 层**:`/ip firewall address-list` 的 CN / CT / CM / CU / CC / blacklist([ROS 规则管线](ros-rules-pipeline.md))
- **隧道**:sstp-jp / wg-home(日本)+ wg-cn(广州)([隧道速查](tunnels.md))

## 2026-08-29 全节点实测(BGP 与地址表基线)

| 邻居 | 本端 loopback | 路由表 | 宣告 network | AR PrefRcv |
|---|---|---|---|---|
| lo-CT (192.168.0.1) | .2 (LoopBack0) | LO-CT | CT | 3084 |
| lo-CM (192.168.0.5) | .6 (LoopBack1) | LO-CM | CM | 1517 |
| lo-wg (192.168.0.9) | .10 (LoopBack2) | lo | blacklist | 1361 |
| lo-sstp (192.168.0.13) | .14 (LoopBack3) | lo | blacklist | 1361 |

- 全部 `network-blackhole=yes`;AR 入向 CT/CM/WG/SSTP_IMPORT、出向 DENY_ALL,`ebgp-max-hop 2`
- AR 路由表共 5989 前缀;blacklist 网段进 AR 后决定电信/移动双线选择

ROS 地址表现状(08-29):CN 4638(待管线新版本导入)、CT/CM 3084/644、blacklist 1361(动态 1357 + 静态手工)、CU/CC 尚未导入。

## 已知问题与待办(08-29)

- [ ] `proxy-domain.rsc`(5197 条单文件双层版)尚未导入,ROS 上仍是旧手工 gfw.rsc 的 4085 条
- [ ] cn-unicom.rsc / cn-cernet.rsc 未导入(CU/CC 空缺)
- [ ] **jp-wg 路由表默认路由(gateway=1.1.1.1 via wg-home)INACTIVE**,fallback sstp-jp 生效中——双隧道负载均衡当前实际全走 sstp,wg-home 握手正常,需排查该路由的 check-gateway 写法
- [ ] AR6140 / ROS 完整配置导出归档
- [ ] VLAN 划分与接口表
- [ ] 防火墙规则清单与用途说明
- [ ] 巡检基线与脚本清单
