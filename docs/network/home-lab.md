# 家庭网络总览

当前基线最后核验于 **2026-09-22**。原始配置和回滚快照仅保存在私有备份中，不进入公开仓库。规则管线见 [ROS 规则管线](ros-rules-pipeline.md)，历史原因见 [复盘笔记](../blog/index.md)。

## 拓扑总览

```
电信 PPPoE (Dialer1, GE0/0/12) ──┐
                                  ├─ AR6140 (192.168.1.1, AS 64527)
移动 PPPoE (Dialer2, GE0/0/13) ──┘    │ Vlanif1: 192.168.1.0/24 (DHCP)
                                       │ XGE0/0/0: 192.168.2.1/30 ── 192.168.2.2（工作机）
                                       │ GE0/0/1 ← x86，入向挂目的端口 PBR
                                  ROS x86 (192.168.1.2, AS 64523, ether6=LAN)
                                       ├─ wg-ct (192.168.20.0/30) ──→ 日本节点，锁电信
                                       ├─ wg-cm (192.168.30.0/30) ──→ 日本节点，锁移动
                                       ├─ wg-cn (192.168.40.0/24) ──→ 广州 VPS（管理隧道）
                                       └─ OxiDNS 容器 192.168.1.3（DNS 分流，见 DNS 分流架构）
```

- LAN 客户端 DHCP（AR global pool）：`192.168.1.30-125`，排除 `.2-.25`，**DNS 首选 192.168.1.2（ROS）**、备 192.168.1.1（两条入口最终都被引到 OxiDNS 192.168.1.3，见 [DNS 分流架构](dns-architecture.md)）
- AR↔ROS 两条互联：Vlanif1 二层（192.168.1.0/24）+ XGE 三层（192.168.2.0/30，ROS 侧经 AR 中转）
- AR `Vlanif1` ↔ ROS `br-lan` 运行 OSPFv2 Area 0，双方接口均为点对点网络类型；只重分发 `192.168.0.0/16` 范围内的直连路由

## AR6140 详设（AS 64527）

- **双拨双默认路由 ECMP**：`0/0 → Dialer1 track nqa ct`（探测电信网关）+ `→ Dialer2 track nqa cm`（探测移动网关），NQA 15s 间隔、2 次失败撤线
- **目的端口 PBR**（2026-09-22 新增）：`GE0/0/1` 入向挂 `traffic-policy p-lb2`，按两条隧道各自的动态端口段 `redirect ip-nexthop` 到电信 / 移动出口 —— 这是"两条隧道各锁一条线"的实现点。该 lan-board **只支持 `redirect ip-nexthop`**（`redirect interface` 会被拒），详见[双线出口与分流](dual-wan-routing.md)
- **NAT**：ACL 2001（192.168.1.0/24 + 192.168.2.0/30），**endpoint-independent** mapping/filter（近似 full-cone，对游戏/P2P 友好），ALG：dns/ftp/rtsp/sip/pptp
- **BGP 出向重写**：`CT_IMPORT` / `CM_IMPORT` / `CM-WG_IMPORT`（→ `192.168.30.1`，wg-cm）/ `WG-CT_IMPORT`（→ `192.168.20.1`，wg-ct）把 ROS 宣告的业务路由下一跳分别改写成对应线路——**"哪个邻居学来的就从哪条线出去"的实现核心**；出向统一 `DENY_ALL`
- **内部路由**：ROS 的 loopback 与隧道直连前缀由 OSPF 学习，已清理对应的遗留静态路由；公网隧道端点仍由双出口保证可达
- **DNS**：`dns server 192.168.1.3`（静态，`Priority 0` 最优先）+ `dns proxy enable`（客户端的备用入口）
- **管理面**：管理协议按可信来源和接口限制；具体入口、账号与端口不在公开文档披露
- 加固：`undo icmp timestamp-request`、drop illegal-mac alarm、NTP cn.pool + ntp.ntsc.ac.cn

## ROS 详设（AS 64523）

