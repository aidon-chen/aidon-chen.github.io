---
title: 从零搭建 GitHub Pages 静态博客：Hexo + Butterfly + Actions 自动部署
date: 2026-07-20 23:30:00
categories: [教程]
tags: [Hexo, GitHub Pages, GitHub Actions, 自动部署, Butterfly, 踩坑]
mathjax: true
---

## 前言

这篇是博客的第一篇正式文章，顺便把“从 0 到 1 把个人博客部署到 GitHub Pages”的完整过程记下来——包括环境、配置、CI 工作流，以及几个**真的会卡住新手**的坑和对应解法。后面自己或别人再部署同类静态站，照着抄就能少走弯路。

> 技术栈：Hexo 7.3 + hexo-theme-butterfly 5.6 + GitHub Pages（用户级）+ GitHub Actions。

---

## 一、技术选型：为什么是 Hexo + Butterfly

静态博客方案一大把（Hugo / Jekyll / Astro / VuePress…），选 Hexo + Butterfly 是因为它一次性满足了我的 5 条硬需求：

| 需求 | Butterfly 表现 |
|------|---------------|
| 1. Markdown + LaTeX/KaTeX 公式 | 内置 `mathjax` 开关，装 `hexo-filter-mathjax` 即生效 |
| 2. 多媒体（图片 + 视频链接嵌入） | 支持标准 Markdown 图片；视频用 `{% raw %}` 包裹 iframe 即可 |
| 3. 相册 / 图集页 | 原生 `type: gallery` 页面，无需额外插件 |
| 4. 时间线 + 分类 / 标签 | 归档、分类、标签页开箱即用 |
| 5. 高可扩展性 + 插件丰富 | Hexo 插件生态成熟，主题配置项极细 |

结论：**想要“开箱即用 + 高度可定制”的中文博客，Butterfly 是省心首选**。

---

## 二、环境准备

```bash
# 确认 Node.js 与 Git
node -v      # 建议 20+
npm -v
git --version
```

> 小提示：如果本机有多个 Node 版本，优先用受管（managed）版本，避免全局污染。

---

## 三、初始化 Hexo 项目

```bash
npm install -g hexo-cli
hexo init blog
cd blog
npm install
```

目录结构（关键部分）：

```
blog/
├── _config.yml            # 站点主配置
├── _config.butterfly.yml  # 主题配置（主题单独文件）
├── source/
│   ├── _posts/            # 文章放这里
│   ├── gallery/           # 相册页
│   ├── about/             # 关于页
│   ├── categories/        # 分类页
│   └── tags/              # 标签页
├── themes/butterfly/      # 主题源码（一般用 npm 包）
└── package.json
```

安装 Butterfly 主题（推荐用 npm 包方式，便于版本管理）：

```bash
npm install hexo-theme-butterfly
```

然后在 `_config.yml` 里把主题指过去：

```yaml
theme: butterfly
```

---

## 四、Butterfly 主题配置要点

主题配置写在 `_config.butterfly.yml`（**不是** `_config.yml`），几个最常用的开关：

```yaml
# 菜单：首页 / 归档 / 分类 / 标签 / 相册 / 关于
menu:
  - name: 首页
    icon: fa-solid fa-house
    path: /
  - name: 归档
    icon: fa-solid fa-archive
    path: /archives/
  - name: 分类
    icon: fa-solid fa-folder-open
    path: /categories/
  - name: 标签
    icon: fa-solid fa-tags
    path: /tags/
  - name: 相册
    icon: fa-solid fa-image
    path: /gallery/
  - name: 关于
    icon: fa-solid fa-address-card
    path: /about/

# 公式渲染：开 true 后需装 hexo-filter-mathjax
mathjax:
  enable: true

# 本地搜索
search:
  use: local

# 评论系统（没接第三方就关掉）
comments:
  use: null
```

相册页 `source/gallery/index.md`：

```yaml
---
title: 相册
type: gallery
---
```

---

## 五、GitHub Pages 仓库设置

1. 新建仓库，命名为 `<用户名>.github.io`（用户级 Pages 必须用这个命名，否则要走自定义域名）。
2. 本地 `_config.yml` 设好：

```yaml
url: https://aidon-chen.github.io
root: /
language: zh-CN
```

3. 推送后，到仓库 **Settings → Pages**，Source 选 `gh-pages` 分支（这个分支由 CI 自动生成并推送，见下一节）。

---

## 六、GitHub Actions 自动部署

核心思路：往 `main` 推代码 → GitHub 在云端 `npm install` → `hexo generate` → 推到 `gh-pages` → Pages 自动发布。**你本地连 Node 都不用装**。

文件：`.github/workflows/deploy.yml`（注意开头的 `.` 和全小写路径）：

