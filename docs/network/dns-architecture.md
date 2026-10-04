# DNS 分流架构

当前配置核验于 **2026-10-04 22:01–22:03，北京时间**。OxiDNS v1.6.0 运行在 ROS 容器中；主链采用互斥三段分流，缓存为纯内存，关闭负缓存。当前配置版本前缀为 `0c0ee0a4`。

## 入口与完整链路

AR DHCP 当前下发 **192.168.1.3（OxiDNS）+ 192.168.1.1（AR 备用）**。ROS 自身 DNS 也配置这两个地址，但 `allow-remote-requests=no`，不为客户端提供递归解析。

```text
客户端 → OxiDNS（直接使用 192.168.1.3）
       或 AR DNS 代理（首选转发到 OxiDNS）
       或经过 ROS 的 LAN 53 端口 dstnat → OxiDNS
            ↓
AAAA → NODATA；缓存命中 → 直接应答
未命中缓存 → 互斥三段分流
  ① 命中代理源                         → JP
  ② 未命中代理源，且不在国内全量表      → JP
  ③ 未命中代理源，且在国内全量表        → CN
无上游应答 → SERVFAIL（reject 2）
JP 分支结果 → ros_address_list → ROS blacklist → BGP / PCC
```

“未命中”指**域名规则未匹配**，不是上游没有应答。JP 失败不会改走 CN，也不会再执行第二次 JP。代理源优先于国内表；国内表来自本地文本与 geosite 的 `cn` / `geolocation-cn` 集合。

| 地址 | 当前角色 | 边界 |
|---|---|---|
| 192.168.1.3 | OxiDNS，应答与分流 | DHCP 首选 |
| 192.168.1.1 | AR DNS 代理与备用入口 | 仍保留 IPCP 学来的运营商 DNS，存在回退路径 |
| 192.168.1.2 | ROS，LAN 53 端口转发入口 | 不提供远程递归解析；并非当前 DHCP 首选 |

ROS 的 DNS 白名单当前仅 OxiDNS 地址启用，AR / ROS 两项为 disabled。OxiDNS 必须被排除，否则它访问上游时可能被转回自身。dstnat 只覆盖经过相应接口的流量，不能据此声称所有 AR 直连客户端的硬编码 DNS 都被接管。

## 上游、缓存与地址表

- JP：日本隧道对端 `192.168.30.2` / `192.168.20.2`，`concurrent: 2`、`response_selection: fastest`。
- CN：电信、移动、阿里、腾讯共六台上游并发竞速。
- OxiDNS：size=65536、max_positive_ttl=86400、lazy_cache_ttl=600、cache_negative=false，无 dump_file。重启后缓存清空。
- 应答 TTL 归一为 60–86400 秒；ros_address_list 当前 fixed_ttl=86400，异步写 blacklist。生命周期由插件控制，不能套用 ROS 原生 DNS 的 address-list-extra-time。
- AR 与日本 dnsmasq 仍各有缓存；日本 cache-size=10000。历史“全网只剩一层缓存”表述不准确。

上游没有应答时用 **SERVFAIL**，不能合成 NXDOMAIN，把故障伪装成域名不存在。`black_hole servfail` 曾通过 validate 但在 reload 初始化失败，实际可用的是 `reject 2`；每次变更必须核对 reload 成功、last_error=null、running 与 target 版本一致。

## 规则更新

ROS 是唯一下载方：每日 06:30 同步六张地址表与 OxiDNS 三份规则文件。OxiDNS 每日 07:00 只做本地 provider reload；rules_download 插件定义也已删除。断网仍可启动和重载已有规则，文件新鲜度由 ROS 同步负责。

按需运行 ROS manual-refresh 会同步后重启容器，历史实测有约 1–8 秒解析中断。缓存命中仍短路，本轮未重新设计缓存命中与 blacklist 续期。

## 尚未解决的边界

1. **AR 备用 DNS 回退路径仍存在。** DHCP 仍下发 AR，AR 仍保存运营商解析器。OxiDNS 不跨组回退不等于客户端、AR 也不回退。flushdns 后暂时恢复不能证明配置问题消失。本轮未故障注入，不能把此风险路径写成每次历史泄露都已证实的唯一根因。
2. **双腿配置不等于双腿已验证可靠。** 本轮 JP 上游 query 均为 9195，移动 success=30、电信=9158；两条隧道 ping 均正常。计数偏斜需结合竞速取消语义与两端 DNS 抓包判断，不能仅凭计数认定移动腿丢包。
3. **累计指标仍有错误。** CN error=182 / timeout=27，JP error=5 / timeout=2，都是采样累计值，不是瞬时故障率。当前健康检查通过，百度与 Google 实际解析成功；尚未证明长期无超时。

## 沿革

- 09-16：OxiDNS 接管 DNS，smartdns 下线。
- 10-01：拉取收口到 ROS，OxiDNS 只读本地规则。
- 10-03：移动腿更换源端口恢复；删除下载插件残留；纯内存缓存并关闭负缓存；恢复 JP 双腿竞速。
- 10-04：三段条件互斥，取消按 !has_resp 跨组回落，无应答改 SERVFAIL。

见 [坑与排障手册](pitfalls.md) 与 [双线出口与分流](dual-wan-routing.md)。
