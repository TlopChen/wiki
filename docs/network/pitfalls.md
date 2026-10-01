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
- 两条 eBGP 要合并成等价路由，**AR import policy 的 next-hop 必须写成 x86 侧的隧道地址**（`192.168.20.1` / `192.168.30.1`）：写成隧道对端地址会让两条路径 next-hop 不同，`display ip routing-table <ip> verbose` 显示成两个独立 Destination 块（容易误判成 ECMP）
- SSTP/PPP 应按 peer `/32` 理解：本地接口地址不会像以太网网段一样自然传播，跨设备下一跳优先使用实际的 PPP 对端地址
- 从静态路由迁移到 OSPF 时，删除静态前先搜索 BGP route-policy、策略路由、探测和脚本对旧下一跳的引用
- BGP 会话正常而 ECMP 消失时，重启只能重新得到相同选路；先对比候选路径属性与递归路由
- 实例见[从静态路由迁移到 OSPF 后，BGP ECMP 为什么只剩一条](../blog/posts/bgp-ecmp-after-ospf.md)

## 双隧道与出口锁定（2026-09-22）

- **lan-board 不支持 `redirect interface`**：在 x86 接入口（`GE0/0/1`）上，`traffic behavior` 里写 `redirect interface Dialer1` 会提示 `The rule or action is not supported on lan-board`——配置能存下但**不执行**；同一位置 `redirect ip-nexthop <网关>` 无告警且实测生效
- **所以 x86 不必挪到 XGE 口**：按目的端口段 `redirect ip-nexthop` 就够；换板卡只在必须要 `redirect interface` 时才有意义
- **策略依赖 PPP 对端网关字面量**：`redirect ip-nexthop` 指向 BRAS 网关，运营商重拨后网关变化即失效（且没有 backup）——变更前先 `ping` 确认网关可达
- **RouterOS 写脚本必须转义 `$`**：`/system script set <name> source="…"` 会把 `$var` 当成终端变量**当场展开成空**，存进去的脚本残缺（运行报 `syntax error`，现象是"隧道还在但端口不再变化"）。正确做法是把 `$` 全部写成 `\$`
- **`wg syncconf` 不会清零未出现的参数**：删掉 `.conf` 里的 `PersistentKeepalive` 行再 `wg syncconf`，运行态仍保留旧值（syncconf 只做增量同步）。要真正改值必须 `wg set <if> peer <pubkey> persistent-keepalive <n>`，`.conf` 只保证重启后生效
- **WireGuard 缺省没有心跳**：`persistent-keepalive` 默认是 `off`（不是"60 秒"）；协议自带的 `KEEPALIVE_TIMEOUT=10s` 被动保活只在有流量时生效，完全空闲的隧道会显示未握手，下次发包自动重建

## 网段与地址选择

- **192.168.19.1 是运营商 CGNAT 网里的真实设备**(ping 返回 21ms / TTL 250,华为设备)——隧道 lo 地址、新网段要避开 `100.64/10`、`10.x`、`192.168.19.x` 等 CGNAT 占用段
- `192.168.2.0/30` 不是废弃网段,是 **PC 10G 口直连段**(XGE0/0/0);接口 down 只是未插线

## 云平台差异

- **腾讯云轻量有平台 PAT**:外部 UDP 入站的源端口会被改写。能固定的只有**服务端监听口**(广州 WG 用 51820);客户端入口端口仍然每分钟跳,广州侧用一段端口 `redirect` 到 51820 承接——别误读成"客户端不跳端口"
- **nftables policy drop 会丢 ICMP echo-request**:导致"VPS ping 通 ROS、ROS ping 不通 VPS"的单向假象,需 input 放行 `icmp type echo-request`
- VPS 侧 wg peer 必须把家庭 LAN 段加进 AllowedIPs **并**加回程路由,否则 LAN 回程不通

## 边界结论(勿再尝试)

- **BFD**:ROS 7.24 BFD 不支持 ip route gateways(`Features not yet supported`),会话只能由 BGP/OSPF 触发
- **"双 WG 双线"本身可行,但别指望 ECMP 去分**:单流永远单线(ECMP 按流 hash + NAT 会话绑定),实测端口→线路映射不可预测。2026-09-22 的做法是**两条独立 WG 接口 + AR 侧按目的端口段 PBR 锁线**,见[复盘](../blog/posts/dual-wg-egress-pinning.md)
- **ROS 本机流量不过隧道**:更新检查失败属正常
- **DNS 层广告表**:不放大体量表(ROS DNS 性能有限),域名拦截在小规模精选表上做

## 相关文档

- [双线出口与分流](dual-wan-routing.md)
- [隧道速查](tunnels.md)
- [DNS 防污染](dns-architecture.md)


## DNS 分流（OxiDNS）相关

- **运营商 DNS 禁 ICMP**：`ping 202.103.224.68` 不通 **≠** 不可用（实测 UDP 53 正常返回）。判断上游可用性要看业务层（发实际查询），别用 ping
- **dstnat 回环**：把 :53 劫持指向某个容器后，该容器自身的上游查询也会命中同一条规则。必须把它的地址加进 `address-list=DNS` 白名单，否则查询自己打自己
- **地址列表写入"看起来没生效"**：OxiDNS 写 `blacklist` 时，若目标 IP 已存在（ROS 联动注入过），它**既不计成功也不计错误**（两个指标都是 0）。别据此判断插件故障——用不在表里的新域名验证
- **容器直连 GitHub 不通**：OxiDNS 拉 `geosite.dat` 这类 GitHub releases 资源会失败；应由 VPS 代拉后放内网镜像，容器再从镜像拉
- **镜像服务改白名单必须重启**：`repos.json` 加了新仓库后要 `systemctl restart github-mirror`，否则新源一律 403
- **全局指标不能判断单个域名走向**：`forward_query_total` 是累计值，会被其它设备的查询干扰；要看单个域名命中哪条规则，得开 `query_recorder` 或看 provider 匹配
- **日更静默失败**：生成器拉不到上游会 abort，但没人看日志就发现不了（曾连续 3 天产物没更新而 ROS 一直拉旧文件）。现在生成器改为"拉取即留档、失败回退上一版"，且可用产物 mtime 快速判断
- **AR 的 DNS 缓存关不掉**：命令空间只有 `dns application cache ttl maximum`（默认 86400，下限 3600 不可配），没有 enable/disable。要让缓存彻底单层化，只能让 AR 退出客户端解析（`undo dns proxy enable` + 不再下发 1.1），代价是丢掉运营商 DNS 兜底
- **静态 DNS 优先于动态**：AR 上静态配置的 `dns server` 拿 `Priority 0`，PPPoE 下发的运营商 DNS 是 `Priority 1–4` 兜底；用 `display dns server verbose` 核对
