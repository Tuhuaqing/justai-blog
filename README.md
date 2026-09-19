# justai-blog

个人技术博客：写 Markdown，`git push`，自动发布。

- **线上地址**：<https://tuhuaqing.github.io>（绑定自定义域名后更新此链接）
- **技术栈**：Jekyll · GitHub Pages · GitHub Actions · Minimal Mistakes 主题
- **写作-发布闭环**：新建 Markdown → commit → push → 几分钟后自动上线

## 架构

```text
Markdown 文章 (_posts/)
        │  git push
        ▼
GitHub 仓库 ──触发──► GitHub Actions (deploy.yml)
                        │  Jekyll 构建：Markdown → HTML + 主题渲染
                        ▼
                  GitHub Pages（CDN + HTTPS）
```

构建使用官方 action `actions/jekyll-build-pages`，与 GitHub Pages 内置
工具链完全一致；本地开发使用同一套 `github-pages` 工具链（见 `Gemfile`），
保证本地与线上构建结果一致。

## 目录结构

```text
justai-blog/
├── _posts/            # 文章（唯一需要日常关注的目录）
├── _pages/            # 独立页面：关于 / 归档 / 分类 / 标签 / 搜索 / 404
├── _data/
│   └── navigation.yml # 顶部导航菜单
├── assets/
│   └── images/        # 文章配图与头像
├── .github/workflows/
│   └── deploy.yml     # 自动构建与部署（push 到 main 触发）
├── _config.yml        # 站点配置：信息 / 主题 / 插件 / 默认值
├── Gemfile            # 本地开发依赖（github-pages 工具链）
└── README.md
```

## 如何写文章

在 `_posts/` 下新建文件，**文件名必须是 `YYYY-MM-DD-标题-slug.md`**：

```markdown
---
title: "我的新文章"
date: 2026-09-20 10:00:00 +0800
categories: [教程]
tags: [Jekyll]
description: "一句话摘要，用于列表页与 SEO。"
---

正文使用标准 Markdown。
```

要点：

- Front Matter 只用标准字段：`title` / `date` / `categories` / `tags` / `description`；
- 不要在文章里写主题专用 HTML，主题行为由 `_config.yml` 的 `defaults` 统一控制；
- `date` 不要写未来时间，否则文章不会发布（Jekyll 默认跳过未来文章）；
- 图片放 `assets/images/`，文中用 `![说明](/assets/images/xxx.png)` 引用。

## 如何发布

```bash
git add .
git commit -m "post: 我的新文章"
git push
```

推送后到仓库 **Actions** 页面可查看构建进度，构建失败会显式报红并收到邮件。

## 本地开发

```bash
bundle install              # 首次或 Gemfile 变更后
bundle exec jekyll serve    # 启动本地预览 http://127.0.0.1:4000
```

本地预览与线上使用同一套 GitHub Pages 工具链，所见即所得。

> Ruby 环境要求：与 [GitHub Pages](https://pages.github.com/versions/) 兼容的
> Ruby 版本即可（建议 3.x）。本机若未安装 Ruby，任何安装方式均可，
> 项目不依赖系统级 Ruby。

## 如何切换主题

**只改一个文件的一个字段**：[`_config.yml`](_config.yml) 中的 `remote_theme`：

```yaml
# 当前主题
remote_theme: "mmistakes/minimal-mistakes@4.28.1"

# 例如换成官方 minima 主题：
# remote_theme: "jekyll/minima@2.5.1"
```

可选的收尾工作（不影响文章，只影响观感）：

1. `_config.yml` 中的 `minimal_mistakes_skin`、`defaults` 段落
   是 Minimal Mistakes 专用选项，换主题时按新主题文档调整或删除；
2. `_data/navigation.yml` 按新主题的导航格式改写；
3. `_pages/` 中各页面的 `layout` 值按新主题的布局名调整
   （页面内容本身无需改动）。

**所有 `_posts/` 文章永远不需要任何改动**——这是本项目的内容/主题解耦约定。

主题目录参考：[jamstackthemes.dev](https://jamstackthemes.dev/ssg/jekyll/)、
[jekyllthemes.org](https://jekyllthemes.org/)（选择支持 `remote_theme` 的主题）。

## 自定义域名

默认地址为 `https://<username>.github.io`。绑定自有域名两步：

1. **DNS 解析**（在你的域名服务商处配置）：

   | 场景 | 记录 | 主机记录 | 记录值 |
   | --- | --- | --- | --- |
   | 子域名（推荐，如 `blog.example.com`） | `CNAME` | `blog` | `tuhuaqing.github.io` |
   | 顶级域（如 `example.com`） | `A` / `AAAA` | `@` | GitHub Pages 官方公布的 IP（见[官方文档](https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site/managing-a-custom-domain-for-your-github-pages-site)） |

2. **仓库设置**：`Settings → Pages → Custom domain` 填入域名保存，
   并勾选 **Enforce HTTPS**。

   也可以在仓库根目录创建 `CNAME` 文件（内容只有一行域名）替代页面操作。

绑定成功后，建议把 `_config.yml` 中的 `url` 更新为新域名（影响 RSS、
canonical 等绝对链接），并同步更新本 README 顶部的线上地址。

## 常见问题

| 现象 | 排查方向 |
| --- | --- |
| push 后网站没更新 | 看 Actions 页面：构建是否失败、失败日志的行号 |
| 文章不出现 | 文件名是否符合 `YYYY-MM-DD-title.md`；`date` 是否为未来时间 |
| 构建报 Liquid 错误 | 文中代码块含 `{{ }}`/`{% %}`，需用 `{% raw %}` 包裹 |
| 本地与线上样式不一致 | 本地执行 `bundle install` 更新工具链 |

## 安全约定

- `.gitignore` 已排除 `.env`、`*.pem`、`*.key`、credentials/token 类文件；
- 本项目无需任何 Secret：Actions 使用内置的 `GITHUB_TOKEN`，
  权限已在 workflow 中最小化（`contents: read` + `pages: write` + `id-token: write`）；
- 永远不要把 token、密钥写进文章或配置文件。
