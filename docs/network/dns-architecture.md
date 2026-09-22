# DNS 分流架构

当前基线核验于 **2026-09-22**。核心：DNS 服务由 **OxiDNS**（运行在 ROS 上的容器）接管，早期的 smartdns 容器已下线；**缓存只保留 OxiDNS 一层**。

## 完整链路

```
LAN 客户端（DHCP 下发 DNS = 192.168.1.2，备用 192.168.1.1）
      ↓
ROS dstnat：LAN 的 :53（udp+tcp）统一劫持到 192.168.1.3（OxiDNS）
      │  白名单 address-list=DNS 含 1.1 / 1.2 / 1.3 自身，避免上游查询被回环劫持
      ↓
OxiDNS（192.168.1.3）
      ├─ AAAA → 空应答（内网无 IPv6，直接压制）
      ├─ IP 测速优选 + 缓存（唯一一层）
      ├─ 命中「自家代理域名表」→ 远端：两条日本隧道对端（192.168.30.2 / 192.168.20.2，concurrent=2 双活）
      ├─ 非国内（cn 表补集反转）→ 同样走远端，避免名单外域名被污染
      └─ 其余（国内）→ 6 台国内 DNS 并发竞速
             └ 凡走远端的域名：解析结果自动注入 ROS 的 blacklist
      ↓
ROS mangle：dst-address-list=blacklist → 两条日本隧道（WG-CT / WG-CM 各半分流）
```

## 三个"入口地址"的分工

| 地址 | 角色 | 说明 |
|---|---|---|
| `192.168.1.2`（ROS） | dstnat 入口 | ROS 本身**不提供** DNS 服务（`allow-remote-requests=no`），LAN 查询被 dstnat 转到 OxiDNS；回程由 conntrack 反向改回源地址 |
| `192.168.1.1`（AR） | 代理入口 | AR 开 `dns proxy`，把客户端查询转发给它的 `dns server` 列表（静态首选 OxiDNS） |
| `192.168.1.3`（OxiDNS） | **唯一应答 + 唯一缓存** | 真正解析与缓存的地方 |

DHCP 下发的是 `192.168.1.2` + `192.168.1.1`（option 6 顺序即如此），**两个地址最终都落到 OxiDNS**。顺位上有差异：`1.2` 无兜底（ROS 只转发），`1.1` 有兜底（AR 代理超时后会回落 PPP 下发的运营商 DNS）。

## 关键点（勿改错）

- **入口劫持在 ROS**：`dstnat` 把 LAN 的 53 端口指向 OxiDNS。`address-list=DNS` 白名单**必须**包含 `192.168.1.3`——否则 OxiDNS 自己的上游查询会被同一条规则重新劫持回自己，形成回环（曾导致上游大面积超时、解析失败）
- **缓存只保留一层**（2026-09-22 定版）：ROS 的 `allow-remote-requests` 关闭，客户端查询全走 dstnat；缓存集中在 OxiDNS（`cache_main`：positive ≤86400、negative ≤300、落盘 `cache.dump`；另有 `ttl_main` 归一 60–86400）。理由：多级缓存 TTL 不同步、会掩盖 OxiDNS 的实时判据（IP 优选、代理/国内分流），并拖慢 blacklist 注入
- **不做"全网挟持"**（2026-09-22 决定）：现有挟持覆盖 DHCP 客户端、挂在 ROS LAN 侧的设备与 wg-cn 侧；直连 AR 的 GE 口设备若手工设外部 DNS 会绕开 OxiDNS。要在 AR 上补全网挟持需排除源 `1.3` 防回环（DoH/DoT 也无法挟持），收益有限，故维持现状
- **AR 侧 DNS 优先级**：`display dns server verbose` 显示静态 OxiDNS 为 `Priority 0`，PPP 下发的 4 条运营商 DNS 为 `Priority 1–4`（仅兜底）——**静态优先**，且 VRP 没有优先级配置命令。AR 自带的 DNS 缓存**没有开关**（只有 `dns application cache ttl maximum`，下限 3600 不可配），这层缓存保持现状
- **两条判据都指向远端**：① 自家代理域名表命中；② 不在国内表内（cn 反转）。前者保精度（有手工与剔除调优），后者保不漏
- **DNS 只解决"去哪解析"，流量走向靠 IP 列表**：远端解析出的 IP 写入 ROS `blacklist`，再由 mangle 决定隧道
- **不走 DNS 的服务**（Telegram 等客户端硬编码 IP）必须靠 IP 段兜底，见 [ROS 规则管线](ros-rules-pipeline.md)
- 国内上游池：电信 `202.103.224.68` / `202.103.225.68`、移动 `211.138.240.100` / `211.138.245.180`、阿里 `223.5.5.5`、腾讯 `119.29.29.29`；远端池 = 两条日本隧道对端

## 沿革

- 早期：ROS `/ip dns static type=FWD`（约 29016 条域名）+ smartdns 容器兜底优选
- 2026-09-16：smartdns 下线；OxiDNS 接管，域名表改为 Git 分发、每日自动更新
- 2026-09-21：日本出口迁到新节点后补齐 dnsmasq（原 CHR 自带 DNS 服务），并让 dnsmasq 同时监听两条隧道
- 2026-09-22：JP 上游改为两条隧道双活（`concurrent: 2`）；定版"缓存只保留 OxiDNS 一层"，ROS 关闭远程 DNS 请求
