# 家庭网络总览

2026-08-29 基于**全量配置导出**（ROS `/export show-sensitive` 781KB + AR `display current-configuration`）逐项核实，原始配置归档于 VPS `/root/config-backups/2026-08-29/`（含密钥，勿外传）。规则管线见 [ROS 规则管线](ros-rules-pipeline.md)。

## 拓扑总览

```
电信 PPPoE (Dialer1, GE0/0/12) ──┐
                                  ├─ AR6140 (192.168.1.1, AS 64527)
移动 PPPoE (Dialer2, GE0/0/13) ──┘    │ Vlanif1: 192.168.1.0/24 (DHCP)
                                       │ XGE0/0/0: 192.168.2.1/30 ── 192.168.2.2（服务器，疑 OxiDNS）
                                       │
                                  ROS x86 (192.168.1.2, AS 64523, ether6=LAN)
                                       ├─ wg-home ──→ 日本 VPS 202.144.194.248:端口跳跃(3258-10240)
                                       ├─ sstp-jp ──→ 202.144.194.248:443（192.168.20.0/30）
                                       └─ wg-cn ───→ 腾讯云 VPS 43.136.80.90（192.168.40.0/24，管理）
```

- LAN 客户端 DHCP（AR global pool）：`192.168.1.30-125`，排除 `.2-.25`，**DNS 首选 192.168.1.2（ROS）**、备 192.168.1.1
- AR↔ROS 两条互联：Vlanif1 二层（192.168.1.0/24）+ XGE 三层（192.168.2.0/30，ROS 侧经 AR 中转）

## AR6140 详设（AS 64527）

- **双拨双默认路由 ECMP**：`0/0 → Dialer1 track nqa ct`（探测电信网关 100.74.0.1）+ `→ Dialer2 track nqa cm`（探测 10.87.128.1），NQA 15s 间隔、2 次失败撤线
- **NAT**：ACL 2001（192.168.1.0/24 + 192.168.2.0/30），**endpoint-independent** mapping/filter（近似 full-cone，对游戏/P2P 友好），ALG：dns/ftp/rtsp/sip/pptp
- **BGP 出向重写**：CT_IMPORT/CM_IMPORT/WG_IMPORT/SSTP_IMPORT 把 ROS 宣告的 /32 黑洞路由的下一跳分别改写为 100.74.0.1（电信出口）/ 10.87.128.1（移动出口）/ 192.168.20.1（sstp）/ 192.168.30.1（wg）——**这是"哪个邻居学来的就从哪条线出去"的实现核心**；出向统一 DENY_ALL
- 静态路由：4 个 ROS loopback /32 + 20.0/30、30.0/30、40.0/24 均指 192.168.1.2；**202.144.194.248/32 双线等价**（日本 VPS 走任意出口均可达）
- 管理：SSH 52222（all 接口）、HTTPS 仅 acl 2998（192.168.2.2 / 192.168.1.24）、FTP 仅 acl 2999、用户 Tlop level 15、SNMPv3（Vlanif1）
- 加固：`undo icmp timestamp-request`、drop illegal-mac alarm、NTP cn.pool + ntp.ntsc.ac.cn

## ROS 详设（AS 64523）

- **地址**：lo 多播（192.168.0.2/.6/.10/.14，BGP 对端）、ether6 192.168.1.2/24（LAN）、wg-cn 192.168.40.2/24、wg-home 192.168.30.1/30
- **服务**：仅 ssh 52222 + winbox 52323（其余全禁）；**无 /ip firewall filter 规则**——安全依赖 AR NAT 与内网信任，属设计取舍
- **NAT**：出 wg-home 伪装为 192.168.30.1、出 sstp-jp 伪装为 192.168.20.1
- **策略路由（递归锚点设计）**：
    - `jp-wg`：默认 gateway=1.1.1.1（check-gateway=ping）⇐ 递归路由 `1.1.1.1/32 via 192.168.30.2`（wg-home 对端 DNS）⇐ wg-home 出隧道
    - `jp-sstp`：默认 gateway=8.8.8.8 ⇐ 递归路由 `8.8.8.8/32 via 192.168.20.2` ⇐ sstp-jp 出隧道
    - 各带 fallback（wg 挂→sstp，sstp 挂→wg）
- **DNS**：主上游 192.168.1.1；缓存 1GB；`address-list-extra-time=1w`（DNS 派生的动态地址条目在解析停止后仍保留一周）；命名转发器 `DNS` = 192.168.20.2 + 192.168.30.2（日本侧）
- **黑名单静态 4 条**：91.108.0.0/16 + 149.154.160.0/20（Telegram）、5.28.192.0/18 + 109.239.140.0/24（Instagram/Meta）
- **脚本**：`wg`（每分钟由 wg-port 调度）：以 wg-home 收发字节数+时间做熵，映射到 1024-10240 端口，改写 wg-home peer 的 endpoint-port（对抗封锁的端口跳跃）
- traffic-flow / socks / container 功能已启用（观测/扩展用）

## DNS 实况与分流闭环

1. LAN 客户端 → ROS DNS：普通域名 → 192.168.1.1（AR）→ ISP；**代理域名（FWD 4085 条）→ 转发器 "DNS"（日本侧 DNS）**，全部 `match-subdomain=yes`、不带 address-list
2. DNS 侧（疑 192.168.2.2 上的 OxiDNS 及/或日本节点）观察解析，把得到的 IP **动态写入 ROS `blacklist` 地址表**（当前 1357 条动态 + 4 条静态，`address-list-extra-time=1w` 延长驻留）
3. mangle：从 ether6 进入、目的在 `blacklist` 的新连接按 PCC（both-addresses-and-ports）2/0→`jp-wg`、2/1→`jp-sstp`
4. 策略路由出日本隧道；同时这些网段经 BGP（lo-wg/lo-sstp）宣告给 AR，AR 用 route-policy 重写下一跳实现**双线择优出境**

## 地址表实测（配置导出修正版；此前 print 因换行折行统计偏小）

| 列表 | 条数 | 说明 |
|---|---|---|
| CN | 5615 | 中国大陆（静态） |
| CT / CM | 3084 / 1517 | 电信 / 移动（= BGP 宣告数，AR PrefRcv 完全一致） |
| blacklist | 4 静态 + 1357 动态 | TG/Meta 网段 + DNS 动态注入 |
| not_global / bad_* / no_forward | 8/7/4 | defconf RFC6890 保留段 |

## 已知问题与待办

- [ ] **jp-wg 默认路由 INACTIVE（当前双隧道负载均衡实际全走 sstp）**：check-gateway ping 1.1.1.1 经 wg-home 不通（隧道握手正常、rx 11.7GiB），疑日本侧不放行转发 ICMP；可把锚点换成 `1.0.0.1` 或改用递归到 192.168.30.2 自身
- [ ] `proxy-domain.rsc`（5197 条）未导入，ROS FWD 仍是旧手工版 4085 条
- [ ] cn-unicom.rsc / cn-cernet.rsc 未导入（CU/CC 空缺）
- [ ] 192.168.2.2 服务器（疑 OxiDNS 所在）未纳入巡检
