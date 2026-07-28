# 博客写作与发布指南

本仓库是一个基于 **Hexo 7.3.0 + Butterfly 5.6.0** 主题的静态博客，托管在 GitHub Pages。
本文档说明**如何编写一篇新文章、放在哪里、以及如何一键发布到线上**。

- 线上地址：<https://aidon-chen.github.io>
- 远程仓库：`git@github.com:aidon-chen/aidon-chen.github.io.git`
- 部署方式：推送 `main` 分支后，GitHub Actions 自动构建并发布，**无需手动部署**。

---

## 一、快速上手（TL;DR）

```bash
# 1. 首次使用先安装依赖（本地 node_modules 不完整时必须执行）
npm install

# 2. 新建文章（会在 source/_posts/ 下生成 你的文章标题.md）
npx hexo new "你的文章标题"

# 3. 编辑文章（用编辑器打开对应 .md，填写正文）

# 4. 本地预览（可选，浏览器打开 http://localhost:4000）
npx hexo server

# 5. 提交并推送到 main —— 剩下的构建与上线由 CI 自动完成
git add source/_posts/你的文章标题.md
git commit -m "add: 新文章 你的文章标题"
git push origin main
```

推送后约 1~2 分钟，访问 <https://aidon-chen.github.io> 即可看到新文章。

---

## 二、文章文件格式

- **格式**：Markdown（`.md`），文件顶部是用 `---` 包裹的 **YAML front-matter**（元信息），下方是正文。
- **编码**：UTF-8。

### 最小可用 front-matter

新建文章时默认脚手架（`scaffolds/post.md`）生成的字段，够用即可：

```markdown
---
title: 文章标题
date: 2026-07-21 10:00:00
tags:
categories:
description:
mathjax: false
---

正文从这里开始，用 Markdown 编写……
```

### 完整 front-matter（参考现有文章 `hello-world.md`）

需要更丰富的展示效果时可使用 Butterfly 主题支持的完整字段：

```markdown
---
title: Hello World — 欢迎来到我的博客
date: 2026-07-19 09:00:00
updated: 2026-07-19 09:00:00
tags:
  - 随笔
  - 博客
categories:
  - 生活
keywords: hello world, 博客, butterfly, hexo
description: 我的第一篇博客文章，介绍这个博客的搭建过程与技术栈。
top_img:
cover:
mathjax: false
comments: true
toc: true
---
```

### 字段说明

| 字段 | 含义 | 备注 |
| --- | --- | --- |
| `title` | 文章标题 | 必填 |
| `date` | 发布时间 | 必填，格式 `YYYY-MM-DD HH:mm:ss` |
| `updated` | 更新时间 | 可选 |
| `tags` | 标签 | 可用列表或内联数组，见下 |
| `categories` | 分类 | 可用列表或内联数组，见下 |
| `keywords` | SEO 关键词 | 可选 |
| `description` | 摘要/描述 | 可选，用于列表页与 SEO |
| `top_img` | 文章顶部大图 | Butterfly 专属，可留空 |
| `cover` | 文章封面图 | Butterfly 专属，可留空 |
| `mathjax` | 是否启用数学公式 | 需要 LaTeX 公式时设为 `true` |
| `comments` | 是否开启评论 | Butterfly 专属 |
| `toc` | 是否显示文章目录 | Butterfly 专属 |

**标签 / 分类的两种写法**（等效，任选其一）：

```yaml
# 写法 A：列表
tags:
  - Hexo
  - GitHub Pages
categories:
  - 教程

# 写法 B：内联数组
tags: [Hexo, GitHub Pages]
categories: [教程]
```

---

## 三、存放位置与文件命名

- **所有文章都放在**：`source/_posts/`
- 文件名即文章的 URL 路径的一部分。本站配置 `new_post_name: :title.md`，即**文件名默认就是标题**。
- 例如 `source/_posts/my-first-post.md` 生成后的访问路径类似：
  `https://aidon-chen.github.io/2026/07/21/my-first-post/`
- 建议英文文件名用短横线连接（如 `how-to-write-a-post.md`），中文文件名也可用，但英文更利于 URL 可读性。

当前已有的示例文章可作参考：

```
source/_posts/
├── hello-world.md
├── image-tags-demo.md
├── latex-math-guide.md
└── github-pages-hexo-butterfly-deploy.md
```

---

## 四、图片与资源

本站配置 `post_asset_folder: false`，**不会**为每篇文章生成独立的资源文件夹。图片统一放在共享目录：

