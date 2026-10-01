# GitHub Pages 与 Actions

## Pages（静态网页托管）

- 任何仓库都能开：**Settings → Pages**
- 个人主站：建 `TlopChen.github.io` 仓库，地址即 `https://tlopchen.github.io`
- 项目页：`https://tlopchen.github.io/<仓库名>`
- 只能跑静态内容（HTML/JS/CSS/Markdown 编译产物），没有后端
- 本知识库就是 Pages：mkdocs 编译 → `gh-deploy` 推到 `gh-pages` 分支 → Pages 发布

## Actions（CI，自动干活机器人）

在仓库 `.github/workflows/*.yml` 里定义任务，触发条件到了 GitHub 就替你跑：

- `on: push` —— 每次 push 触发（自动质检、自动部署）
- `on: schedule` —— 定时任务（cron 语法，在 GitHub 的机器上跑）
- `on: workflow_dispatch` —— 给页面加一个「手动运行」按钮，想重跑时不用空提交
- 免费额度：公开仓库每月 2000 分钟

### 本仓库的 workflow

`.github/workflows/deploy.yml`：push 到 main（或手动触发）→ 装 mkdocs-material → `mkdocs build --strict` → `mkdocs gh-deploy --force`。

```yaml
name: deploy-wiki
on:
  push:
    branches: [main]
  workflow_dispatch:
concurrency:
  group: deploy-wiki
  cancel-in-progress: true
permissions:
  contents: write
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
          cache: pip
      - run: pip install -r requirements.txt
      - run: mkdocs build --strict
      - run: mkdocs gh-deploy --force
```

三个容易忽略的开关：

- **`permissions: contents: write`**：`gh-deploy` 要把编译结果推到 `gh-pages` 分支，权限不够会 403。新仓库默认的 `GITHUB_TOKEN` 权限可能只有只读。
- **`concurrency`**：连着 push 两次时，取消上一次还没跑完的部署，避免两个部署互相覆盖、线上状态说不清。
- **`cache: pip`**：把依赖下载缓存住，构建快一些。

### `--strict` 是道闸

`mkdocs build --strict` 把警告直接当错误：`nav` 里登记了不存在的文件、页面里的站内链接指错路径、front matter 写坏，都会**构建失败**。

构建失败 = **不部署 = 旧站点保持在线**。所以「改了但线上没变」时，第一件事是看 Actions 那一栏是不是红的，而不是怀疑缓存。

### 本地没装 mkdocs 也能改站点

这台机器上没有 mkdocs（服务器上也没有 pip），编辑流程是：

1. 改 Markdown；
2. push（或让有凭据的机器 push）；
3. Actions 负责安装依赖 + 严格构建 + 部署，1–2 分钟后刷新线上页面看效果。

也就是说**构建验证交给 Actions**，本地只做内容写作。

### 注意

- 密码/私钥等敏感信息走仓库 **Settings → Secrets and variables → Actions**，绝不写进 yml 或 Markdown。
- 公开仓库全世界可见，含隐私的内容放私有仓库（私有仓库的 Actions 有免费额度限制）。
- 部署目标是 `gh-pages` 分支，**不要手工往它提交**——下次部署会覆盖，且历史会乱。
