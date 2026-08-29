# DNS 防污染

## 完整链路

```
PC → 家里 ROS(192.168.1.2,主 DNS)
      ├─ blacklist 域名(FWD 命中)→ forwarders "DNS"(192.168.20.2, 192.168.30.2)
      │     → sstp/wg 双隧道 → 日本 CHR → CHR forwarders "DNS"(1.1.1.1, 8.8.8.8)
      │     → 真实 IP → 自动写入 blacklist → 走隧道
      └─ 普通域名 → servers=192.168.1.1(AR)→ AR dns proxy → 运营商 DNS
```

## 关键点(勿改错)

- 家里 ROS 的 DNS 上游转发在 `/ip/dns/forwarders`,**不在 servers**:
  `forwarders name="DNS" dns-servers=192.168.20.2,192.168.30.2`
- FWD 条目的 `forward-to=DNS` 指向这个名为 `DNS` 的 forwarders 条目,不是全局 servers
- 验证记录(twitter.com):经 ROS 解析 = `172.66.0.227`(Cloudflare 真实);
  直连运营商 = `104.244.42.197`(污染)。防污染链路完好时勿动

## blacklist 联动设计

- DNS 解析出被墙域名的真实 IP 后,自动加入 `blacklist` 地址表
- BGP 把 blacklist 通告给 AR(回送 ROS)+ mangle PBR 分流,见[双线出口与分流](dual-wan-routing.md)
- blacklist 域名 FWD 表唯一来源:`proxy-domain.rsc`(手工 + 上游,整表重建),见[ROS 规则管线](ros-rules-pipeline.md)

## 相关文档

- 分流逻辑:[双线出口与分流](dual-wan-routing.md)
- 隧道参数:[隧道速查](tunnels.md)
