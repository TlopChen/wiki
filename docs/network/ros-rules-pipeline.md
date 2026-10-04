# 规则管线（Git 分发）

家宽分流规则的全自动更新链路。2026-08 建成，2026-09-16 重构为**分层生成 + 双消费端**；
2026-10-01 把仓库整理成**脚本 / 输入 / 输出**三层，代码权威回到 Git。

## 整体架构

```text
上游源（域名源 + IP 段源）
   ↓ VPS 06:00 日更（拉取即留档 input/upstream/<owner>__<repo>/；拉不到回退上一版素材）
   ├─ gen_rules.py  → proxy-domain.rsc / .domains.txt / .exact.txt / cn*.rsc / blacklist.rsc
   ├─ gen_oxi.py    → proxy-domain.oxi.txt（OxiDNS domain_set 格式）
   ├─ gen_cn.py     → cn-domains.oxi.txt（国内域名表）
   └─ fetch_bin.py  → geosite.dat（二进制素材）
   ↓ scripts/_infra/publish.sh → 内网 HTTP 根 /ros/<文件>
   └─ ROS  唯一拉取方（06:30 定时 / 按需 manual-refresh）
        ├─ 六张地址表       → /import
        └─ OxiDNS 三份规则  → 本地文件（OxiDNS 自己不再联网）
```

设计要点：**生成与分发解耦、格式适配分层**。仓库主体是脚本，ROS / OxiDNS 只是产物的消费端；
拉取与导入由 ROS 自带的脚本能力完成，不塞进 Git。

## 仓库结构

```text
README.md
scripts/
├── domain-rules/            域名 + CIDR 规则管线
│   ├── sources.json         ← 来源清单：加减源只改这一处
│   ├── store.py             ← 取数 / 快照 / 记出处的唯一入口
│   ├── gen_rules.py  gen_cn.py  gen_oxi.py  fetch_bin.py
│   ├── input/upstream/      上游原始快照（每个仓库一个目录，带 _provenance.json）
│   ├── input/manual/        手工清单（Git 权威，网页可直接改）
│   └── output/              产物
├── direct-ip/               直连 IPv4 管线（同结构）
├── _infra/                  服务端分发与调度：mirror.py / publish.sh / 日更入口
└── ros-side/                ROS 本地脚本留档（消费端，Git 只存不跑）
```

## 组件

| 组件 | 位置 | 说明 |
|---|---|---|
| 镜像服务 | 仓库 `scripts/_infra/mirror.py`（systemd 运行） | 按需镜像 GitHub raw（白名单 `repos.json`，TTL 6h），**只监听隧道地址 192.168.40.1:18080** |
| 生成器 | `scripts/domain-rules/` | 见上 |
| 来源清单 | `scripts/domain-rules/sources.json` | 上游仓库 / 文件、目标表名、手工修正文件 |
| 手工清单 | `scripts/domain-rules/input/manual/` | `manual-blacklist.txt`、`exclude-blacklist.txt`、`manual-cn-domains.txt`；**以仓库为准** |
| 分发 | `scripts/_infra/publish.sh` | 把 `output/` 平铺到 `/srv/github-mirror/static/ros/` |
| 日更脚本 | `/usr/local/bin/ros-rules-daily.sh` | cron 06:00，日志 `/var/log/ros-rules-daily.log` |
| 服务器数据 | `/srv/github-mirror/` | 只剩 `repos/`（Git 缓存）与 `static/`（HTTP 根）；不再放代码副本 |

## 产物与消费端

| 文件 | 消费端 |
|---|---|
| `cn-telecom.rsc`、`cn-mobile.rsc`、`blacklist.rsc`、`direct-ipv4*.rsc` | RouterOS 地址表（由 ROS 的 `sync.rsc` 导入） |
| `proxy-domain.rsc`、`cn.rsc`、`cn-unicom.rsc`、`cn-cernet.rsc` | RouterOS 备用产物（当前六表同步不导入；域名分流已交给 OxiDNS） |
| `proxy-domain.oxi.txt`、`cn-domains.oxi.txt`、`geosite.dat` | OxiDNS（ROS 拉取到本地后再 reload） |
| `proxy-domain.domains.txt`、`proxy-domain.exact.txt` | 通用中间产物（其它 DNS 可自行转换） |
| `direct-ipv4.rsc`、`direct-ipv4-nocm.rsc`、`direct-ipv4-noct.rsc` | RouterOS 地址表 / 按运营商派生 |

