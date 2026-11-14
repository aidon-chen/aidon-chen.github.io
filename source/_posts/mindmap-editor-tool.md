---
title: 给 161 个测试节点找个家：基于 simple-mind-map 的可协作思维导图工具
date: 2026-11-14 16:40:00
categories: [前端, 工具]
tags: [思维导图, simple-mind-map, esbuild, yjs, 协作, 测试]
description: 给 AI 文档管理平台的 161 节点测试路线图做的可编辑思维导图：esbuild 合成四插件 UMD 包，deepmerge 垫片、禁用 minify、样式拍平三个坑，以及 y-webrtc 协作。
comments: true
toc: true
---

## 前言

AI 文档管理平台的测试用例整理出来后有 161 个节点、6 个阶段（环境准备、冒烟测试、主站全量、后台与权限矩阵、异常边界、回归收口）。放在表格里看不动，放在思维导图工具里编辑不了——XMind 这类软件导来导去太麻烦，我想要的是：打开就能改、改完自动存、还能发给另一个人一起勾进度。

所以写了个小工具：基于 [simple-mind-map](https://github.com/wanglin2/mind-map)（v0.14.0，MIT）的可编辑思维导图，内置富文本、关联线、图片导出、多人协作四个插件。工具与导图数据完全分离，换一份 `roadmap.json` 就能复用到任何项目。

## 时间线（按 git 记录）

仓库的提交历史只有一条：

- **2026-11-04 13:31**：唯一一次提交 `新增独立思维导图测试工具`，推到 GitHub 并创建仓库（13:31:44）。
- **截至 2026-11-14**：`master` 分支没有新的 commit。

不过仓库内的 `vendor/PLUGINS.md` 留了带日期的工作记录，可以补出实际过程：**2026-11-03** 完成 simple-mind-map 四插件打包、浏览器端到端自动化验证；**2026-11-04** 修复节点自定义样式被静默忽略的问题并同步拍平 `roadmap.json`。

## 工具长什么样

一个单页：深色顶栏放工具栏，中间是导图画布，底部一行快捷键提示。操作全部跟手：

1. 点选节点后 `Tab` 加子节点、`Enter` 加同级、`Delete` 删除，也可用工具栏按钮；
2. 双击节点直接出 Quill 富文本工具条（加粗、字号、颜色）；
3. 关联线：选中起点 → 点「＋ 关联线」→ 点目标节点，点选连线按 `Delete` 删除；
4. 导出 PNG / SVG / JSON，也能导入 JSON；「恢复初始」一键还原路线图；
5. 「加入协作」输入房间名，同一房间名的浏览器实时互相同步。

数据变化后 400ms 防抖写入 localStorage，页面关闭前（`beforeunload`）再强制落盘一次，刷新不丢。

## 工具与数据分离

页面启动时 `fetch('./roadmap.json')`，拿到的是 `{ "root": { "data": { "text": "..." }, "children": [...] } }` 结构，去掉外层包装直接喂给构造函数。当前内置的就是那份 161 节点的测试路线图；换成自己的导图，只需要改这个 JSON 文件，页面一行代码不用动。节点还可以带扁平的自定义样式（`fillColor`、`fontSize`、`borderRadius` 等）。

这也是它叫「测试工具」而不是「应用」的原因——没有后端、没有账号体系，`python3 -m http.server 8877` 起个静态服务就能跑（fetch 不支持 `file://` 直开）。

## 打包：把四个插件合进一个 IIFE

npm 包的 `dist/` 只有核心 UMD 产物，富文本、关联线、导出、协作这些插件全在 `src/plugins/` 下以 ESM 源码形式提供，需要自己用 esbuild 合成一份：

```bash
esbuild entry.js --bundle --format=iife --global-name=simpleMindMap \
  --preserve-symlinks \
  --alias:deepmerge=./shims/deepmerge.js \
  --define:process.env.NODE_ENV='"production"'
```

产物 `simpleMindMap.full.js` 1.6MB（未压缩），包含核心 + RichText / AssociativeLine / Export / Cooperate，页面一个 `<script>` 引入。Cooperate 不走 `usePlugin`，实例化后 `setProvider` 绑定房间名和信令地址。入口文件和垫片都留档在仓库 `build/` 目录，重新打包照着做即可。

## 三个坑

打包过程踩了三个坑，`PLUGINS.md` 里都记了，这里展开最关键的两个：

1. **deepmerge 版本漂移**。库声明的是 `deepmerge ^1.5.2`（官方 UMD 内置 isPlainObject 语义），esbuild 直接打包会装到 v4，v4 的 `isMergeableObject` 不排除 DOM 元素——结果构造函数里 `deepmerge(defaultOpt, opt)` 把 `el` 这个 DOM 容器深拷贝成空对象，运行时报 `this.el.getBoundingClientRect is not a function`。解法是 `--alias:deepmerge=./shims/deepmerge.js`，垫片只对纯对象和数组做深合并。

2. **`--minify` 直接崩**。压缩后运行时抛 `RangeError: Maximum call stack size exceeded`，未压缩一切正常——esbuild 的 minify 与 quill/yjs 源码存在兼容问题。本地工具不差这 1.6MB，交付未压缩产物。

3. **节点样式必须拍平**。库读取的是 `data` 上的扁平属性（`data.fillColor`），嵌套在 `data.style = {...}` 里的样式会被静默忽略，回退默认主题色——表现为根节点永远是绿色而不是自定义深色。修复就是 `Object.assign(data, style)` 把样式拍平，`roadmap.json` 也同步改了。

## 几个渲染细节

161 个节点的大图还带来几个小问题：

- **整图 fit 的可读性**：161 节点 fit 之后会被缩到 10% 左右，文字糊成一条线。所以初始和「回到中心」都设了缩放下限 100%——按可读比例铺满，宁可超出画布用平移看，也不缩小到不可读。
- **文字被顶出节点框**：库按 `measureText` 给节点的 `foreignObject` 预估固定宽高，但没重置 `<p>` 的浏览器默认 margin（上下各 1em），白字落白底只见上半截。CSS 里重置段落外边距 + `translateY(-50%)` 垂直居中兜底。
- **容器宽高为 0**：flex 布局下异步初始化时容器可能尚未完成布局，库会直接抛「容器元素el的宽高不能为0」。加了个 `whenLayoutReady`，用 `requestAnimationFrame` 等到拿到非零尺寸再 boot。

## 协作是怎么做的

协作走 yjs CRDT + y-webrtc：`new Cooperate({ mindMap })` 后 `setProvider(null, { roomName, signalingList })`，同一房间名的浏览器实时互相同步，连富文本格式和关联线都在同步范围内。

信令默认用公共的 `wss://signaling.yjs.dev`（仅测试用），地址会记忆到 localStorage；生产环境建议用 `y-webrtc` 包自带的 `y-webrtc-signaling` 自建一个，在弹窗里填入自建地址即可，不需要改代码。

## 浏览器端到端验证

打包完成后做了一轮自动化浏览器验证，全部通过：渲染、富文本编辑 + 自动保存、关联线创建（真实点击流程）、PNG/SVG 导出、协作加入房间、增节点回归。唯一没闭环的是删线操作（点选连线变高亮后按 Delete）——合成事件对 `svg` 连线的选中模拟不可靠，这一项留了 30 秒人工点验记录在 `PLUGINS.md` 里。

## 仓库

代码在这里：[aidon-chen/mindmap-editor](https://github.com/aidon-chen/mindmap-editor)（MIT）。

如果你的项目也需要一张「能编辑、能存、能协作」的导图——测试路线图、学习计划、复盘图都行——把 `roadmap.json` 换成自己的数据就能直接用；想加别的插件（小地图、搜索、XMind 导出等），`PLUGINS.md` 里有完整的插件清单和统一接入流程。
