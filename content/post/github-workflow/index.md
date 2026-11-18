---
title: "与 GitHub 建立一套可用的工作流"
date: 2026-11-18T13:40:00+08:00
tags: ["GitHub", "工作流", "CI"]
---

单个仓库、一个人写代码的时候，Git 怎么用都行。但只要有两件事出现——**要长期维护**、**有第二个人参与**——就必须有一套固定流程，否则历史会乱到没人敢改。

这篇记录我这套站点（Hugo + GitHub + Cloudflare Pages）实际在跑的工作流。

## 一、分支模型：只保留两条线

不要搞 Git Flow 那套 `develop` / `release` / `hotfix`，个人和小团队根本维护不住。只需要：

| 分支 | 用途 | 规则 |
|---|---|---|
| `main` | 随时可发布 | **保护**，不允许直接 push |
| `feature/xxx` | 单个功能或文章 | 从 main 拉，合完就删 |

```bash
git switch main
git pull --rebase
git switch -c feature/add-post
# 改完之后
git push -u origin feature/add-post
```

一个人在本地写也得走分支——习惯成自然，出问题时好回退。

## 二、认证：用 SSH，别用 token

个人访问令牌（PAT）一旦泄漏，别人就能用你的身份操作仓库。更糟的是它经常被手滑写进配置文件、脚本、甚至提交记录里。

**推荐做法：SSH 密钥。**

```bash
# 生成（不要设空密码以外的选项，deploy key 通常不带 passphrase）
ssh-keygen -t ed25519 -C "my-laptop" -f ~/.ssh/github_key

# 查看公钥，粘贴到 GitHub → Settings → SSH and GPG keys
cat ~/.ssh/github_key.pub
```

如果是给**某一台服务器**用，就加到仓库的 **Settings → Deploy keys**，并勾选 **Allow write access**；这样权限只限制在这一个仓库，比账号级密钥安全。

顺手把 SSH 配好别名：

```
# ~/.ssh/config
Host github-myrepo
  HostName github.com
  User git
  IdentityFile ~/.ssh/github_key
  IdentitiesOnly yes
```

然后：

```bash
git remote set-url origin git@github-myrepo:用户名/仓库名.git
```

从此推送无需输入任何凭据。

> 如果团队确实要用 token，请只勾选最小权限，并放进 GitHub Secrets 或系统凭据管理器，**绝不能进仓库**。

## 三、提交规范：让历史自己会说话

用 Conventional Commits，格式：

```
类型(范围): 简述

可选正文

可选脚注
```

常用类型：

| 类型 | 含义 |
|---|---|
| `feat` | 新功能 |
| `fix` | 修复缺陷 |
| `docs` | 文档 |
| `style` | 格式（不影响逻辑） |
| `refactor` | 重构 |
| `perf` | 性能优化 |
| `chore` | 构建/依赖/杂项 |

示例：

```
feat(post): 新增 Docker 使用技巧

fix(theme): 修复文章日期在未来时不渲染的问题
```

好处很实际：`git log --oneline` 一眼能扫，将来自动化生成 CHANGELOG 也直接用得上。

## 四、Pull Request：哪怕只有自己也要提

很多人觉得一个人写代码提 PR 是形式主义。其实不是，它带来三样东西：

1. **Diff 视图**：GitHub 的对比页比 `git diff` 直观得多，能提前发现误提交的文件
2. **CI 卡点**：所有检查在合并前跑，问题不会进 `main`
3. **可追溯**：每个变更都有讨论记录和上下文

PR 描述建议固定三段：

```markdown
## 改了什么
## 为什么改
## 怎么验证
```

## 五、分支保护：把规则写死在仓库里

`main` 分支一定要开保护（Settings → Branches）：

- ✅ Require a pull request before merging
- ✅ Require approvals（有协作时至少 1 人）
- ✅ Require status checks to pass（挂上 CI）
- ✅ Require branches to be up to date before merging
- ❌ 关闭 Allow force pushes
- ❌ 关闭 Allow deletions

设置完，`git push origin main` 会直接被拒绝——这是好事，防止手滑。

## 六、CI：用 GitHub Actions 做构建校验

在 `.github/workflows/ci.yml` 里放一个最基础的检查，保证每次 PR 都能构建成功：

```yaml
name: CI

on:
  pull_request:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v3
        with:
          hugo-version: '0.170.1'
          extended: true

      - name: Build
        run: hugo --gc --minify --baseURL "https://example.com/"

      - name: Check output
        run: test -f public/index.html
```

关键点：**CI 里的 Hugo 版本要和本地一致**，否则会出现「本地能过、线上失败」。这也是 Cloudflare Pages 必须设 `HUGO_VERSION` 环境变量的原因。

## 七、部署：交给 Cloudflare Pages

本站的部署链路是：

```
本地写文章 → git push → GitHub → Cloudflare Pages 自动构建 → 全球 CDN
```

配置要点：

| 项 | 值 |
|---|---|
| Framework preset | Hugo |
| Build command | `hugo --gc --minify -b $CF_PAGES_URL` |
| Build output directory | `public` |
| 环境变量 | `HUGO_VERSION=0.170.1` |

每次 push 到 `main` 自动触发；其它分支和 PR 会生成独立的预览地址，非常适合 review。

## 八、Secrets 管理：这是最容易出事的一环

原则只有一条：**凡是密钥，一律不进仓库。**

- 放 GitHub：Settings → Secrets and variables → Actions
- 在 workflow 里这样引用：

```yaml
- name: Deploy
  env:
    API_TOKEN: ${{ secrets.API_TOKEN }}
  run: ./deploy.sh
```

- 本地凭据放 `~/.ssh/` 或系统凭据管理器，不放项目目录
- `.gitignore` 至少包含：

```gitignore
.env
*.pem
*.key
public/
resources/
node_modules/
```

**如果 token 已经泄漏过**（比如贴进了对话、写进了提交），正确做法是**立刻去后台撤销重签**，而不是删掉代码里那一行——历史记录里还留着。

## 九、发布与版本

```bash
git tag -a v1.2.0 -m "发布说明"
git push origin v1.2.0
```

然后在 GitHub 上基于 tag 建 Release，附上变更摘要。比翻 `git log` 找「上次发版改了啥」高效得多。

## 十、日常一天长什么样

```bash
# 1. 同步主线
git switch main && git pull --rebase

# 2. 开分支
git switch -c feature/xxx

# 3. 改动、提交（小步多次）
git add -p
git commit -m "feat: xxx"

# 4. 推送并开 PR
git push -u origin feature/xxx

# 5. CI 通过、review 完成后合并，删除分支
git switch main && git pull --rebase
git branch -d feature/xxx
```

---

这套流程的价值不在于「规范」，而在于**每一次改动都有据可查、每一步都能回退**。真正需要救火的时候，你会庆幸当初多按了几次分支和 PR。
