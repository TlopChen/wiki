# Git 基础操作

## 日常三步

```bash
git add .                     # 挑选要保存的改动
git commit -m "说明这次改了啥"  # 打包成一版
git push                      # 推送到 GitHub
```

别人/别的机器有更新时，先 `git pull` 再干活。

## 本机环境（2026-08-29 配置）

- 全局身份：`TlopChen` + `114015032+TlopChen@users.noreply.github.com`（隐私代发邮箱）
- 默认分支：`main`
- 仓库统一放 `C:\Users\TlopC\Documents\GitHub\<仓库名>`
- 推送凭据已缓存在 Windows 凭据管理器，免登录
- GitHub 账号：**TlopChen**

## 踩坑记录

- `LF will be replaced by CRLF` 警告：Windows/Linux 换行符差异，无害，忽略
- 密码、API token、内网拓扑**绝不提交进公开仓库**
- 仓库命名避免和工具重名（比如用 `git` 当仓库名，找起来容易混）
