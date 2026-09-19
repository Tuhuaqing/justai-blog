# frozen_string_literal: true

source "https://rubygems.org"

# ---------------------------------------------------------------------------
# 本地开发工具链：与 GitHub Pages 官方构建环境保持一致。
#
# github-pages gem 锁定了 GitHub Pages 当前支持的 Jekyll 版本与全部插件白名单
# （含 jekyll-remote-theme、jekyll-paginate、jekyll-sitemap、jekyll-feed、
#  jekyll-seo-tag、jekyll-include-cache 等），
# 因此本地 `bundle exec jekyll build` 的结果与线上部署结果一致。
#
# 说明：GitHub Actions 端使用官方 action `actions/jekyll-build-pages` 构建，
# 它自带同一套工具链，不读取本文件；本文件仅用于本地开发/预览。
# ---------------------------------------------------------------------------
gem "github-pages", group: :jekyll_plugins

# 性能更好的分页插件（不在 Pages 白名单内，仅本地可选，勿用于线上）
# group :development do
#   gem "jekyll-paginate-v2"
# end
