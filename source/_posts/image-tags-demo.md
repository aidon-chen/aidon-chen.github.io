---
title: 多媒体与标签功能演示
date: 2025-07-15 11:00:00
updated: 2025-07-15 11:00:00
tags:
  - 演示
  - 多媒体
  - Butterfly
  - Markdown
categories:
  - 技术
keywords: butterfly, 标签插件, 图片, 视频, bilibili, youtube
description: 演示 Butterfly 主题的多媒体嵌入、标签插件和各类 Markdown 高级用法。
top_img:
cover:
mathjax: false
comments: true
toc: true
---

## Markdown 基础格式

### 文本样式

- **粗体** (Bold)
- *斜体* (Italic)
- ~~删除线~~ (Strikethrough)
- `行内代码` (Inline Code)
- [超链接](https://hexo.io)
- > 引用块 (Blockquote)

### 列表

#### 无序列表

- Hexo 博客框架
  - Markdown 渲染器
  - 丰富的插件生态
- Butterfly 主题
  - 响应式设计
  - 暗色模式支持
- GitHub Pages 托管

#### 有序列表

1. 安装 Node.js
2. 安装 Hexo CLI
3. 初始化博客项目
4. 安装 Butterfly 主题
5. 配置并发布

### 代码块

```python
def fibonacci(n: int) -> list[int]:
    """Generate the first n Fibonacci numbers."""
    if n <= 0:
        return []
    seq = [0, 1]
    for _ in range(2, n):
        seq.append(seq[-1] + seq[-2])
    return seq[:n]

print(fibonacci(10))
# Output: [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]
```

```javascript
// Quick sort implementation
function quickSort(arr) {
  if (arr.length <= 1) return arr;
  const pivot = arr[Math.floor(arr.length / 2)];
  const left = arr.filter(x => x < pivot);
  const middle = arr.filter(x => x === pivot);
  const right = arr.filter(x => x > pivot);
  return [...quickSort(left), ...middle, ...quickSort(right)];
}
```

```typescript
interface BlogPost {
  title: string;
  date: Date;
  tags: string[];
  categories: string[];
  content: string;
}

const post: BlogPost = {
  title: "Hello World",
  date: new Date(),
  tags: ["hexo", "blog"],
  categories: ["tech"],
  content: "Welcome to my blog!",
};
```

## 表格

| 功能 | Butterfly | 其他主题 | 说明 |
|------|:---------:|:--------:|------|
| 暗色模式 | ✅ | ⚠️ | 自动/手动切换 |
| MathJax | ✅ | ⚠️ | 需要额外配置 |
| 本地搜索 | ✅ | ⚠️ | 需要插件 |
| 图片懒加载 | ✅ | ❌ | 开箱即用 |
| PWA | ✅ | ❌ | 内置支持 |

## 视频嵌入

### B站 视频

使用 iframe 嵌入 B站 视频：

<iframe
  src="//player.bilibili.com/player.html?aid=17000101&bvid=BV1yx411L7tX&cid=27752094&page=1&high_quality=1"
  scrolling="no"
  border="0"
  frameborder="no"
  framespacing="0"
  allowfullscreen="true"
  width="100%"
  height="450">
</iframe>

### YouTube 视频

<iframe
  width="100%"
  height="450"
  src="https://www.youtube.com/embed/dQw4w9WgXcQ"
  title="YouTube video player"
  frameborder="0"
  allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture"
  allowfullscreen>
</iframe>

## 任务清单

- [x] 创建 Hexo 项目
- [x] 安装 Butterfly 主题
- [x] 配置 MathJax
- [x] 创建示例文章
- [ ] 配置评论系统 (Giscus)
- [ ] 绑定自定义域名
- [ ] 添加 PWA 支持

### 折叠面板

<details>
<summary>点击展开 — 搭建步骤详情</summary>

1. **初始化项目**
   ```bash
   npx hexo-cli init blog
   cd blog
   npm install
   ```

2. **安装 Butterfly 主题**
   ```bash
   npm install hexo-theme-butterfly
   npm install hexo-renderer-pug hexo-renderer-stylus
   ```

3. **配置主题**
   - 将 `_config.yml` 中 `theme` 设为 `butterfly`
   - 创建 `_config.butterfly.yml` 配置文件

4. **本地预览**
   ```bash
   hexo server
   ```

</details>

---

> 这篇文章展示了 Butterfly 主题的各类 Markdown 渲染能力，供后续写作参考。
