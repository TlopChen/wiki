---
date: 2026-09-22
categories:
  - 网络复盘
tags:
  - WireGuard
  - PBR
  - BGP
  - ECMP
  - AR6140
  - RouterOS
---

# 两条 WG 隧道怎样各锁一条出口

## 结论

想让"两条隧道分别固定走电信 / 移动"，不能指望 AR 的 ECMP 去分——实测端口到线路的映射并不可预测。真正解决的是 **AR6140 在 x86 接入端口上按目的端口段做策略路由**：两条隧道各用一段独立的动态跳端口，AR 把两段分别 `redirect ip-nexthop` 到电信 / 移动的 BRAS 网关。

一个必须记住的限制：这块 lan-board **只认 `redirect ip-nexthop`**，`redirect interface Dialer1` 会被拒绝（配置能存下，但不执行）。

<!-- more -->

## 背景

家宽是双 PPPoE（电信 `Dialer1` + 移动 `Dialer2`），日本出口原本是一条 WG 隧道，出口由 AR 的双出口 ECMP 决定。两条链路质量差异很大（同一路径晚高峰 144 Mbps ↔ 凌晨 683 Mbps），因此希望：

- 有两条独立隧道，能分别承载不同业务；
- 并且**每条隧道固定在一条家宽上**，可控、可解释。

## 为什么"靠 ECMP 分"走不通

先把已有事实摆出来：

- 静态 ECMP 的哈希只按源 / 目的 IP（`load-balance src-dst-ip`），而隧道流量的这两个值都是固定的；
- 直接换端口试出口：同一时刻、同一条隧道，1500 → 电信、4096 → 移动、8192 → 移动，**没有确定的"端口段 ↔ 线路"对应**；
- 结论：ECMP 的哈希结果是实现细节，不能作为"锁线"的依据。

## 探索路径

| 尝试 | 结果 |
|---|---|
| MQC 策略挂在 `Vlanif1`（内网网关口） | 命令接受、`display this` 可见，但**不生效**（改端口段后出口不变） |
| MQC 挂在 `GE0/0/1`（x86 实际接入口），动作用 `redirect interface Dialer1` | 报 `The rule or action is not supported on lan-board`——**配置存下但不执行** |
| 同一位置改用 `redirect ip-nexthop <电信 BRAS>` | ✅ 无告警，实测生效 |
| 在 `XGigabitEthernet`（WAN 板卡）上试 `redirect interface` | 生效，但需要把 x86 挪到 XGE 口、多插一根线 |

既然 `redirect ip-nexthop` 在现有接线下就能用，就不动线了。

## 三侧定版

**1. 日本节点（Debian）**：两条独立接口，各自监听端口

```text
wg-ct : 192.168.20.2/30，监听 6813    （电信线）
wg-cm : 192.168.30.2/30，监听 6812    （移动线，原 wg0/wg-home）
nft ip nat prerouting：
  udp dport <wg-ct 跳端口段> redirect to :6813
  udp dport <wg-cm 跳端口段> redirect to :6812
dnsmasq：同时 interface=wg-ct 与 interface=wg-cm（两条隧道各一个上游地址）
```

**2. 家里 ROS**：两条隧道 + 两段动态端口 + 按连接分流

```text
/interface wireguard peers set [find interface=wg-ct] endpoint-port=$pT   # 段 A
/interface wireguard peers set [find interface=wg-cm] endpoint-port=$pM   # 段 B
mangle：blacklist 新连接 both-addresses-and-ports:2/0 → 表 WG-CT、2/1 → 表 WG-CM
srcnat：出 wg-ct 伪装 192.168.20.1、出 wg-cm 伪装 192.168.30.1
```

脚本每分钟改写 `endpoint-port`，段内随机化（抗封锁特性保留），但**段到线路的映射是静态的**，所以出口不会跟着漂。

**3. AR6140**：按目的端口段重定向

```text
acl 3999 / 3998：源 192.168.1.2 → 目的日本节点，分别匹配两段动态端口
traffic behavior：redirect ip-nexthop <电信 BRAS> / <移动 BRAS>
traffic policy    → 挂 GigabitEthernet0/0/1 inbound
```

顺带把 BGP 的两条通道也理顺，让"日本线路"能同时挂在两条隧道上：

```text
route-policy CM-WG_IMPORT : apply ip-address next-hop 192.168.30.1   # 走 wg-cm
route-policy WG-CT_IMPORT : apply ip-address next-hop 192.168.20.1   # 走 wg-ct
```

next-hop 必须写成 **x86 侧的隧道地址**，且两条路径的 RelayNextHop 与出接口一致，才会合并成等价路由。VRP 不支持 route-policy 改名，实际步骤是"建新 → 改 peer 引用 → 删旧 → `refresh bgp all import` 软复位"。

## 验证

- **出口归属**：在节点侧多次采样两条隧道的握手来源——`wg-ct` 恒为电信出口、`wg-cm` 恒为移动出口，且**端口在段内持续变化、线路不变**，跨多轮端口跳跃采样结论一致
- **策略命中**：两条 ACL 的匹配计数持续增长
- **延迟**：两条隧道对端 `ping` 均 0% 丢包（电信侧约 65 ms、移动侧约 78 ms）
- **吞吐**：双隧道并行下载时两条各承担约一半流量，符合按连接均分的预期

## 回滚

1. `undo` 掉 `GE0/0/1` 上的 `traffic-policy`——策略立即失效，两条隧道回到 ECMP 抽签状态；
2. 若要完全回到单隧道：合并两段跳端口、删除其中一条 WG peer 及其 mangle / srcnat；
3. BGP 侧回滚：把 peer 的 import policy 指回旧策略（或把 next-hop 改回原值）后软复位。

## 可复用经验

1. **"锁线"要自己做，别赌 ECMP**：哈希行为依赖实现细节，先做"端口 → 线路"的实测再定方案；
2. **同机不同板卡能力不同**：WAN 板卡支持 `redirect interface`，lan-board 不支持，只能退到 `redirect ip-nexthop`；
3. **跳端口与锁线并不冲突**：端口仍可每分钟随机化，锁线靠"端口段 → 线路"的静态映射；
4. **ECMP 等价看 next-hop**：要写成对端设备的本地地址，写成隧道对端地址会变成两条独立路由；
5. **"配置能存下"不等于生效**：`not supported on lan-board` 这类提示必须当真，改完用实际流量验证。
