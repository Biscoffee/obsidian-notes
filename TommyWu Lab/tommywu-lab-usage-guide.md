---
title: "TommyWu's Lab 使用指南"
published: 2026-05-04
description: "这篇文章记录 TommyWu's Lab 的日常写作、同步、预览、构建和发布流程。"
tags: ["Blog", "Astro", "Fuwari", "Obsidian"]
category: "工程"
draft: false
---

# TommyWu's Lab 使用指南

这篇指南记录这个博客的日常使用方式。它适合当成自己的操作手册：以后忘了怎么新增文章、怎么同步、怎么预览、怎么排查问题，就回来翻这一篇。

当前博客由三部分组成：

- 博客项目：`/Users/tommywu/tommywu-lab`
- Obsidian 公开写作目录：`/Users/tommywu/Obsidian/TommyWu Lab`
- 静态站点框架：Fuwari / Astro

只有放在 `TommyWu Lab` 这个 Obsidian 文件夹里的内容会被同步到网站。其他私人笔记不会被博客读取。

## 日常写作流程

最常用的流程是：

```bash
cd /Users/tommywu/tommywu-lab
pnpm dev
```

然后打开：

```text
http://127.0.0.1:4321/
```

接着在 Obsidian 里编辑：

```text
/Users/tommywu/Obsidian/TommyWu Lab
```

每次启动 `pnpm dev` 时，博客会先自动执行同步，把 Obsidian 公开目录里的 Markdown 文件同步到博客项目的 `src/content/posts`。

如果开发服务器已经开着，新增或修改文章后，可以手动同步一次：

```bash
pnpm sync-posts
```

## 新增一篇文章

在 Obsidian 的 `TommyWu Lab` 文件夹中新建一个 `.md` 文件，例如：

```text
my-first-technical-note.md
```

文件顶部需要写 frontmatter：

```md
---
title: "我的第一篇技术笔记"
published: 2026-05-04
description: "这是一篇测试文章。"
tags: ["Astro", "Obsidian"]
category: "工程"
draft: false
---
```

然后在下面写正文：

```md
# 我的第一篇技术笔记

这里是正文内容。
```

写完后运行：

```bash
cd /Users/tommywu/tommywu-lab
pnpm sync-posts
```

如果本地服务器开着，页面会自动刷新或在刷新浏览器后看到新文章。

## frontmatter 字段说明

每篇文章建议都包含这些字段：

```md
---
title: "文章标题"
published: 2026-05-04
description: "文章摘要，会用于 SEO、首页卡片和搜索结果。"
tags: ["Tag1", "Tag2"]
category: "分类"
draft: false
---
```

字段含义：

- `title`：文章标题，必填。
- `published`：发布日期，必填，格式必须是 `YYYY-MM-DD`。
- `description`：文章摘要，建议填写。
- `tags`：标签，可以有多个。
- `category`：分类，通常放一个大方向，比如 `工程`、`AI`、`iOS`、`读书`。
- `draft`：是否草稿。`false` 表示发布，`true` 表示隐藏。

同步脚本会检查 `title` 和 `published`。如果缺少这两个字段，同步会失败，这样可以避免不完整的文章被发布。

## 隐藏文章

如果文章还没写完，但你想保留在公开写作目录里，可以设置：

```md
draft: true
```

Fuwari 会把它当成草稿，不展示在网站上。

等要发布时再改回：

```md
draft: false
```

## 删除文章

删除文章只需要两步：

1. 从 Obsidian 的 `TommyWu Lab` 文件夹里删除对应 `.md` 文件。
2. 在博客项目中运行同步：

```bash
cd /Users/tommywu/tommywu-lab
pnpm sync-posts
```

同步脚本会清空并重建博客项目里的文章目录，所以网站内容会和 Obsidian 公开目录保持一致。

## 修改文章文件名

文章 URL 和文件名有关。

例如：

```text
hello-tommywu-lab.md
```

会生成类似这样的地址：

```text
/posts/hello-tommywu-lab/
```

如果你修改文件名，文章地址也会变化。已经发出去的旧链接可能会失效，所以正式发布后的文章文件名尽量少改。

## 图片使用

最稳妥的方式是把图片放在文章旁边。

例如：

```text
TommyWu Lab/
  my-post.md
  my-image.png
```

在 Obsidian 里可以写：

```md
![[my-image.png]]
```

同步脚本会把它转换成标准 Markdown：

