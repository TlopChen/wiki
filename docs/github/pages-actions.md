# GitHub Pages 与 Actions

## Pages（静态网页托管）

- 任何仓库都能开：**Settings → Pages**
- 个人主站：建 `TlopChen.github.io` 仓库，地址即 `https://tlopchen.github.io`
- 项目页：`https://tlopchen.github.io/<仓库名>`
- 只能跑静态内容（HTML/JS/CSS/Markdown 编译产物），没有后端
- 本知识库就是 Pages：mkdocs 编译 → `gh-deploy` 推到 `gh-pages` 分支 → Pages 发布

## Actions（CI，自动干活机器人）

在仓库 `.github/workflows/*.yml` 里定义任务，触发条件到了 GitHub 就替你跑：

- `on: push` —— 每次 push 触发（自动质检）
- `on: schedule` —— 定时任务（cron 语法，GitHub 机器上跑）
- 免费额度：公开仓库每月 2000 分钟

### 本仓库的 workflow

`.github/workflows/deploy.yml`：每次 push 到 main → 装 mkdocs-material → `mkdocs gh-deploy`。
运行记录在仓库的 **Actions** 标签页，失败会发邮件。

### 注意

- 密码/私钥等敏感信息走仓库 **Settings → Secrets and variables → Actions**，绝不写进 yml
- 公开仓库全世界可见，含隐私的内容放私有仓库（私有仓库 Actions 有免费额度限制）