```yaml
name: Deploy Blog to GitHub Pages

on:
  push:
    branches:
      - main
  workflow_dispatch:

permissions:
  contents: write
  pages: write
  id-token: write

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: "22"
          cache: npm

      - name: Install dependencies
        run: npm install

      - name: Build site
        run: npx hexo clean && npx hexo generate

      - name: Deploy to GitHub Pages
        uses: peaceiris/actions-gh-pages@v3
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./public
          publish_branch: gh-pages
```

> ⚠️ **不要写 `npm ci`**，除非你确定 `package-lock.json` 和 `package.json` 严格同步（见坑 7.3）。新手用 `npm install` 更稳。

**创建方式**：在 GitHub 网页上 `Add file → Create new file`，路径填 `.github/workflows/deploy.yml`，粘贴上面内容，直接 commit 到 `main` 即可——**完全不需要本地命令行**。

---

## 七、踩坑实录（重点）

下面这几个坑都是实打实踩过的，按顺序排好。

### 7.1 KaTeX 公式不渲染

- **现象**：文章里写 `$$E=mc^2$$` 或行内 `$a^2+b^2=c^2$`，页面出来是纯文本，公式没渲染。
- **根因**：只开了 `_config.butterfly.yml` 里的 `mathjax.enable: true`，但**没装渲染插件** `hexo-filter-mathjax`，MathJax 的 CDN 根本没注入页面。
- **修复**：

```bash
npm install hexo-filter-mathjax
```

并在**每篇需要公式的文章** front-matter 加 `mathjax: true`（全局开启有时不够，逐篇声明最稳）。本文开头就带了这个开关，下面验证一下行内 $a^2+b^2=c^2$ 和块级：

$$
\int_{-\infty}^{\infty} e^{-x^2}\,dx = \sqrt{\pi}
$$

能正常显示就说明通了。

### 7.2 视频 iframe 被剥离

- **现象**：在文章里贴 B 站 / YouTube 的 `<iframe ...></iframe>`，发布后页面**只剩空白**，源码里 iframe 不见了。
- **根因**：Hexo 默认的 `hexo-renderer-marked` 用了 DOMPurify 做 XSS 清洗，**会把 iframe 当成危险标签剥掉**。
- **修复**：用 Hexo 的 `{% raw %}` 标签把 iframe 包起来，告诉渲染器“这块原样输出、别清洗”：

```
{% raw %}
<iframe src="//player.bilibili.com/player.html?bvid=BVxxxx" ...></iframe>
{% endraw %}
```

注意：上面示例放在**代码块**里所以能原样显示；你实际写文章时要去掉代码块包裹，直接用 `{% raw %}...{% endraw %}` 包住 iframe 才行。

### 7.3 `npm ci` 锁文件不同步

- **现象**：CI 红叉，日志一堆 `npm error Missing: xxx@x.x.x from lock file`，退出码 1。
- **根因**：`npm ci` 是“严格安装”，要求 `package-lock.json` 和 `package.json` **逐字节一致**。中途手动往 `package.json` 加依赖（比如 7.1 的 `hexo-filter-mathjax`）却没重跑 `npm install` 更新锁文件，CI 上就炸。
- **修复（二选一）**：
  1. **懒人法**：把工作流里的 `npm ci` 改成 `npm install`（容错，自动补齐缺失包）。本文 6 节用的就是这招。
  2. **正规法**：本地 `npm install` 重新生成锁文件 → 推上去 → 改回 `npm ci`，保证可复现。

### 7.4 本地终端连不上 GitHub

- **现象**：`gh auth login` / `git push` 直接报 `dial tcp ...:443: connectex: ... failed to respond`，但浏览器能正常上 GitHub。
- **根因**：本机命令行网络出口到 `github.com:443` 超时（代理 / 防火墙限制），而浏览器走的是另一条通路。
- **Workaround（无需修网络）**：
  - 所有“建文件 / 改文件”操作走 **GitHub 网页**（仓库主人账号天然有 `workflow` 等权限，不受 CLI token scope 限制）；
  - 本文的 `deploy.yml` 和这篇文章本身，都能直接在网页 `Add file` 建好，commit 到 `main` 即触发部署。
  - 等本地终端网络通了，再回归 `git push` 工作流。

---

## 八、总结

到这里，一个**支持公式 / 多媒体 / 相册 / 分类标签、且推送即自动部署**的个人博客就完整跑通了。当前状态：

| 项 | 状态 |
|----|------|
| 线上地址 | https://aidon-chen.github.io/ ✅ |
| 自动部署（CI） | GitHub Actions 已激活 ✅ |
| 公式渲染 | hexo-filter-mathjax ✅ |
| 视频嵌入 | `{% raw %}` 包裹 ✅ |

**后续建议**：
1. 写新文章 → 放 `source/_posts/xxx.md` → 推 `main` → 全自动上线；
2. 想完全可复现，把 `npm install` 换回 `npm ci` 并维护好锁文件；
3. 有自定义域名再在 Settings → Pages 里填，不影响现有流程。

踩过的坑都在上面了，祝部署顺利 🚀
