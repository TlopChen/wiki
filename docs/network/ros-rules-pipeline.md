# 规则管线（Git 分发）

家宽分流规则的全自动更新链路。2026-08 建成，2026-09-16 重构为**分层生成 + 双消费端**。

## 整体架构

```
上游源（域名源 + IP 段源）
   ↓ VPS 06:00 日更：拉取即留档（static/src/；拉不到时回退上一版素材 = 兜底）
   ├─ gen_rules.py  → proxy-domain.rsc / .domains.txt / .exact.txt / cn*.rsc / blacklist.rsc
   ├─ gen_oxi.py    → proxy-domain.oxi.txt（OxiDNS domain_set 格式）
   ├─ gen_cn.py     → cn-domains.oxi.txt（国内域名表）
   └─ fetch_bin.py  → geosite.dat（二进制素材）
   ↓ git push（TlopChen/git，路径 ros/）
   ├─ ROS     06:30  sync.rsc → 内网镜像 18080 → /import
   └─ OxiDNS  07:00  download → reload_provider（三份表热重载）
```

设计要点：**生成与分发解耦、格式适配分层**。新增消费端只需写一个适配器（读中间产物）+ 一条拉取链路。

## 组件

| 组件 | 位置 | 说明 |
|---|---|---|
| 镜像服务 | VPS `/srv/github-mirror/mirror.py` | 按需镜像 GitHub raw（白名单 `repos.json`，TTL 6h），**只监听隧道地址 192.168.40.1:18080** |
| 生成器 | 同目录 `gen_rules.py` / `gen_oxi.py` / `gen_cn.py` / `fetch_bin.py` | 见上 |
| 源配置 | `sources.json` | 每张表的上游地址、列表名、手工修正文件 |
| 手工清单 | `manual-blacklist.txt` / `exclude-blacklist.txt` | **以 Git 仓库 `ros/generator/` 那份为准** |
| 日更脚本 | `/usr/local/bin/ros-rules-daily.sh` | cron 06:00，日志 `/var/log/ros-rules-daily.log` |
| 源素材 | `static/src/` → 仓库 `ros/src/` | 每份上游原始文件留档，可追溯、可离线复现 |

## 产物与消费端

| 文件 | 消费端 |
|---|---|
| `proxy-domain.rsc`、`cn*.rsc`、`blacklist.rsc`、`sync.rsc` | RouterOS（`/import`） |
| `proxy-domain.oxi.txt`、`cn-domains.oxi.txt`、`geosite.dat` | OxiDNS |
| `proxy-domain.domains.txt`、`proxy-domain.exact.txt` | 通用中间产物（其它 DNS 可自行转换） |

## 两条重要语义（勿破坏）

- **后缀域 vs 精确域**：`DOMAIN-SUFFIX,x` / 裸域名 → PSL 收敛成注册域（`domain:x`、`match-subdomain=yes`）；`DOMAIN,x` 保持完全限定（`full:x`、`match-subdomain=no`）。
  **精确域绝不能被放大成整域**——曾把一个精确 FQDN 概括成整域，导致该域下的国内子域被错误代理。
- **IP 段类规则**：Telegram / Twitter 这类客户端把服务器 IP 硬编码、不走 DNS 的服务，域名规则无效，必须靠 `blacklist` 的 CIDR 段兜底。
  判定方法：**看客户端是否直连 IP**。

## 增删规则

| 需求 | 操作 |
|---|---|
| 某域名强制走代理 | 改仓库 `ros/generator/manual-blacklist.txt` |
| 某域名改走直连 | 改仓库 `ros/generator/exclude-blacklist.txt` |
| 增加/更换域名源 | 改 `sources.json` 的 `sources` |
| 增删被墙服务的 IP 段 | 改 `sources.json` 里 `blacklist` 的 `sources` / `extra_cidrs` |
| 新增消费端格式 | 写适配器读 `proxy-domain.domains.txt`，产物加进日更脚本 |

改完等次日 06:00 日更自动生效，或手动 `python3 /srv/github-mirror/gen_rules.py` 立即重生成。

## 验证

- 产物是否当天：`ls -la /srv/github-mirror/static/ros/`（看 mtime）；日更日志 `tail -20 /var/log/ros-rules-daily.log`
- ROS 侧：`/ip dns static print count-only where type=FWD`、`/ip firewall address-list print count-only where list=blacklist`
- OxiDNS 侧：`curl -s http://192.168.1.3:9199/api/logs | grep "snapshot built"` 看各 provider 的规则数
