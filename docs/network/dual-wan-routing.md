# 双线出口与分流(BGP 黑洞通告 + mangle PBR)

2026-08-21 实测版。SOCKS 方案已废弃,当前机制为 **BGP 黑洞通告 + mangle PBR 负载均衡**。

## 一句话原理

- AR6140 双 PPPoE(电信 `Dialer1` + 移动 `Dialer2`)做 NAT 出口
- 家里 ROS 把 `CT` / `CM` / `blacklist` 三张地址表通过 BGP 通告给 AR
- AR 按表精确选线:CT 资源 → 电信线、CM 资源 → 移动线、blacklist → 回送 ROS
- ROS 把 blacklist 流量用 mangle PBR 均分到两条隧道,经日本 CHR 出公网

## 节点与角色

| 节点 | 系统 | 角色 |
|---|---|---|
| AR6140 | V300R024C00SPC100,`192.168.1.1`,AS 64527 | 双线出口、NAT、BGP 路由接收 |
| 家里 ROS | RouterOS 7.24 stable,`192.168.1.2`,AS 64523 | BGP 黑洞通告、mangle PBR、隧道端点 |
| 日本 CHR | RouterOS 7.24 stable,`202.144.194.248` | 隧道对端、公网出口 |

内网 `Vlanif1 192.168.1.0/24`,主 DNS `192.168.1.2`;`XGE0/0/0 = 192.168.2.1/30` 是 PC 10G 口直连段。

## BGP 设计

### 地址规划(2026-08-21 版,勿再用旧 251-254 规划)

| 通道 | AR 源(loopback) | ROS lo | AR import next-hop |
|---|---|---|---|
| lo-CT | 192.168.0.1(LoopBack0) | 192.168.0.2 | CT_IMPORT → 100.74.0.1(电信 BRAS,Dialer1) |
| lo-CM | 192.168.0.5(LoopBack1) | 192.168.0.6 | CM_IMPORT → 10.87.128.1(移动 BRAS,Dialer2) |
| lo-wg | 192.168.0.9(LoopBack2) | 192.168.0.10 | WG_IMPORT → 192.168.30.1 |
| lo-sstp | 192.168.0.13(LoopBack3) | 192.168.0.14 | SSTP_IMPORT → 192.168.20.1 |

### AR 侧

- 静态回程:4 条 peer `/32`、`192.168.20.0/30`、`192.168.30.0/30`、`192.168.40.0/30` → `192.168.1.2`;`202.144.194.248/32` → Dialer1+Dialer2
- export 策略 `DENY_ALL` 纯接收;`maximum load-balancing ebgp 2`
- 默认路由 2 条静态 + `track nqa admin ct/cm` 做线路探测

### ROS 侧

- 单 BGP instance(lo),`output.network=CT/CM/blacklist`(直接引用 address-list 名)+ `network-blackhole=yes` → 自动生成 `distance=255` 动态黑洞路由并通告
- 三张清单(2026-08-29 数据):`CT` 3081 条、`CM` 1493 条、`blacklist`(17 条 CIDR + DNS 联动动态条目)
- lo-wg / lo-sstp 两条通道都通告 blacklist → AR 收到同一网段的双 next-hop,靠 ebgp 负载均衡

## mangle PBR(blacklist 流量均分到双隧道)

```
prerouting, in-interface=ether6, dst-address-list=blacklist, connection-state=new
  + per-connection-classifier=both-addresses-and-ports:2/0 → mark-connection jp-wg-conn
  + per-connection-classifier=both-addresses-and-ports:2/1 → mark-connection jp-sstp-conn
→ mark-routing jp-wg / jp-sstp(passthrough=no)
→ 路由表 jp-wg / jp-sstp 默认路由分别走 wg-home / sstp-jp
→ srcnat:出隧道前改源为 192.168.30.1 / 192.168.20.1
```

## 隧道故障转移:递归公共 DNS 方案(已实施 + 演练验证)

!!! tip "原理"
    默认路由 next-hop 写公共 DNS IP(递归路由),main 表放锚点让网关递归解析到隧道对端;
    `check-gateway=ping` 每 10s 探测公共 DNS,**端到端**(含日本出公网)不通则主路由失效,
    `distance=2` 的 fallback 自动接管。

- main 表锚点:`1.1.1.1/32 → 192.168.30.2`(wg 腿)、`8.8.8.8/32 → 192.168.20.2`(sstp 腿),两条腿用不同探测目标区分
- jp-wg 表:`0.0.0.0/0 gateway=1.1.1.1 check-gateway=ping distance=1 target-scope=11`;fallback `0.0.0.0/0 via sstp-jp distance=2`
- jp-sstp 表:`0.0.0.0/0 gateway=8.8.8.8 check-gateway=ping distance=1 target-scope=11`;fallback `0.0.0.0/0 via wg-home distance=2`
- 故障注入演练通过:disable `1.1.1.1` 锚点 → 25s 内主路由失效、fallback 接管;恢复后自动回切

### 递归路由三大坑(实测)

1. **自定义 routing-table 里 gateway 的递归解析查 main 表,不是同表**——锚点必须放 main 表
2. 递归要求中间路由 `scope` < 引用路由 `target-scope`(默认 `target-scope=10` 装不下 `scope=30` 的静态锚点,需锚点 `scope=10` + 引用 `target-scope=11`)
3. `check-gateway=ping` 两连败(约 20s)才撤路由,两连胜才恢复

## 已知弱点

- **BGP 首包收敛窗口**:DNS 联动即时写 list,但 AR 学到路由要等 BGP 传播,新域名首包可能短暂走错线
- blacklist 里的大范围条目(如 `co.jp`)有误伤风险,排障见[坑与排障手册](pitfalls.md)

## 相关文档

- 隧道参数:[隧道速查](tunnels.md)
- DNS 链路:[DNS 防污染](dns-architecture.md)
