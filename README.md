# justai-blog

Just Tu 的个人技术博客。文章以 Markdown 存放在仓库中，推送到 `main` 后由
GitHub Actions 构建并发布至 GitHub Pages。

- **线上地址**：<https://justtu.com>
- **技术栈**：Ruby 4.0.7 · Jekyll 4 · Agency Jekyll Theme · GitHub Actions · GitHub Pages
- **主题**：[Agency Jekyll Theme](https://github.com/raviriley/agency-jekyll-theme)

## 架构

```text
_posts/*.md
    │ git push main
    ▼
GitHub Actions
    │ Ruby 4 + Bundler + Jekyll build
    ▼
_site/ Pages artifact
    ▼
GitHub Pages + justtu.com
```

## 本地开发

项目通过 [`.ruby-version`](.ruby-version) 固定为 Ruby `4.0.7`。确认当前终端使用
该版本后，首次或依赖变化后执行：

```bash
ruby -v
bundle install
bundle exec jekyll serve
```

本地预览地址为 <http://127.0.0.1:4000>。提交前运行：

```bash
bundle exec jekyll build
```

## 如何写文章

在 `_posts/` 创建 `YYYY-MM-DD-title-slug.md`：

```markdown
---
title: "我的新文章"
date: 2026-09-20 10:00:00 +0800
categories: [教程]
tags: [Jekyll]
description: "一句话摘要，用于文章列表和 SEO。"
---

正文使用标准 Markdown。
```

- 保持 Front Matter 包含 `title`、`date`、`categories`、`tags` 与 `description`；
- 不要在文章中依赖主题专用 HTML，文章只写 Markdown；
- 图片放入 `assets/images/`，使用 `![说明]({{ site.baseurl }}/assets/images/xxx.png)` 引用；
- 日期不要写成未来时间，否则 Jekyll 默认不会发布文章。

## 如何发布

```bash
bundle exec jekyll build
git add .
git commit -m "post: 我的新文章"
git push origin main
```

[`deploy.yml`](.github/workflows/deploy.yml) 会在 `main` 推送时使用 Ruby 4、
Bundler 与本仓库的 `Gemfile.lock` 构建站点，上传 `_site` artifact，再由
GitHub Pages 发布。进度和日志可在仓库的 Actions 页面查看。

## 主题与内容边界

当前使用 Agency 的远程主题，并在 [`_config.yml`](_config.yml) 中固定上游 commit：

```yaml
remote_theme: "raviriley/agency-jekyll-theme@d477a171ec9633c980c8ef9098eeac39b60ceba3"
```

主题相关内容集中在以下位置：

- `_config.yml`：主题、Jekyll、文章默认 layout 和插件；
- `_data/navigation.yml`：导航；
- `_data/sitetext.yml`：首页横幅和页脚文本；
- `_layouts/`：为文章、独立页、归档、分类、标签和搜索提供与 Agency 一致的布局。

文章内容留在 `_posts/`，使用标准 Markdown，因此不需要因为主题迁移而修改。

## 如何换主题

1. 修改 `_config.yml` 的 `remote_theme` 或替换为新主题的 `theme` 配置；
2. 根据新主题文档调整 `_layouts/`、`_data/navigation.yml` 与 `_data/sitetext.yml`；
3. 保留 `_posts/` 的标准 Front Matter 和 Markdown；
4. 执行 `bundle update`、`bundle exec jekyll build`，确认构建成功后再推送。

更换到需要不同 Jekyll 版本的主题时，也要同步修改 `Gemfile` 中的 Jekyll 约束与
`.ruby-version`，然后更新 `Gemfile.lock`。

## 自定义域名

根目录 [`CNAME`](CNAME) 设置为 `justtu.com`。域名解析与 HTTPS 状态由 GitHub
仓库的 `Settings -> Pages` 管理；修改域名后，同时更新 `_config.yml` 的 `url`。

## 安全约定

- `.gitignore` 排除了 `.env`、私钥、证书、token/secret 类文件和构建产物；
- Actions 使用 `GITHUB_TOKEN` 的最小部署权限，不需要把凭据写入仓库；
- 不要把 Token、密码或私钥提交到文章、配置或工作流文件。
