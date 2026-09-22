# 隧道速查

家宽节点与两台 VPS 之间的隧道参数。GM / VPN 相关隧道按当地法规使用，仅个人网络研究用途。

| 隧道 | 网段 | 家里端 | 对端 | 出口归属 | 端口/备注 |
|---|---|---|---|---|---|
| `wg-ct` | 192.168.20.0/30 | .1 | 日本节点 .2 | **锁电信** | 动态 endpoint 端口；公网跳端口段与实际监听端口不公开 |
| `wg-cm` | 192.168.30.0/30 | .1 | 日本节点 .2 | **锁移动** | 同上（原 `wg-home`，2026-09-22 更名） |
| `wg-cn` | 192.168.40.0/24 | .2 | 广州 VPS .1 | — | 固定 UDP 端口（云平台 PAT 下不使用跳端口） |

`sstp-jp` 已于 2026-09-21 随日本节点迁移退役（原 PPP 点对点隧道），现在是**两条独立 WG 隧道**并行，各自锁定一条家宽出口。

## 出口归属是怎么做到的

两条隧道各自都是一条 UDP 流。早期结论是"单流永远单线、ECMP 按流哈希分不出确定性"，实测也确认了 AR 的 ECMP 端口→线路映射不可预测。真正生效的做法是：

- 两条隧道**各用一段独立的动态跳端口**（段内每分钟随机化，保留抗封锁特性）；
- **AR6140 在 x86 入口口上按目的端口段做策略路由**，把两段分别重定向到电信/移动的 PPP 出口；
- 于是"哪条隧道走哪条线"由策略决定，不再依赖 ECMP 哈希。

详见[双线出口与分流](dual-wan-routing.md)，以及复盘[《两条 WG 隧道怎样各锁一条出口》](../blog/posts/dual-wg-egress-pinning.md)。

## 回程路由

| 节点 | 回程 |
|---|---|
| 日本节点 | `192.168.1.0/24`、`192.168.2.0/30` 经两条 WG + masquerade 回内网 |
| 广州 VPS | `192.168.1.0/24` 走 wg-cn(peer AllowedIPs 含该段 + PostUp 加路由) |
| AR6140 | 隧道直连前缀由 ROS 通过 OSPF 发布 |

## 日本节点（Debian 13，2026-09-21 起）

- 系统：Debian 13，WireGuard 走 `wg-quick@wg-ct` / `wg-quick@wg-cm`（配置在 `/etc/wireguard/`）
- 入口：nftables `ip nat prerouting` 把两条动态端口段分别 `redirect` 到各自的内部监听端口；`input` 默认 DROP，只放业务端口与 ICMP/ICMPv6
- 转发：`ip_forward=1`、出 `eth0` masquerade、双向 **MSS 自动钳制**（`set rt mtu`）
- DNS：dnsmasq 同时 `interface=wg-ct` 与 `interface=wg-cm`，两条隧道各暴露一个上游地址供内网 OxiDNS 双活
- 保活：家里 ROS 的 `wg` 脚本每分钟改写两条隧道的 `endpoint-port`；两侧 peer `persistent-keepalive=25s`
- 迁移前该角色由 RouterOS CHR 承担（单 `wg0` + SSTP），CHR 已退订、配置零残留

## 广州 VPS(wg-cn,Debian 13)

- WireGuard 配置:`/etc/wireguard/wg-cn.conf`，使用固定监听端口
- peer AllowedIPs = `192.168.40.2/32 + 192.168.1.0/24`(回程全家 LAN)
- PostUp 回程路由:`192.168.1.0/24 via 192.168.40.2 dev wg-cn`
- nftables 默认 drop，仅放行受控管理来源、WireGuard 与必要 ICMP
- peer `persistent-keepalive=25s`（2026-09-22 与日本两条统一）

## 为什么日本能跳端口、广州不能

- 日本节点无云平台 NAT，公网直达，redirect 跳端口有效
- 腾讯云轻量有平台 PAT：外部 UDP 入站会被改源端口，redirect 计数不涨或涨了不投递；只能固定端口

## 相关文档

- 分流逻辑:[双线出口与分流](dual-wan-routing.md)
- DNS 链路:[DNS 防污染](dns-architecture.md)
- 坑与排障:[坑与排障手册](pitfalls.md)
- 复盘:[两条 WG 隧道怎样各锁一条出口](../blog/posts/dual-wg-egress-pinning.md)
