---
title: "使用 Jekyll + GitHub Pages 搭建个人技术博客"
date: 2026-09-19 10:00:00 +0800
categories: [教程, DevOps]
tags: [Jekyll, GitHub Pages, GitHub Actions, Markdown, 静态网站]
description: "从零搭建一个可长期维护、自动发布的个人技术博客：为什么选静态站点、Jekyll 的工作原理、Markdown 到 HTML 的生成过程、主题机制、GitHub Actions 自动构建与 Pages 发布、自定义域名配置，以及常见问题排查。"
---

> 本文就是这套架构的"自举"产物：你现在看到的这篇文章，本身就是一个放在
> `_posts/` 目录下的纯 Markdown 文件，push 之后由 GitHub Actions 自动构建发布。

## 为什么选择静态博客

对个人技术博客来说，动态站点（WordPress、Ghost 等）的数据库、运行时、
安全补丁、服务器费用，都是长期负担。而静态博客只需要回答一个问题：
**一堆 Markdown 文件，如何变成一个网站？**

| 维度 | 动态博客 | 静态博客（本项目） |
| --- | --- | --- |
| 运行时依赖 | 数据库 + 应用服务器 | 无，纯 HTML/CSS/JS |
| 部署方式 | 服务器运维 | Git push 即发布 |
| 安全面 | 插件/内核漏洞需持续修补 | 无后端可攻击 |
| 写作体验 | 在线编辑器 | 本地任意编辑器 + Git |
| 内容所有权 | 存在数据库里 | 纯文本文件，永不锁定 |
| 成本 | 服务器/托管费 | GitHub Pages 免费 |
| 迁移成本 | 导出转换 | 复制目录即可 |

代价是你需要接受"写作即写文件"的心智模型——但对程序员来说，这反而是最舒服的方式。

## 整体架构

先看全貌，后文逐层拆解：

```text
┌──────────────┐   git push    ┌───────────────────────────┐
│  本地仓库     │ ───────────► │  GitHub 仓库（源码即内容）  │
│              │               │                           │
│ _posts/*.md  │               │  push 事件触发             │
│ _config.yml  │               └────────────┬──────────────┘
│ 主题配置      │                            │
└──────────────┘                            ▼
                                  ┌───────────────────┐
                                  │  GitHub Actions    │
                                  │  (deploy.yml)      │
                                  │                   │
                                  │  Jekyll 构建       │
                                  │  Markdown → HTML   │
                                  │  主题 → 渲染       │
                                  └────────┬──────────┘
                                           │  upload artifact
                                           ▼
                                  ┌───────────────────┐
                                  │  GitHub Pages      │
                                  │  全球 CDN + HTTPS  │
                                  └────────┬──────────┘
                                           ▼
                              https://<username>.github.io
                              或 https://your-domain.com
```

每日写作的完整流程只有三步：

```bash
# 1. 在 _posts/ 下新建一篇 Markdown
vim _posts/2026-09-20-my-new-post.md

# 2. 提交
git add .
git commit -m "post: 我的新文章"

# 3. 推送，几分钟后自动上线
git push
```

## Jekyll 的基本原理

Jekyll 是一个**静态站点生成器**：读入一组文本文件，输出一组 HTML 文件。
它没有数据库，没有运行时，"构建"就是全部的动态过程。

核心机制是**约定优于配置**，几个特殊目录有固定含义：

| 目录/文件 | 作用 |
| --- | --- |
| `_posts/` | 文章，文件名必须形如 `YYYY-MM-DD-title.md` |
| `_pages/` | 独立页面（关于、归档、404 等） |
| `_data/` | 结构化数据（如导航菜单 `navigation.yml`） |
| `_config.yml` | 全站配置：站点信息、主题、插件、默认值 |
| `assets/` | 原样复制的静态资源（图片、CSS、JS） |
| `_site/` | 构建输出目录（应加入 `.gitignore`） |

构建时，Jekyll 对每个文件执行同一条流水线：

1. 读取文件，解析文件头部的 **Front Matter**；
2. Markdown 正文交给转换器（本项目为 kramdown）转为 HTML 片段；
3. 片段填充进主题的 **Liquid 模板**（layout）；
4. 写入 `_site/`，目录结构即 URL 结构。

## Markdown 如何生成 HTML

每篇文章是一个带 Front Matter 的纯文本文件。Front Matter 用 YAML 写在
文件最开头，两条 `---` 之间：