- **地址**：lo 多播（192.168.0.2/.6/.10/.14，BGP 对端）、ether6 192.168.1.2/24（LAN）、wg-ct 192.168.20.1/30、wg-cm 192.168.30.1/30、wg-cn 192.168.40.2/24
- **管理服务**：仅用于内网与管理隧道，具体暴露面以设备实时配置为准；管理入口不在公开文档披露
- **NAT**：出 wg-ct / wg-cm 分别伪装为 `192.168.20.1` / `192.168.30.1`，出 wg-cn 伪装为 `192.168.40.2`
- **策略路由（递归锚点设计）**：`WG-CT` / `WG-CM` 两张表，默认路由 next-hop 写公共 DNS（递归路由），`check-gateway=ping` 探测；main 表放锚点指向对应隧道对端；两表互为主备（各带 `distance=2` 指向另一条隧道的路由）
- **DNS**：`servers = 192.168.1.3`（OxiDNS）；**`allow-remote-requests = no`**（对外不提供解析，只保留本机解析与本地缓存——全网缓存只在 OxiDNS 一层）；`address-list-extra-time=1w`
- **黑名单静态段**：Telegram、Instagram/Meta 等客户端硬编码 IP 的服务，以 IP 段兜底（详见地址表现状）
- **脚本**：`wg`（每分钟由 `wg-port` 调度）：以三条隧道各自的收发字节数 + 时间做熵，映射到两段端口池，分别改写 `wg-ct` / `wg-cm` / `wg-cn` peer 的 `endpoint-port`（抗封锁端口跳跃）。⚠️ 写入脚本时 `$var` 必须转义成 `\$var`，否则会被当场展开成空
- **WireGuard 保活**：三个 peer 统一 `persistent-keepalive=25s`
- traffic-flow / socks / container 功能已启用（观测/扩展用）

## DNS 实况与分流闭环

1. LAN 客户端 → 下发 DNS（`192.168.1.2`，备 `192.168.1.1`）→ 两条入口最终都到 **OxiDNS 192.168.1.3**
2. OxiDNS：命中代理域名表或"非国内" → 两条日本隧道对端并发（`concurrent: 2`）远端解析；其余走国内 6 台并发竞速；AAAA 直接空应答
3. 走远端的域名：解析结果自动注入 ROS `blacklist`（`address-list-extra-time=1w` 延长驻留）
4. mangle：从 ether6 进入、目的在 `blacklist` 的新连接按 PCC（both-addresses-and-ports）`2/0`→`WG-CT`、`2/1`→`WG-CM`，各走一条隧道；同时这些段经 BGP 宣告给 AR 做**双线择优出境**

## 地址表现状（管线整表重建）

| 列表 | 条数 | 说明 |
|---|---|---|
| CN | 6223 | 中国大陆（管线整表重建） |
| CT / CM | 3081 / 1493 | 电信 / 移动（= BGP 宣告数） |
| CU / CC | 1907 / 396 | 联通 / 教育网 |
| blacklist | 静态 + 动态 | TG/Meta/Twitter/MikroTik 静态（ros-rules-auto）+ OxiDNS 远端解析动态注入 |
| not_global / bad_* / no_forward | 8/7/4 | defconf RFC6890 保留段 |

## 已知问题与待办

- [x] 静态路由迁移 OSPF 后 BGP ECMP 消失：根因为 SSTP PPP `/32` 与遗留下一跳造成 IGP cost 不等，已改用 PPP 对端下一跳并恢复 ECMP，见[复盘](../blog/posts/bgp-ecmp-after-ospf.md)
- [x] ~~jp-wg 默认路由间歇翻动~~ **确认为设计内行为**：跳端口改写的瞬间 `check-gateway` 探测短暂失败、路由翻到 fallback，握手恢复后自愈；已有连接靠 connection-mark 不受影响
- [x] 2026-09-21：日本出口迁到新节点（Debian 13），补齐 dnsmasq 与入口端口重定向；修正隧道表默认路由网关必须落在直连网段（否则整条路由 INACTIVE）
- [x] 2026-09-22：建成双隧道并让两条分别锁定电信 / 移动（AR 目的端口 PBR + eBGP 等价路由），见[复盘](../blog/posts/dual-wg-egress-pinning.md)
- [ ] **PBR 无失败回退**：`redirect ip-nexthop` 写死、没有 backup，家宽故障或运营商重拨导致网关变化时，对应端口段（整条隧道）会黑洞。候选改法 `backup-nexthop` + NQA track，需先验证该 lan-board 是否支持
- [ ] 192.168.2.2 服务器未纳入巡检