- 把图片放到 `source/img/` 下，例如 `source/img/my-pic.png`
- 在文章中用**站点根路径**引用：

```markdown
![示意图](/img/my-pic.png)
```

（构建后 `source/img/` 会原样复制到站点根目录 `/img/`。）

---

## 五、新建文章的两种方式

### 方式 A（推荐）：命令生成

```bash
npx hexo new "文章标题"
```

会自动在 `source/_posts/` 下按脚手架生成 `文章标题.md`，并填好 `title` 和 `date`。

> 前提：已执行过 `npm install`（本地 `node_modules` 完整）。

### 方式 B：手动新建

直接在 `source/_posts/` 下新建一个 `.md` 文件，把上面「最小可用 front-matter」复制进去，改好标题日期即可开始写。

---

## 六、本地预览与自检（可选，但推荐）

发布前在本地检查排版和构建是否正常：

```bash
# 首次或依赖有变动时
npm install

# 启动本地预览服务，浏览器访问 http://localhost:4000
npx hexo server

# 清理缓存并完整生成一遍，验证能否正常构建（输出在 public/）
npx hexo clean && npx hexo generate
```

> 注意：本项目未配置 npm scripts，也未配置本地 `hexo deploy`，所以**不要**用 `npm run build` 或 `hexo deploy`，构建命令就是上面的 `hexo generate`。

---

## 七、上传到 GitHub Pages 的完整流程

本站采用「源码分支 + 生成分支」分离的策略：

- `main` 分支 = **Hexo 源码**（你的 Markdown、配置等）
- `gh-pages` 分支 = **CI 自动生成的静态站点**（不要手动改）

### 发布步骤

```bash
# 1. 把新文章加入版本控制
git add source/_posts/你的文章.md
# 如有新增图片： git add source/img/你的图片.png

# 2. 提交
git commit -m "add: 新文章 你的文章标题"

# 3. 推送到 main（仅此一步即触发自动部署）
git push origin main
```

### 之后 CI 会自动完成（`.github/workflows/deploy.yml`）

1. 检出代码
2. 安装 Node.js 20，执行 `npm ci` 安装依赖
3. 执行 `npx hexo clean && npx hexo generate` 生成静态站点到 `public/`
4. 用 `peaceiris/actions-gh-pages@v3` 把 `public/` 推送到 `gh-pages` 分支
5. GitHub Pages 从 `gh-pages` 分支发布

约 1~2 分钟后访问 <https://aidon-chen.github.io> 即可看到新文章。可在仓库的 **Actions** 页查看构建进度。

> 首次配置时需在仓库 **Settings → Pages** 里把 Source 设为 `gh-pages` 分支（已配置过则无需重复）。

**要点：**

- 只需 `git push origin main`，**无需**本地 `hexo deploy`。
- **不要**手动修改或提交 `gh-pages` 分支，它由 CI 覆盖。
- `node_modules/`、`public/`、`db.json` 等已被 `.gitignore` 忽略，不需要提交。

> ⚠️ **部署前先确认 CI 已激活**：GitHub 仓库 `Settings → Actions → General` 需允许工作流运行（若提示权限不足，本地执行 `gh auth refresh -s workflow` 后再推 `main`）。若 CI 未激活，推 `main` **不会**自动更到线上——此时需手动部署 `gh-pages` 分支，或先激活 CI。

---

## 八、常见问题与提醒

- **数学公式不渲染**：在该文章 front-matter 中设置 `mathjax: true`（主题已全局开启 MathJax，但需逐篇声明）。
- **嵌入视频 / iframe 被吞掉**：Hexo 会用 DOMPurify 清洗原始 HTML，嵌入 `<iframe>`（如 B 站、YouTube）时必须用 `{% raw %}` 包裹：

  ```markdown
  {% raw %}
  <iframe src="//player.bilibili.com/player.html?bvid=xxxx" ... ></iframe>
  {% endraw %}
  ```

- **CI 没有触发 / 没有更新**：确认是推送到了 `main` 分支（而非 `gh-pages`）；并检查仓库 **Settings → Actions** 的工作流权限是否开启。
- **关于旧教程文**：`source/_posts/github-pages-hexo-butterfly-deploy.md` 里内联的工作流示例写的是 `npm install` / Node 22，但**实际生效**的 `.github/workflows/deploy.yml` 用的是 `npm ci` / Node 20。以实际的 `deploy.yml` 为准。

---

## 九、用 Obsidian + Obsidian Git 一键发布

如果你习惯在 Obsidian 里写作，可以直接把它连到这个博客，实现「写文 → 一键发布」，全程不用碰命令行。

