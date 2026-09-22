# 双线出口与分流(BGP 黑洞通告 + AR 目的端口 PBR + mangle PBR)

当前基线最后核验于 **2026-09-22**。机制沿革：SOCKS 中转 → BGP 黑洞通告 + mangle PBR → （2026-09-22）**两条 WG 隧道各自锁定电信/移动**。

## 一句话原理

- AR6140 双 PPPoE（电信 `Dialer1` + 移动 `Dialer2`）做 NAT 出口
- 家里 ROS 把 `CT` / `CM` / `blacklist` 三张地址表通过 BGP 通告给 AR
- AR 按表精确选线：CT 资源 → 电信线、CM 资源 → 移动线、blacklist → 回送 ROS
- ROS 把 blacklist 流量用 mangle PBR 按连接均分到**两条 WG 隧道**（`wg-ct` / `wg-cm`）
- 两条隧道各用**一段独立的动态跳端口**，AR 在 x86 入口口上按目的端口段做策略路由，把它们分别锁到电信 / 移动出口

## 节点与角色

| 节点 | 系统 | 角色 |
|---|---|---|
| AR6140 | V300R024C00SPC100,`192.168.1.1`,AS 64527 | 双线出口、NAT、BGP 路由接收、**目的端口 PBR** |
| 家里 ROS | RouterOS 7.24 stable,`192.168.1.2`,AS 64523 | BGP 黑洞通告、mangle PBR、隧道端点 |
| 日本节点 | Debian 13（2026-09-21 起，原 RouterOS CHR） | 两条隧道对端、公网出口、dnsmasq |

内网 `Vlanif1 192.168.1.0/24`,主 DNS `192.168.1.2`;`XGE0/0/0 = 192.168.2.1/30` 是 PC 10G 口直连段。

## BGP 设计

### 地址规划（2026-09-22 版）

| 通道 | AR 源(loopback) | ROS lo | AR import policy | AR 导入 next-hop | 含义 |
|---|---|---|---|---|---|
| lo-CT | 192.168.0.1(LoopBack0) | 192.168.0.2 | `CT_IMPORT` | `100.74.0.1` | 电信直连 |
| lo-CM | 192.168.0.5(LoopBack1) | 192.168.0.6 | `CM_IMPORT` | `10.87.128.1` | 移动直连 |
| lo-CT-WG | 192.168.0.9(LoopBack2) | 192.168.0.10 | **`CM-WG_IMPORT`** | `192.168.30.1` | 经 **wg-cm** 出日本 |
| lo-CM-WG | 192.168.0.13(LoopBack3) | 192.168.0.14 | **`WG-CT_IMPORT`** | `192.168.20.1` | 经 **wg-ct** 出日本 |

> 2026-09-22 调整：原 `WG_IMPORT` / `SSTP_IMPORT` 更名为 `CM-WG_IMPORT` / `WG-CT_IMPORT`。VRP **不支持 route-policy 改名**，实际操作是"建新 policy → 改 peer 引用 → 删旧 → `refresh bgp all import` 软复位"。SSTP 通道已随节点迁移退役。

### ECMP 等价条件（重要）

两条 eBGP 路由要合并为等价路由，**next-hop 必须写成 x86 侧的隧道地址**（`192.168.20.1` / `192.168.30.1`），且 RelayNextHop 与出接口一致。

若把 next-hop 写成隧道**对端**地址（如 `192.168.20.2`），两条路径 next-hop 不同 → 不构成等价，`display ip routing-table <ip> verbose` 会显示成**两个独立的 Destination 块**（容易与 ECMP 的"同一 Destination 两行"混淆）。全局 `maximum load-balancing ebgp 2`。

### AR 侧

- 内部可达性由 AR↔ROS 的 OSPFv2 Area 0 提供，接口使用点对点网络类型，只重分发 `192.168.0.0/16` 直连路由
- 日本隧道公网端点保留双出口等价静态可达，避免隧道底座反向依赖隧道内路由
- export 策略 `DENY_ALL` 纯接收
- 默认路由 2 条静态 + `track nqa admin ct/cm` 做线路探测（15s 间隔、2 次失败撤线）

### ROS 侧

