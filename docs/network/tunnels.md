# 隧道速查

家宽节点与两台 VPS 之间的隧道参数。GM / VPN 相关隧道按当地法规使用,仅个人网络研究用途。

| 隧道 | 网段 | 家里端 | 对端 | 端口/备注 |
|---|---|---|---|---|
| sstp-jp | 192.168.20.0/30 | .1 | 日本 CHR .2 | 定时重连保活 |
| wg-home | 192.168.30.0/30 | .1 | 日本 CHR .2 | 日本 listen 6812,家里 endpoint 指向 202.144.194.248:7138(实测);公网 1024–10240 redirect 跳端口;家里 scheduler 每分钟随机化 endpoint-port 防封 |
| wg-cn | 192.168.40.0/24 | .2 | 广州 VPS .1 | **固定 UDP 51820**(腾讯云有平台 PAT,跳端口方案失效) |

## 回程路由

| 节点 | 回程 |
|---|---|
| 日本 CHR | `192.168.1.0/24`、`192.168.2.0/30` 走 sstp + wg ECMP 回家 |
| 广州 VPS | `192.168.1.0/24` 走 wg-cn(peer AllowedIPs 含该段 + PostUp 加路由) |
| AR6140 | `192.168.40.0/24 → 192.168.1.2`(已 save) |

## 广州 VPS(wg-cn,Debian 13)

- WireGuard 配置:`/etc/wireguard/wg-cn.conf`,listen-port 51820
- peer AllowedIPs = `192.168.40.2/32 + 192.168.1.0/24`(回程全家 LAN)
- PostUp 回程路由:`192.168.1.0/24 via 192.168.40.2 dev wg-cn`
- nftables 默认 drop,input 放行 52222 + 51820 + ICMP echo(否则 ping 单向假象)

## 日本 VPS(RouterOS CHR,非 Linux)

- 系统:RouterOS CHR 7.24 stable,身份名 CHR,SSH 52222
- 公网入站仅允许中国大陆 IP 段(src-address-list=450000)访问 52222/52323/443 与 ICMP
- WG 端口隐藏:nat dstnat 把公网 UDP 1024–10240 redirect 到 6812(防探测)
- 防火墙:input/forward 规则全限定 `in-interface-list=WAN`(ether1);sstp/wg 在 VPN 接口列表,隧道内 ICMP 默认 accept
- 保活设计(用户定,勿再加 netwatch/BFD):
    - 日本 CHR 定时 `remove_ppp_active` 重连 SSTP
    - 家里 ROS 靠 wg-port scheduler 每分钟随机化 `wg-home` endpoint-port

## 为什么不做双 WG 双线

探索结论(已废案):单条 WG = 单 UDP 流,AR ECMP 按流 hash,单流永远单线(NAT 会话绑定);
第二 IP / VRF 方案也不可行。用户最终放弃,保持单 WG 现状。

## 为什么日本能跳端口、广州不能

- 日本 CHR 无云平台 NAT,公网直达,redirect 跳端口有效
- 腾讯云轻量有平台 PAT:外部 UDP 入站会被改源端口(如 `113.15.161.206:25819`),redirect 计数不涨或涨了不投递;只能固定端口

## 相关文档

- 分流逻辑:[双线出口与分流](dual-wan-routing.md)
- DNS 链路:[DNS 防污染](dns-architecture.md)
- 坑与排障:[坑与排障手册](pitfalls.md)
