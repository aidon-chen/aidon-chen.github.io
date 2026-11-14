---
title: 161 个测试节点的导图工具：simple-mind-map 加四个插件
date: 2026-11-14 16:40:00
categories: [前端, 工具]
tags: [思维导图, simple-mind-map, esbuild, yjs, 协作, 测试]
description: 给测试路线图做的可编辑导图页面，以及打包 simple-mind-map 四个插件时踩的坑：deepmerge 版本、minify 崩溃、样式被静默忽略。
comments: true
toc: true
---

## 前言

AI 文档管理平台的测试用例整理完是 161 个节点，分 6 个阶段：环境准备、冒烟、主站全量、后台与权限、异常边界、回归收口。一开始用表格管，管着管着就乱了，加一条用例得往下翻半天，哪块还没测也看不出来。

想过用 XMind，导进导出太烦。干脆拿 simple-mind-map 包一层，做成一个能直接改、改完自己存、还能拉个人一起勾的页面。

## 时间

仓库只有一次提交，2026-11-04 13:31，就一句「新增独立思维导图测试工具」，推上去时仓库也刚建好，后来没有再动。

折腾其实都在提交之前。`vendor/PLUGINS.md` 里记着两天：11 月 3 号把四个插件打成一个包，跑了一轮浏览器自动化验证；11 月 4 号发现节点自定义样式不生效，翻源码才找到原因，顺手把 `roadmap.json` 也改了。

## 用起来是这样

单页，顶栏工具栏，中间画布，底下一行快捷键提示。点中节点按 `Tab` 加子节点、`Enter` 加同级、`Delete` 删掉，双击进富文本编辑，能改字号颜色。关联线先点起点，点「＋ 关联线」，再点目标节点。导出有 PNG、SVG、JSON，导入只认 JSON，「恢复初始」把图打回原样。

改完 400ms 防抖写一次 localStorage，关页面前再补一次，基本不丢。底部右侧会显示「已自动保存 14:32」，看着踏实。

## 工具和数据分开

页面启动先 fetch 一个 `roadmap.json`，结构是 `{ root: { data, children } }`，去掉外面一层壳直接塞给构造函数。现在里面装的是那份测试路线图，换项目就换这个文件，页面不用动。

代价是不能双击打开，fetch 在 `file://` 下会被拦，得起个 `python3 -m http.server 8877`。也算提醒自己这不是给终端用户装的应用，是个本地小工具。

## 打包最费劲

npm 包的 dist 里只有核心，四个插件都在 `src/plugins/` 下是 ESM 源码，得自己合。用 esbuild 打成 IIFE，1.6MB 没压缩，一个 script 标签引进去。Cooperate 比较特殊，不走 `usePlugin`，是 new 出来再 `setProvider`。

命令行里 `--preserve-symlinks` 和 deepmerge 的 alias 少一个就起不来。前后卡了三处：

1. **deepmerge 版本**。库声明 `^1.5.2`，esbuild 直接装会拿到 v4。v4 的 `isMergeableObject` 不认 DOM 元素，构造函数里 `deepmerge(defaultOpt, opt)` 就把 `el` 这个容器深拷贝成空对象，报 `this.el.getBoundingClientRect is not a function`，盯着这句话看了很久也没想通跟 el 什么关系。最后加 alias 指向自己写的垫片，只合并纯对象和数组，才过。
2. **minify 崩**。加 `--minify` 打包正常，一跑就 `RangeError: Maximum call stack size exceeded`，去掉就好。查了下跟 quill、yjs 有关，具体哪行没深究。反正本地工具，1.6MB 就 1.6MB。
3. **样式被吃掉**。给根节点设了深色，跑起来还是库默认那个绿。翻源码发现它读的是 `data.fillColor` 这种平铺属性，我写成了 `data.style.fillColor`，嵌套的直接忽略，也不报错。改成 `Object.assign` 拍平就好，`roadmap.json` 里也得同步。

## 161 个节点带来的麻烦

图太大，fit 一下缩到 10%，字糊成一条线，点都点不准。所以初始和「回到中心」都把缩放下限锁在 100%，宁可超出去拖着看，也不缩到看不清。

还有个更奇怪的，节点文字只显示上半截。库按 `measureText` 给 `foreignObject` 估宽高，但没管 `<p>` 默认的 1em 上下外边距，文字被顶出框，白字落白底就只剩上面一点。CSS 里把 `p` 的 margin 清零，再加个 `translateY(-50%)` 兜一下。

另外 flex 布局下初始化太快，容器宽高还是 0，库直接抛「容器元素el的宽高不能为0」。加了个 `requestAnimationFrame` 轮询，等到有尺寸再 boot。

## 协作

yjs 加 y-webrtc。`new Cooperate({ mindMap })` 然后 `setProvider(null, { roomName, signalingList })`，同一个房间名的浏览器实时同步，富文本格式和关联线都能带过去。

信令默认走 `wss://signaling.yjs.dev`，公共的，连不连得上看网络。要正经用就自己起一个，y-webrtc 包里带着，弹窗里换地址就行，代码不用动。

## 验证

打包完跑了一轮浏览器自动化：渲染、双击编辑加自动保存、关联线、PNG 和 SVG 导出、加协作房间、增删节点都过了。只有删线没验完，点选连线让它高亮那一步，合成事件在 SVG 上不太靠得住，最后手动点了一遍，`PLUGINS.md` 里记着这条待办。

## 仓库

https://github.com/aidon-chen/mindmap-editor ，MIT。

拿去装自己的东西，改 `roadmap.json` 就行。想加插件，`PLUGINS.md` 里列了十来个，小地图、搜索、XMind 导出都有，流程照第 4 节走，主要还是那几个 esbuild 参数别漏。
