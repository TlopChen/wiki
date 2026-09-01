# 家庭网络总览

当前基线最后核验于 **2026-09-01**。原始配置和回滚快照仅保存在私有备份中，不进入公开仓库。规则管线见 [ROS 规则管线](ros-rules-pipeline.md)，历史原因见 [复盘笔记](../blog/index.md)。

## 拓扑总览

```
电信 PPPoE (Dialer1, GE0/0/12) ──┐
                                  ├─ AR6140 (192.168.1.1, AS 64527)
移动 PPPoE (Dialer2, GE0/0/13) ──┘    │ Vlanif1: 192.168.1.0/24 (DHCP)
                                       │ XGE0/0/0: 192.168.2.1/30 ── 192.168.2.2（服务器，疑 OxiDNS）
                                       │
                                  ROS x86 (192.168.1.2, AS 64523, ether6=LAN)
                                       ├─ wg-home ──→ 日本 VPS（动态端口）
                                       ├─ sstp-jp ──→ 日本 VPS（PPP 点对点）
                                       └─ wg-cn ───→ 广州 VPS（管理隧道）
```

- LAN 客户端 DHCP（AR global pool）：`192.168.1.30-125`，排除 `.2-.25`，**DNS 首选 192.168.1.2（ROS）**、备 192.168.1.1
- AR↔ROS 两条互联：Vlanif1 二层（192.168.1.0/24）+ XGE 三层（192.168.2.0/30，ROS 侧经 AR 中转）
- AR `Vlanif1` ↔ ROS `br-lan` 运行 OSPFv2 Area 0，双方接口均为点对点网络类型；只重分发 `192.168.0.0/16` 范围内的直连路由

## AR6140 详设（AS 64527）

- **双拨双默认路由 ECMP**：`0/0 → Dialer1 track nqa ct`（探测电信网关 100.74.0.1）+ `→ Dialer2 track nqa cm`（探测 10.87.128.1），NQA 15s 间隔、2 次失败撤线
- **NAT**：ACL 2001（192.168.1.0/24 + 192.168.2.0/30），**endpoint-independent** mapping/filter（近似 full-cone，对游戏/P2P 友好），ALG：dns/ftp/rtsp/sip/pptp
- **BGP 出向重写**：CT_IMPORT/CM_IMPORT/WG_IMPORT/SSTP_IMPORT 把 ROS 宣告的业务路由下一跳分别改写为电信出口、移动出口、`192.168.30.1`（WG）和 `192.168.20.2`（SSTP 对端）——**这是“哪个邻居学来的就从哪条线出去”的实现核心**；出向统一 DENY_ALL
- **内部路由**：ROS 的 loopback 与隧道直连前缀由 OSPF 学习，已清理对应的遗留静态路由；公网隧道端点仍由双出口保证可达
- **管理面**：管理协议按可信来源和接口限制；具体入口、账号与端口不在公开文档披露
- 加固：`undo icmp timestamp-request`、drop illegal-mac alarm、NTP cn.pool + ntp.ntsc.ac.cn

## ROS 详设（AS 64523）

- **地址**：lo 多播（192.168.0.2/.6/.10/.14，BGP 对端）、ether6 192.168.1.2/24（LAN）、wg-cn 192.168.40.2/24、wg-home 192.168.30.1/30
- **管理服务**：仅用于内网与管理隧道，具体暴露面以设备实时配置为准；管理入口不在公开文档披露
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

1. LAN 客户端 → ROS DNS：普通域名 → 192.168.1.1（AR）→ ISP；**代理域名（FWD 5197 条，2026-08-29 起为管线单文件双层版）→ 转发器 "DNS"（日本侧 DNS）**，全部 `match-subdomain=yes`、不带 address-list
2. ROS 原生机制完成"感知"：FWD 静态条目带 `address-list=blacklist` 参数，**域名一旦被解析，得到的 IP 自动作为动态条目写入 blacklist**（`address-list-extra-time=1w` 延长驻留，当前 ~1476 条动态）——无需任何外部 DNS，纯 RouterOS 内建能力
3. mangle：从 ether6 进入、目的在 `blacklist` 的新连接按 PCC（both-addresses-and-ports）2/0→`jp-wg`、2/1→`jp-sstp`
4. 策略路由出日本隧道；同时这些网段经 BGP（lo-wg/lo-sstp）宣告给 AR，AR 用 route-policy 重写下一跳实现**双线择优出境**

## 地址表现状（2026-08-29 管线版已全部导入 ROS）

| 列表 | 条数 | 说明 |
|---|---|---|
| CN | 6223 | 中国大陆（管线整表重建） |
| CT / CM | 3081 / 1493 | 电信 / 移动（管线整表重建，= BGP 宣告数） |
| CU / CC | 1907 / 396 | 联通 / 教育网（新建） |
| blacklist | 18 静态 + ~1476 动态 | TG/Meta/Twitter/MikroTik 静态（ros-rules-auto）+ DNS 动态注入 |
| not_global / bad_* / no_forward | 8/7/4 | defconf RFC6890 保留段 |

> 导出时旧值为 CN 5615 / CT 3084 / CM 1517 / blacklist 4 静态（仅 TG/Meta），已被管线版本整表替换。

## 已知问题与待办

- [x] 静态路由迁移 OSPF 后 BGP ECMP 消失：根因为 SSTP PPP `/32` 与遗留下一跳造成 IGP cost 不等，已改用 PPP 对端下一跳并恢复 ECMP，见[复盘](../blog/posts/bgp-ecmp-after-ospf.md)
- ~~jp-wg 默认路由间歇翻动~~ **确认为设计内行为，无问题**（2026-08-29 用户确认）：wg-port 每分钟改写 endpoint-port 的瞬间 check-gateway 探测短暂失败、路由翻到 fallback，握手恢复后自愈；已有连接靠 connection-mark 不受影响。无需修复
- [x] proxy-domain.rsc（5197 条）已导入，ROS DNS 静态 4085 → 5197
- [x] cn-unicom / cn-cernet 已导入（CU=1907 / CC=396），CN/CT/CM 已按管线数据整表重建
- [ ] 192.168.2.2 服务器（疑 OxiDNS 所在）未纳入巡检