```md
![my-image.png](./my-image.png)
```

也可以手动写标准 Markdown 图片：

```md
![图片说明](./my-image.png)
```

支持同步的图片格式包括：

- `.png`
- `.jpg`
- `.jpeg`
- `.gif`
- `.webp`
- `.avif`
- `.svg`

## Obsidian 双链

同步脚本会简单处理 Obsidian 双链：

```md
[[hello-tommywu-lab]]
```

会转换成：

```md
[hello-tommywu-lab](/posts/hello-tommywu-lab/)
```

如果你写：

```md
[[hello-tommywu-lab|这篇测试文章]]
```

会转换成：

```md
[这篇测试文章](/posts/hello-tommywu-lab/)
```

注意：双链转换是按文件名推断 URL 的，所以文件名最好使用英文、小写和短横线。

## 推荐文件命名

建议使用英文小写和短横线：

```text
build-my-blog-with-astro.md
notes-on-rag.md
ios-memory-debugging.md
```

不太建议使用：

```text
我的第一篇博客.md
My First Blog.md
2026/05/04/文章.md
```

中文标题可以放在 `title` 里，文件名保持稳定、简短、URL 友好。

## 本地预览

启动开发服务器：

```bash
cd /Users/tommywu/tommywu-lab
pnpm dev
```

访问：

```text
http://127.0.0.1:4321/
```

常看的页面：

- 首页：`http://127.0.0.1:4321/`
- 归档：`http://127.0.0.1:4321/archive/`
- 关于：`http://127.0.0.1:4321/about/`
- RSS：`http://127.0.0.1:4321/rss.xml`

## 构建检查

正式发布前建议运行：

```bash
cd /Users/tommywu/tommywu-lab
pnpm check
pnpm build
```

`pnpm check` 用来检查 Astro、TypeScript 和内容结构。

`pnpm build` 会生成生产环境静态网站，并生成搜索索引、RSS、sitemap。

当前 `pnpm build` 已经配置为先自动同步 Obsidian 文章，所以不用额外先跑 `pnpm sync-posts`。

## 常用命令速查

```bash
cd /Users/tommywu/tommywu-lab
```

同步文章：

```bash
pnpm sync-posts
```

本地预览：

```bash
pnpm dev
```

类型和内容检查：

```bash
pnpm check
```

生产构建：

```bash
pnpm build
```

预览生产构建结果：

```bash
pnpm preview
```

## 发布前检查清单

发布文章前可以扫一遍：

- `title` 是否准确。
- `published` 是否是正确日期。
- `description` 是否能概括文章内容。
- `tags` 是否不要太散。
- `category` 是否和已有分类一致。
- `draft` 是否已经改成 `false`。
- 图片是否放在公开目录里。
- 文章里是否有不想公开的私人信息。
- 本地 `pnpm build` 是否通过。

## 常见问题

### 同步失败：缺少 title

说明某篇文章没有写：

```md
title: "文章标题"
```

补上后重新运行：

```bash
pnpm sync-posts
```

### 同步失败：缺少 published

发布日期必须是：

```md
published: 2026-05-04
```

不要写成：

```md
published: 2026/05/04
published: May 4, 2026
```

### 文章没有显示

优先检查：

- 文件是否在 `/Users/tommywu/Obsidian/TommyWu Lab`。
- 文件扩展名是否是 `.md` 或 `.mdx`。
- `draft` 是否是 `true`。
- 是否运行过 `pnpm sync-posts` 或重新启动过 `pnpm dev`。

### 图片没有显示

优先检查：

- 图片是否也在 `TommyWu Lab` 目录里。
- 图片路径是否写对。
- 图片文件名是否和 Markdown 里的引用一致。
- 图片格式是否是支持同步的格式。

### 搜索搜不到最新文章

搜索索引是生产构建时生成的。本地开发环境里搜索可能不是最终效果。

要测试搜索，运行：

```bash
pnpm build
pnpm preview
```

然后在预览页面里测试搜索。

## 后续可以优化的地方

现在这套流程已经能稳定写作和发布。之后还可以继续优化：

- 替换默认头像和 favicon。
- 配置真实域名到 `astro.config.mjs` 的 `site`。
- 接入 Cloudflare Pages 自动部署。
- 给 Obsidian 增加文章模板。
- 给常用分类和标签定一套命名规则。
- 增加自动提交和发布脚本。

先把写作流程跑顺，比一开始就追求完美更重要。
