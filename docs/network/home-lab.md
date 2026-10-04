# 家庭网络总览

业务配置核验于 **2026-10-04 22:01–22:03，北京时间**。原始配置和认证仅存私有档案。近期变化见 [本轮总结](../blog/posts/operations-review-20261004.md)。

## 拓扑与出口

```text
电信 Dialer1 ─┐
              ├─ AR6140（192.168.1.1，AS64527）─ 家庭 LAN
移动 Dialer2 ─┘       ├─ XGE / 192.168.2.0/30 ─ 工作机 192.168.2.2
                      └─ ROS（192.168.1.2，AS64523）
                           ├─ wg-ct ─ 日本，电信承载
                           ├─ wg-cm ─ 日本，移动承载
                           ├─ wg-cn ─ 广州，回家与规则分发
                           └─ OxiDNS（192.168.1.3）
```

XGE 是工作机直连段，不能写成第二条 AR–ROS 物理互联。OSPF 当前 Full，AR 四条 BGP Established。

- AR 国内明细来自 DIRECT_IP_NOCM / DIRECT_IP_NOCT，blacklist 仍来自两条日本 BGP 通道。
- AR 优先默认路由指向两个日本隧道对端，preference 50，递归下一跳均为 ROS；旧运营商缺省 preference 60，本轮 Inactive。
- ROS blacklist PCC 保留，并有非 DIRECT_IP、非内部网段的 LAN 新连接 PCC 承接默认流量。ROS main 缺省仍回 AR。
- WG 两表主腿与 distance=2 互备路由只 ping 隧道对端，不能证明日本公网健康；AR 优先缺省也不能靠 OSPF 证明公网可用。
- 日本两条外层 UDP 由 AR 目的端口 PBR 锁线。ROS 每分钟改三条目的入口端口，源监听端口不自动轮换。

见 [双线出口与分流](dual-wan-routing.md) 与 [隧道速查](tunnels.md)。

## DNS 与规则

- DHCP 首选 OxiDNS，备用 AR；AR 仍保留运营商 DNS 回退路径。本工作机现只配置 OxiDNS，不能外推全网已取消备用。
- OxiDNS：代理 → JP；非代理且非国内 → JP；非代理且国内 → CN。条件互斥，无应答 SERVFAIL。纯内存正缓存，负缓存关闭。
- blacklist 动态注入由 OxiDNS 插件控制，fixed_ttl=86400 秒；ROS allow-remote-requests=no，自身 DNS 为 OxiDNS + AR，address-list-extra-time=0s。
- 广州生成发布；ROS 06:30 唯一下载；OxiDNS 07:00 本地 reload。广州每分钟检测手工清单，设备立即生效仍需按需执行 ROS manual-refresh。

见 [DNS 分流架构](dns-architecture.md)、[规则管线](ros-rules-pipeline.md)。

## 数量与健康基线

| 项目 | 本轮数量 | 说明 |
|---|---:|---|
| CT / CM | 3080 / 1493 | 仍同步，已不是国内 BGP 宣告源 |
| DIRECT_IP | 6326 | 国内与手工直连 IPv4 合并表 |
| DIRECT_IP_NOCM / NOCT | 7646 / 6909 | 国内宣告源，与 AR 收到数量一致 |
| 日本两条 BGP | 各 2455 | blacklist 动态变化，属采样基线 |

三隧道握手在两分钟内；ROS 到日本电信 / 移动 / 广州各三次 ping 无丢包，均值约 69 / 78 / 22ms。OxiDNS health / reload 正常，国内外 A 实查成功；广州镜像健康，日本 dnsmasq 与 WG active，两台 Linux 无 failed unit。

待收口：AR DHCP 备用回退风险；JP 移动腿竞速计数偏斜与累计 DNS 超时；PBR 固定运营商网关依赖；默认路由缺少端到端健康保证。本轮设备查询只读，未做断线测试。
