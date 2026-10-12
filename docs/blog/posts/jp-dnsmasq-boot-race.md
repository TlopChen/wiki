---
date: 2026-10-12
categories:
  - 网络复盘
tags:
  - DNS
  - dnsmasq
  - systemd
  - nftables
  - WireGuard
---

# 日本节点 dnsmasq 冷启动竞态：境外解析"全断"的 6.5 小时

## 结论

境外域名解析全断的根因既不是分流规则、也不是隧道，而是日本节点上的 dnsmasq **在系统冷启动时被 systemd 的顺序按死**：配置里写死了 `interface=wg-ct` / `interface=wg-cm` 又开着 `bind-interfaces`，而开机时 `wg-quick@` 必然排在它后面 —— 接口还不存在，dnsmasq 直接以 `unknown interface` 退出。该 unit 又是 `Restart=no`，失败即终态，一直躺到人工发现。

同一天还叠加了另一件事：一条家宽的外部光缆故障，两条日本隧道暂时共用另一条出口。两件事互不相干，但都很容易被读成"策略路由（PBR）出问题了"。

<!-- more -->

## 现象

- 客户端境外域名解析失败、国内正常；OxiDNS 的 JP 组指标是「查询数在涨、成功率恒为 0」。
- 隧道本身完全正常：两条腿握手新鲜、ping 0% 丢包、其它业务可用。
- 日本节点上 `dnsmasq` 状态是 `failed`，53 端口无人监听。

## 时间线（日本节点，单调时间 / JST）

| 时刻 | 事件 |
|---|---|
| 05:17:01 | 上一轮开机记录的最后一条（宿主机侧约 1 小时 40 分钟不可达，原因未知） |
| 06:57:21 | 系统重新引导 |
| 06:57:22 | `nftables.service` 加载完成（防火墙在位） |
| 06:57:24 | `dnsmasq` 启动 → `unknown interface wg-ct` → `FAILED to start up`（exit 2） |
| 06:57:24 | `wg-quick@wg-cm` / `wg-quick@wg-ct` 才开始 active |
| 06:57:24 | `ip link add wg-cm` / `ip link add wg-ct`（接口这时才存在） |
| 13:25:30 | 人工 `systemctl start dnsmasq`，服务恢复 |

dnsmasq 只比接口创建早约 **100 毫秒**，但足以致命。

## 为什么是"必然"，不是"运气不好"

三个条件叠加：

1. **服务配置写死了接口名**：`interface=wg-ct` / `interface=wg-cm` + `bind-interfaces`。`bind-interfaces` 在启动时解析接口列表，名字解析不到就**致命退出**（实测 exit 2）；而 `ExecStartPre` 的 `checkconfig` 只做语法检查，仍然返回 0，拦不住。
2. **systemd 顺序注定撞车**：`dnsmasq.service` 是 `After=network-online.target` + `Before=nss-lookup.target`；`wg-quick@.service` 是 `After=network-online.target nss-lookup.target`。同一次初始事务里，wg 接口**必然**晚于 dnsmasq。
3. **没有自愈**：unit 是 `Restart=no`，失败即终态；接口 100 毫秒后建好也不会触发重试。

## 为什么以前一直没事

这份"接口绑定式"配置是 9-22 定稿的，当天那几次重启都发生在接口已经存在的时候（重载、改配置），全部成功；此后这台 VPS 连续跑了三周没重启过 —— **冷启动路径直到今天才第一次被走到**。

## 改法：服务只管监听，ACL 交给防火墙

```diff
- interface=wg-cm
- interface=wg-ct
- bind-interfaces
+ except-interface=eth0        # 通配监听（本机所有地址，含环回），排除公网口
```

- **服务侧**：配置里不再出现任何 wg 接口名，`ss` 显示 `0.0.0.0:53` + `[::]:53`，冷启动竞态从根上消失；环回口也被通配覆盖，节点自身的 `/etc/resolv.conf`（指向 127.0.0.1）照旧可用。
- **访问控制**：交给已有的 nftables 入站 ACL —— `policy drop` + 白名单（管理口、WireGuard 监听段、两条 wg 接口、必要 ICMP/ICMPv6），规则落在 `/etc/nftables.conf` 且 `nftables.service` enabled，开机时比 DNS 服务先加载。
- **另两层兜底**：`except-interface=eth0` 与 systemd 包装脚本自带的 `--local-service`。

顺带纠正一个容易想当然的点：**环回口不等于"本机所有 IP"**。绑 `127.0.0.1` 的 socket 只收目的地址是 `127.0.0.1` 的包，隧道地址上的查询收不到；"本机所有 IP"对应的是通配监听。实测（同一台机器两个临时实例）：只绑环回时打 `127.0.0.1` 有应答、打隧道地址超时；通配实例对环回与两个隧道地址全部应答。

## 验证

- **服务**：`0.0.0.0:53` + `[::]:53`、`is-active=active`，本机 `getent hosts` 正常。
- **业务**：从内网 ROS 分别打两条腿的上游地址，解析均成功（返回真实 A 记录）；OxiDNS JP 组 30 秒增量「查询 +1 / 成功 +1」，之前的 error / timeout 计数冻结不再增长；国内解析不受影响。
- **外部暴露**：打 VPS 公网地址的 53 端口无任何真实应答（链路中间注入的假包不算）。

## 可复用经验

1. **服务配置里不写接口名**：监听 `0.0.0.0`，谁能访问交给防火墙 ACL；
2. **unit 之间要显式表达依赖**，或者干脆让服务对"接口还没出现"免疫；
3. **`Restart=no` 的失败是终态**：长跑服务要么自愈、要么有告警 —— 这次是靠人发现的；
4. **测试路径要确认打到了谁**：从局域网打任何地址的 53 都可能被家里的 dstnat 转走，用 AAAA 判别或看 dstnat 计数；
5. **指标要分层看**：隧道层（握手、丢包）完全正常，业务层（DNS 成功率）其实是 0 —— "链路通"不等于"服务在"。

## 相关

- [隧道速查](../../network/tunnels.md)
- [DNS 防污染](../../network/dns-architecture.md)
- [坑与排障手册](../../network/pitfalls.md)