**原理**：Obsidian Git 插件复用本文件夹已有的 `.git`。在 Obsidian 内点一下 `Commit & push`，就等价于 `git push origin main`，随即触发上面的 GitHub Actions 自动部署。这对应前面讨论的「方式 B」。

### 1. 把博客目录变成 Obsidian Vault

- 这个本地 `blog/` 文件夹**已经是一个 git 仓库，且其 `origin` 正是线上 Pages 的源仓库 `aidon-chen.github.io`**（可 `git remote -v` 验证）。也就是说「本地仓库 ↔ 远程仓库」这条线已经连好了。
- 它**目前还不是一个 Obsidian Vault**——本地没有 `.obsidian` 配置。用 Obsidian 打开它就会自动生成 `.obsidian`，从而成为「对应的 Obsidian 仓库」。
- 打开方式：Obsidian → 「打开其他仓库 / Open another vault」→ 选择文件夹 `G:\project\workbuddy\githubpages\blog`。
- **建议只打开 `blog/source/_posts/`**：整个项目里有 `node_modules/` 等目录，整目录打开会非常杂乱。

### 2. 安装 Obsidian Git 插件

- 设置 → 第三方插件 → 关闭安全模式 → 浏览社区插件 → 搜索 **Obsidian Git** → 安装并启用。

### 3. 配置 Git 凭证（关键，针对本机环境）

本仓库 remote 是 SSH：`git@github.com:aidon-chen/aidon-chen.github.io.git`。

- **SSH（对本仓库可用）**：本机 SSH 密钥 `~/.ssh/id_ed25519_github` 正是对 `aidon-chen.github.io` 有效的受限 deploy key，所以**对这个仓库** SSH 推送是通的。确保 SSH agent 已加载该密钥（或 `~/.ssh/config` 里为该 host 指定 `IdentityFile`）。
- **HTTPS + PAT（最稳妥）**：若 SSH 报 `Repository not found`，在 Obsidian Git 设置里把 remote 改为 `https://github.com/aidon-chen/aidon-chen.github.io.git`，并用 fine-grained PAT 作为密码（本机 `gh` 已登录时，git-credential-manager 也可免填）。
- 建议先在终端跑一次 `git push`，确认凭证 OK，再交给 Obsidian Git。

### 4. 推荐的 Obsidian Git 设置

- `Auto commit` + `Auto push` / `Vault backup interval`：可设为每 10 分钟自动提交并推送。**注意：每次推送都会触发站点重新部署**，按需开启。
- `Commit message`：留空用默认，或设模板如 `docs: update from obsidian`。
- `Pull before push`：开启，避免多端/多人冲突。
- `Disable push`：若只想本地版本管理、暂不自动上线，可关掉自动 push，改用手动命令面板执行 `Obsidian Git: Commit & push`。

### 5. 发布流程（全程在 Obsidian 内）

1. 在 `source/_posts/` 写文章（front-matter 写法见上文第二、三节）。
2. 命令面板（Ctrl/Cmd+P）→ `Obsidian Git: Commit & push`。
3. 等待 1~2 分钟，访问 <https://aidon-chen.github.io> 查看新文章。

### 6. 注意事项

- 同「八」：Obsidian 的 `[[双链]]`、`![[图片]]`、`> [!note]` 等语法 Hexo 默认不渲染，博客文章请用标准 Markdown（图片用 `![alt](/img/xxx.png)`）。
- `.obsidian/` 目录：当前未被 `.gitignore` 忽略，会被跟踪进仓库。不想提交 Vault 配置就把它加进 `.gitignore`；想团队共享配置则保留。
- 若仓库当前有其它未提交改动，Obsidian Git 的自动提交会一并带入；发布前建议保持工作区干净，或按需挑选文件。
- 自动推送会触发部署，这是预期行为。

---

## 附：项目结构速览

```
blog/
├── _config.yml                 # Hexo 主配置（url、theme、new_post_name 等）
├── _config.butterfly.yml       # Butterfly 主题配置
├── package.json                # 依赖（Hexo 7.3、Butterfly 5.6）
├── scaffolds/                  # 新建文章/页面的模板
│   └── post.md
├── source/
│   ├── _posts/                 # ★ 文章都放这里
│   ├── img/                    # ★ 图片放这里（用 /img/xxx 引用）
│   ├── about/  categories/  tags/  gallery/   # 主题页面
└── .github/workflows/deploy.yml  # GitHub Actions 自动部署
```
