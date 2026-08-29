# ROS 规则管线

家宽 ROS 的分流规则全自动更新链路，2026-08 建成。

## 整体架构

```
上游规则源(GitHub) → 腾讯云VPS镜像服务 → gen_rules.py 生成 .rsc → ROS 走 WireGuard 隧道拉取
```

## 组件

| 组件 | 位置 | 说明 |
|---|---|---|
| 镜像服务 | VPS `/srv/github-mirror/mirror.py` | 按需经 SSH 从 GitHub 取单个文件并缓存（6h），监听 :18080 |
| 生成器 | VPS `/srv/github-mirror/gen_rules.py` | 解析上游 → PSL 收敛注册域 → 产出 .rsc |
| 源配置 | VPS `/srv/github-mirror/sources.json` | 定义每张表的上游地址/列表名 |
| 手工域名 | VPS `/srv/github-mirror/manual-blacklist.txt` | 每行一个域名，`#` 注释，打 manual 标记合入 |
| 日更脚本 | VPS `/usr/local/bin/ros-rules-daily.sh` | cron 06:00：生成 → 同步进仓库 → 有变化才提交 |

## 产物与地址表命名

| 文件 | 地址表 | 内容 |
|---|---|---|
| `cn.rsc` | `CN` | 中国大陆 CIDR |
| `cn-telecom.rsc` | `CT` | 电信 |
| `cn-mobile.rsc` | `CM` | 移动 |
| `cn-unicom.rsc` | `CU` | 联通 |
| `cn-cernet.rsc` | `CC` | 教育网 |
| `blacklist.rsc` | `blacklist` | 被墙 IP（TG+Twitter+MikroTik AS51894） |
| `proxy-domain.rsc` | `blacklist`（DNS 静态） | 手工域名 + 代理侧域名，零正则，match-subdomain |

## ROS 端拉取

```
/tool fetch url="http://192.168.40.1:18080/ros/<文件名>" mode=http
/import file-name=<文件名>
```

- 走 WireGuard 隧道，不需要公网开端口
- 所有脚本幂等（整表重建），重复导入安全，导入顺序无关

## 常见操作

**加域名进黑名单**：编辑 VPS 的 `manual-blacklist.txt`，等 06:00 或手动跑
`python3 /srv/github-mirror/gen_rules.py`。

**加新 IP 规则源**：编辑 `sources.json` 增加条目（支持 CIDR 文本/Surge 规则行/
Clash payload YAML/裸域名四种格式，以及 `asn` 字段按自治系统从 RIPEstat 实时取段）。

**排查拉取失败**：SSH 到 VPS 看 `journalctl -u github-mirror` 里的
`[fetch-404]` / `[fetch-502]` 日志行。

## 经验教训

- ROS 正则能力极差，生成的配置**永远零正则**，正则/通配符行在上游解析时直接丢弃
- DNS 层不要放大体量广告表（ROS DNS 性能有限），域名级拦截用小规模精选表
- 镜像缓存失败可能留下 0 字节文件被 TTL 当合法缓存（已修复：空缓存视为未命中）