- 单 BGP instance(lo)，`output.network=CT/CM/blacklist`（直接引用 address-list 名）+ `network-blackhole=yes`
- `CT` / `CM` 表（约 3000 / 1500 条）与 `blacklist` 一起通告；两条日本通道都通告 blacklist → AR 收到同一网段的双 next-hop

## AR 目的端口 PBR（两条隧道各自锁线）

`GigabitEthernet0/0/1`（x86 的实际接入口）入向挂 `traffic-policy p-lb2`：

```text
acl 3999   permit udp source 192.168.1.2 0 destination <日本节点> 0 destination-port range <wg-ct 跳端口段>
acl 3998   permit udp source 192.168.1.2 0 destination <日本节点> 0 destination-port range <wg-cm 跳端口段>
traffic behavior b-tel-nh : redirect ip-nexthop 100.74.0.1   (电信)
traffic behavior b-mob-nh : redirect ip-nexthop 10.87.128.1  (移动)
traffic policy   p-lb2    : classifier c-tel → b-tel-nh ; classifier c-mob → b-mob-nh
interface GigabitEthernet0/0/1 : traffic-policy p-lb2 inbound
```

要点与限制：

- 该 **lan-board 只支持 `redirect ip-nexthop`**：写 `redirect interface Dialer1` 会提示 `not supported on lan-board`（配置能存下，但不执行）
- 策略只匹配「源 `192.168.1.2` → 目的日本节点」的 UDP，不触及其它内网流量，也不影响 wg-cn
- 两条隧道因此**各锁一条家宽**：`wg-ct` 恒走电信、`wg-cm` 恒走移动（端口仍在段内跳，线路不变）
- 代价：依赖 PPP 对端网关字面量，且**没有 backup**（见"已知弱点"）

## mangle PBR（blacklist 流量均分到两条隧道）

```text
prerouting, in-interface=<LAN>, dst-address-list=blacklist, connection-state=new
  + per-connection-classifier=both-addresses-and-ports:2/0 → mark-connection → 路由表 WG-CT
  + per-connection-classifier=both-addresses-and-ports:2/1 → mark-connection → 路由表 WG-CM
→ 两表默认路由分别走 wg-ct / wg-cm（各带 distance=2 的互备路由）
→ srcnat：出隧道前把源改写成 192.168.20.1 / 192.168.30.1
```

分流粒度按连接均分（所有者决定），不按业务类型细分。

## 隧道故障转移：递归公共 DNS 方案

!!! tip "原理"
    默认路由 next-hop 写公共 DNS IP（递归路由），main 表放锚点让网关递归解析到隧道对端；
    `check-gateway=ping` 每 10s 探测公共 DNS，端到端不通则主路由失效，`distance=2` 的 fallback 接管。

- main 表锚点：`1.1.1.1/32 → <wg-cm 对端>`、`8.8.8.8/32 → <wg-ct 对端>`，两条腿用不同探测目标区分
- 两条隧道互为主备（各自另有一条 `distance=2` 指向另一条隧道的路由）
- 递归路由三大坑（实测）：
    1. 自定义 routing-table 里 gateway 的递归解析查 **main 表**——锚点必须放 main 表
    2. 递归要求中间路由的 `scope` < 引用路由的 `target-scope`
    3. `check-gateway=ping` 两连败（约 20s）才撤路由，两连胜才恢复

## 已知弱点

- **PBR 无失败回退**：目的端口 PBR 是写死的 `redirect ip-nexthop`，某条家宽故障或运营商重拨导致网关变化时，对应端口段（等于整条隧道）会黑洞，没有 backup。候选改法是给 traffic behavior 加 `backup-nexthop` + NQA track，但需先验证该 lan-board 是否支持
- **BGP 首包收敛窗口**：DNS 联动即时写 list，但 AR 学到路由要等 BGP 传播，新域名首包可能短暂走错线
- blacklist 里的大范围条目（如 `co.jp`）有误伤风险，排障见[坑与排障手册](pitfalls.md)

## 相关文档

- 复盘:[从静态路由迁移到 OSPF 后，BGP ECMP 为什么只剩一条](../blog/posts/bgp-ecmp-after-ospf.md)
- 复盘:[两条 WG 隧道怎样各锁一条出口](../blog/posts/dual-wg-egress-pinning.md)
- 隧道参数:[隧道速查](tunnels.md)
- DNS 链路:[DNS 防污染](dns-architecture.md)
