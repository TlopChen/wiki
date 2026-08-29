# 家庭网络总览

!!! note "骨架页"
    本页是目录骨架，细节按需补充。

## 拓扑

- 出口/光猫 → **AR6140**（华为路由器）
- **ROS**（MikroTik RouterOS）× 2：192.168.1.1 / 192.168.1.2，SSH 端口统一 52222
- **OxiDNS**：DNS 服务，向 ROS 注入记录
- **腾讯云 VPS**（43.136.80.90，广州）：WireGuard 管理隧道对端
    - 隧道网段：VPS `192.168.40.1` ↔ 家里 ROS `192.168.40.2`
    - VPS 可直达家庭网段 192.168.1.0/24（仅管理用，不做转发/NAT）

## 分流体系

- 域名层：ROS `/ip dns static` FWD 条目，`match-subdomain=yes`，命中打标 `blacklist` 地址表
- IP 层：`/ip firewall address-list` 里的 CN / CT / CM / CU / CC / blacklist
- 数据源与更新：见 [ROS 规则管线](ros-rules-pipeline.md)

## 待补充

- [ ] VLAN 划分与接口表
- [ ] 防火墙规则清单与用途说明
- [ ] BGP 对等体设计（AR6140 ↔ ROS）
- [ ] 巡检基线与脚本清单