```markdown
---
title: "使用 Jekyll + GitHub Pages 搭建个人技术博客"
date: 2026-09-19 10:00:00 +0800
categories: [教程, DevOps]
tags: [Jekyll, GitHub Pages, GitHub Actions, Markdown]
description: "从零搭建一个可长期维护、自动发布的个人技术博客。"
---

正文从这里开始，使用标准 Markdown……
```

这些元数据会变成模板里可用的变量，例如 `page.title`、`page.tags`。
本项目的正文渲染采用 **GFM（GitHub Flavored Markdown）**，
所以表格、围栏代码块、删除线等写法与你在 GitHub 上看到的完全一致：

```markdown
| 列 A | 列 B |
| --- | --- |
| 1   | 2    |

`行内代码`、**加粗**、[链接](https://jekyllrb.com) 均为标准语法。
```

转换由 kramdown 完成。它是一个纯 Ruby 的 Markdown 实现，
GitHub Pages 官方支持，无需额外配置。

**关键点：文章正文不要写主题专用的 HTML。**
正文只依赖标准 Markdown 与标准 Front Matter，
所有主题相关的行为（布局、侧栏、目录、阅读时长）都集中在
`_config.yml` 的 `defaults` 段统一注入。这保证了文章与主题解耦，
将来换主题时，所有文章一个字符都不用改。

## Theme 如何工作

主题本质上是一组可复用的**布局（layouts）、片段（includes）和样式（SASS）**。
Jekyll 的查找规则是：本地目录优先，本地没有的去主题里找。
因此主题负责"长什么样"，你的仓库只保留"写了什么"。

本项目的主题方案是 `remote_theme`（写在 `_config.yml` 中）：

```yaml
# 主题直接从 GitHub 仓库引用，并固定版本，不进入本地依赖
remote_theme: "mmistakes/minimal-mistakes@4.28.1"
```

这样做的好处：

- **零文件侵入**：主题代码不进仓库，仓库里只有内容和配置；
- **版本可控**：固定 tag，升级主题是一次显式的版本号变更，可回滚；
- **一键换肤**：切换主题只改这一行；
- **构建时拉取**：GitHub Actions 构建时自动下载主题，无需本地安装。