## 手工改动的即时生效

广州已启用 manual-watch：每分钟检测六个手工输入，覆盖远端提交与本地编辑，变化命中才生成发布，无变化静默。上游仍日更，不监控 sources.json。ROS 不轮询、不做哈希。网页改清单可等下一分钟生成发布，也可显式执行：

```text
# 1) 广州：只重生成手工相关产物，不拉上游
/usr/local/bin/manual-refresh rules        # 域名类
/usr/local/bin/manual-refresh direct-ip    # 地址类
```

```text
# 2) ROS：按需联动
/system script run manual-refresh
```

ROS 的 `manual-refresh` = 跑一次 `ros-rules-sync`（一次拉齐六张地址表 + OxiDNS 三份规则）+ 重启容器立即重载。

**OxiDNS 已改成只读本地文件**：它的 `rules_download` 任务调用与插件定义均已删除，只保留每天 07:00 的 `rules_reload`。
容器启动与重载都不再需要网络，隧道或镜像不可用也不影响它；规则新鲜度由 ROS 侧拉取决定
（06:30 放好文件 → 07:00 本地 reload 生效，没有中断）。重启容器只是为了让手工改动立即生效，
代价是约 1–8 秒 DNS 不可用。

## 两条重要语义（勿破坏）

- **后缀域 vs 精确域**：`DOMAIN-SUFFIX,x` / 裸域名 → PSL 收敛成注册域（`domain:x`、`match-subdomain=yes`）；`DOMAIN,x` 保持完全限定（`full:x`、`match-subdomain=no`）。
  **精确域绝不能被放大成整域**——曾把一个精确 FQDN 概括成整域，导致该域下的国内子域被错误代理。
- **IP 段类规则**：Telegram / Twitter 这类客户端把服务器 IP 硬编码、不走 DNS 的服务，域名规则无效，必须靠 `blacklist` 的 CIDR 段兜底。
  判定方法：**看客户端是否直连 IP**。

## 增删规则

| 需求 | 操作 |
|---|---|
| 某域名强制走代理 | 改 `input/manual/manual-blacklist.txt` |
| 某域名改走直连 | 从代理源排除并确认进入国内表；只改 exclude 仍可能命中非国内 → JP |
| 国内域名表补充 | 改 `input/manual/manual-cn-domains.txt`（`domain:` / `full:` 前缀） |
| 增加/更换域名源 | 改 `sources.json` 的 `tables.proxy-domain.sources`（引用写法 `owner/repo:path`，新仓库还要加进 `_infra/repos.json` 白名单） |
| 增删被墙服务的 IP 段 | 改 `sources.json` 里 `blacklist` 的 `sources` / `extra_cidrs` |
| 新增消费端格式 | 写适配器读 `output/proxy-domain.domains.txt`，产物加进 `_infra/publish.sh` |

手工输入每分钟自动生成发布；sources.json 与上游仍等日更或全量生成。设备按 06:30 / 07:00 消费，立即生效执行 ROS 手动入口。

06:15 direct-ip 全量日更保留，旧十分钟 DNS 刷新 cron 已注释。同步先下载校验，再逐表导入和替换文件；不是跨六表与三文件的整体事务，后续失败不会自动回滚已完成步骤。

## 验证

- 产物是否当天：`tail -20 /var/log/ros-rules-daily.log`；HTTP 根 `ls -la /srv/github-mirror/static/ros/`
- ROS 侧：`/ip firewall address-list print count-only where list=DIRECT_IP`、`/system script run ros-rules-sync`
- OxiDNS 侧：容器内 `wget -qO- http://127.0.0.1:9199/api/logs | grep "snapshot built"` 看各 provider 的规则数