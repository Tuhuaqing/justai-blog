# assets/images

文章配图目录。

约定：

- 文章图片统一放在本目录，按文章名或用途命名，例如 `2026-09-19-jekyll-blog-architecture.svg`；
- 文章中直接用标准 Markdown 图片语法引用：`![说明](/assets/images/xxx.png)`；
- 不要把图片放在 `_posts/` 里（会被当作站点资源处理，路径不可控）；
- `avatar.svg` 是作者侧栏头像，被 `_config.yml` 的 `author.avatar` 引用；
- `post-placeholder.svg` 是通用占位图，可复制改名使用。

首页横幅图片位于 `assets/img/technology-circuit-hero.jpg`，由
`assets/css/justtu-agency.css` 引用。
