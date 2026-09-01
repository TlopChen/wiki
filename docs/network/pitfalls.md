# 坑与排障手册

把踩过的坑集中放这儿,排障时先对照查一遍。

## 通用排查路径

- **"解析正常但连不上/黑洞"**:先查 `blacklist` 联动是否误伤(曾经 MikroTik 官网被黑洞误伤)
- **ROS 本机更新失败 / Check For Updates 报 Host unreachable**:架构限制,ROS 本机流量不走隧道形成环路,不是故障;版本最新时忽略,更新走手动 npk(VPS 下载 → 隧道 → ROS)
- **remote 排障**:AR SSH exec 模式断连缺陷仍在,用交互式登录(paramiko 交互方案),勿再查配置

## 路由与 rp-filter

- 家里 ROS `rp-filter=loose`(strict 会丢非对称回程,**勿改回**)
- 回程需补 `192.168.2.0/30` 路由(gateway=192.168.1.1),否则 PC 10G 段不通
- 自定义 routing-table 的 gateway 递归解析查 **main 表**,锚点必须放 main 表

## BGP ECMP 与下一跳递归

- BGP 候选路径属性满足多路径条件仍不够，**下一跳递归后的 IGP cost 也必须相等**
- SSTP/PPP 应按 peer `/32` 理解：本地接口地址不会像以太网网段一样自然传播，跨设备下一跳优先使用实际的 PPP 对端地址
- 从静态路由迁移到 OSPF 时，删除静态前先搜索 BGP route-policy、策略路由、探测和脚本对旧下一跳的引用
- BGP 会话正常而 ECMP 消失时，重启只能重新得到相同选路；先对比候选路径属性与递归路由
- 实例见[从静态路由迁移到 OSPF 后，BGP ECMP 为什么只剩一条](../blog/posts/bgp-ecmp-after-ospf.md)

## 网段与地址选择

- **192.168.19.1 是运营商 CGNAT 网里的真实设备**(ping 返回 21ms / TTL 250,华为设备)——隧道 lo 地址、新网段要避开 `100.64/10`、`10.x`、`192.168.19.x` 等 CGNAT 占用段
- `192.168.2.0/30` 不是废弃网段,是 **PC 10G 口直连段**(XGE0/0/0);接口 down 只是未插线

## 云平台差异

- **腾讯云轻量有平台 PAT**:外部 UDP 入站被改源端口,跳端口 redirect 方案失效,只能固定端口(WG 用 51820)
- **nftables policy drop 会丢 ICMP echo-request**:导致"VPS ping 通 ROS、ROS ping 不通 VPS"的单向假象,需 input 放行 `icmp type echo-request`
- VPS 侧 wg peer 必须把家庭 LAN 段加进 AllowedIPs **并**加回程路由,否则 LAN 回程不通

## 边界结论(勿再尝试)

- **BFD**:ROS 7.24 BFD 不支持 ip route gateways(`Features not yet supported`),会话只能由 BGP/OSPF 触发
- **双 WG 双线**:单流永远单线(ECMP 按流 hash + NAT 会话绑定),已废案
- **ROS 本机流量不过隧道**:更新检查失败属正常
- **DNS 层广告表**:不放大体量表(ROS DNS 性能有限),域名拦截在小规模精选表上做

## 相关文档

- [双线出口与分流](dual-wan-routing.md)
- [隧道速查](tunnels.md)
- [DNS 防污染](dns-architecture.md)