本项目使用 [Minimal Mistakes](https://mmistakes.github.io/minimal-mistakes/)
——一个维护了十年以上、文档完善、对技术博客非常友好的主题，
自带的功能包括：响应式布局、暗色系皮肤切换、代码高亮、
目录（TOC）、阅读时长、分类/标签归档页、站内搜索、RSS、SEO 标签。

文章与主题的边界在本项目中被刻意收紧为：

```text
文章（_posts/）  →  只写 title/date/categories/tags/description + 标准 Markdown
主题（_config.yml）→  remote_theme、皮肤、defaults（布局/侧栏/TOC 等）
导航（_data/navigation.yml）→  菜单项
```

## GitHub Actions 如何自动构建

仓库中的 `.github/workflows/deploy.yml` 定义了自动构建流程。
push 到 `main` 分支时触发：

{% raw %}
```yaml
name: Build and Deploy Jekyll site to GitHub Pages

on:
  push:
    branches: ["main"]
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: "pages"
  cancel-in-progress: false

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - uses: actions/configure-pages@v6
      - uses: actions/jekyll-build-pages@v1
        with:
          source: ./
          destination: ./_site
      - uses: actions/upload-pages-artifact@v5
        with:
          path: ./_site

  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - id: deployment
        uses: actions/deploy-pages@v5
```
{% endraw %}

几个值得注意的设计：

- **`actions/jekyll-build-pages` 是官方构建器**，与 GitHub Pages 内置的
  Jekyll 工具链完全一致（Jekyll 版本、插件白名单同步更新），
  不存在"本地能构建、线上不认识"的漂移问题；
- **权限最小化**：`contents: read` + `pages: write` + `id-token: write`，
  只授予部署所需的最小权限；
- **major 版本固定**：所有 action 固定主版本号，行为可预期；
- **失败即显式报错**：任何一步失败，workflow 直接变红，
  GitHub 会发邮件通知，不会静默发布旧内容；
- **并发控制**：`concurrency` 保证同一时刻只有一个部署，
  连续 push 时排队执行而非互相取消，避免读到半成品。

## GitHub Pages 如何发布

部署阶段用了两个官方 action，配合成一套标准的"工件传递"模式：

1. `upload-pages-artifact` 把 `_site/` 打包成一个 artifact；
2. `deploy-pages` 将 artifact 发布到 Pages 的托管层，
   并返回最终 URL（`environment.url`）。

Pages 的发布源（Source）设置为 **GitHub Actions** 而非
传统的"从分支发布"——这是官方当前推荐方式，部署历史、失败原因、
预览 URL 都可以在仓库的 Actions 和 Environments 页面追溯。

发布完成后，Pages 自动提供：

- 全球 CDN 加速；
- 全站 HTTPS（Let's Encrypt 证书，自动续期）；
- 自定义域名绑定（见下一节）。

## 自定义域名如何工作

默认地址是 `https://<username>.github.io`。绑定自己的域名需要两件事：

**第一，在 DNS 服务商处添加解析记录**：

| 记录类型 | 主机记录 | 记录值 | 适用场景 |
| --- | --- | --- | --- |
| `CNAME` | `www` 或 `blog` | `<username>.github.io` | 子域名（推荐） |
| `A` / `AAAA` | `@` | GitHub Pages 官方公布的 IP | 顶级域（Apex） |

> 顶级域不支持 CNAME（DNS 规范限制），所以要用 A/AAAA 记录指向
> GitHub 公布的服务 IP，或使用 DNS 服务商提供的 CNAME 扁平化。

**第二，告诉 GitHub 你的域名**：

- 方式一（推荐）：仓库 `Settings → Pages → Custom domain` 填入域名并保存；
- 方式二：在仓库根目录放一个 `CNAME` 文件，内容只有一行域名。

随后在 Pages 设置中勾选 **Enforce HTTPS** 强制 HTTPS。
DNS 生效通常几分钟到几小时不等，可用 `dig` 验证：

```bash
dig blog.example.com +short
# 应返回 <username>.github.io 解析出的 IP 地址
```

## 常见问题

**Q：push 了文章，但线上没更新？**
先看仓库 Actions 页面的 workflow 状态。常见原因：
文章文件名不符合 `YYYY-MM-DD-title.md` 格式；
Front Matter 的 YAML 语法错误（冒号后少空格、引号不闭合）；
构建失败会在 Actions 日志里给出精确的报错行号。

**Q：文章写好了却不出现在列表里？**
Jekyll 默认不发布"未来时间"的文章。本站时区为 `Asia/Shanghai`，
如果 `date` 写成了未来时刻，文章会被跳过（构建不报错，只是不出现）。

**Q：本地怎么预览？**

```bash
bundle exec jekyll serve
# 打开 http://127.0.0.1:4000 ，修改文件自动刷新
```

本地使用与线上相同的 `github-pages` 工具链（见 `Gemfile`），
所见即所得。

**Q：构建时报 Liquid 语法错误？**
文章里如果出现 &#123;&#123; 双花括号 &#125;&#125; 或 &#123;% 百分号 &#125; 这类
Liquid 模板写法（即使写在行内代码或代码块里），都会被 Liquid 解析而报错。
解决方法：在代码块前一行写 &#123;% raw %&#125;、后一行写
&#123;% endraw %&#125;，把整块内容原样保护起来（本文「GitHub Actions
如何自动构建」一节的 workflow 代码就是这样处理的）。

**Q：想换主题怎么办？**
只改 `_config.yml` 的 `remote_theme` 一行（详见仓库 README
「切换主题」章节），所有文章零改动。

**Q：图片怎么放？**
统一放在 `assets/images/`，文章里用 Jekyll 官方的标准写法引用
（`site.baseurl` 保证在任何部署路径下都正确，绑定自定义域名后自动为空）：

{% raw %}
```markdown
![图片说明]({{ site.baseurl }}/assets/images/2026-09-19-demo.png)
```
{% endraw %}

下面是一张占位图的实际引用效果（文件位于 `assets/images/post-placeholder.svg`）：

![图片占位符示例]({{ site.baseurl }}/assets/images/post-placeholder.svg)

## 总结

这套架构把"写作"压缩到了最简：

- **写**：`_posts/` 下一个 Markdown 文件，只含标准 Front Matter；
- **发**：`git push`，其余全部自动化；
- **托管**：GitHub Pages 免费 CDN + HTTPS；
- **演进**：主题一行配置可换，文章资产永远属于你。

静态博客的核心思想是**把复杂性从运行时移到构建时**：
没有数据库就没有数据丢失，没有后端就没有安全补丁，
没有服务器就没有账单。剩下的只有一个 Git 仓库——
这对一个打算写十年的技术博客来说，是最稳的底座。

## 参考链接

- [Jekyll 官方文档](https://jekyllrb.com/docs/)
- [GitHub Pages 文档](https://docs.github.com/en/pages)
- [Minimal Mistakes 主题文档](https://mmistakes.github.io/minimal-mistakes/docs/)
- [GitHub Actions 文档](https://docs.github.com/en/actions)
