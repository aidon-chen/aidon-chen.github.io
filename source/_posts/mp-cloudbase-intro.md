---
title: 第一次用云开发，云函数、数据库、存储各管什么
date: 2026-09-12 20:00:00
categories: [微信小程序, 云开发]
tags: [云开发, 云函数, 云数据库, 云存储]
description: 项目跑起来之后回头整理概念：云函数、云数据库、云存储分别管什么，以及一次「确认接受」在后台走了哪几步。
comments: true
toc: true
---

任务派发小程序跑了两周之后，我才回头把云开发的概念理清楚。之前是照着文档抄，能跑就行；等到要加功能、要排查问题时，发现脑子里没有一张完整的图，经常不知道一个请求到底经过了哪些地方。

这篇写给和我一样第一次接触云开发的人。概念不多，但每个概念最好挂一个项目里的真实例子，不然很容易看完就忘。

## 云开发是什么

简单说，微信把服务器、数据库、文件存储打包成了一个服务，你不用买机器、不用配域名、不用自己装数据库。小程序里调一个接口，代码就跑在微信的服务器上。

它和买云服务器的区别在于「谁负责运维」。买服务器，操作系统、运行时、数据库、备份、证书都要自己管；云开发把这些收走了，你只写函数和查询。它和云托管的区别在于「管到哪一层」：云托管还要你提供镜像或代码包，自己定端口和容器；云开发连容器都不需要，一个函数目录就是一个部署单元。

对这个项目来说，最实际的好处是登录。云函数里一行 `cloud.getWXContext().OPENID` 就能拿到当前用户的微信身份，不用自己发 token、存 session、写鉴权中间件。坏处也在这儿：整套东西长在微信里，离开微信就不好用。

## 云函数

云函数就是一段放在云端的 Node.js 代码。目录结构很固定：

```
cloudfunctions/
├── createTask/
│   ├── index.js
│   └── package.json
├── updateTask/
│   ├── index.js
│   └── package.json
└── ...
```

每个目录就是一个函数，入口是 `exports.main`：

```js
const cloud = require('wx-server-sdk');
cloud.init({ env: cloud.DYNAMIC_CURRENT_ENV });
const db = cloud.database();

exports.main = async (event, context) => {
  const { OPENID } = cloud.getWXContext();
  // ...
};
```

`event` 是前端传来的参数，`OPENID` 是调用者的微信身份。`DYNAMIC_CURRENT_ENV` 的意思是「用当前所在的环境」，同一个函数部署到测试环境和生产环境，不用改代码。

前端调用统一走 `wx.cloud.callFunction`：

```js
const res = await wx.cloud.callFunction({
  name: 'updateTask',
  data: { taskId, updates: { status: 'confirmed' } }
});
```

冷启动是云函数绕不开的东西。第一次调用、或者隔了很久没人调用时，云函数要重新拉起运行环境，明显比后面的调用慢。项目里没有做预热，因为四十个人的内部工具，慢那一两秒不影响使用。如果是对响应时间敏感的场景，就得考虑预留实例或者换个部署方式。

## 云数据库

云数据库是文档型的。没有建表语句，没有字段类型约束，一个集合里可以放结构完全不同的文档。项目里的集合有：

| 集合 | 放什么 |
|---|---|
| `users` | 用户、部门、角色、学期、每周空闲时段 |
| `tasks` | 任务详情、指派人、状态、评价 |
| `user_status` | 成员每次上报的状态记录 |
| `locations` | 拍摄机位 |
| `keys` | 一次性权限密钥 |
| `notifications` | 小程序内通知 |
| `news_cache` | 新闻匹配缓存 |

查询长这样：

```js
const { data: tasks } = await db.collection('tasks')
  .where({
    status: 'confirmed',
    startTime: _.gte(now).and(_.lte(twoHoursLater))
  })
  .orderBy('startTime', 'asc')
  .limit(100)
  .get();
```

没有表结构的好处是改字段方便，加一个 `priority`、加一个 `ratings` 数组，不用迁移。坏处是脏数据没人拦。同一个字段，有的文档是字符串，有的是 `Date`，查询时就会出问题。这个项目里时间字段统一存 `Date` 对象，展示时再格式化，就是为了避开这个坑。

数据库权限有四种模式：所有用户可读、仅创建者可读写、所有用户可读仅创建者可写、所有用户不可读写。它只能判断「这条记录是不是你创建的」，判断不了业务角色。所以这个项目的写操作都走云函数，前端不直接写库。这件事在 {% post_link mp-cloudbase-pitfalls %} 里展开讲。

## 云存储

云存储放文件，比如成员上报「不可用」时上传的截图。前端这样传：

```js
const cloudPath = `status-images/${Date.now()}-${Math.random().toString(36).slice(2)}.jpg`;
const res = await wx.cloud.uploadFile({
  cloudPath,
  filePath: tempFilePath
});
// res.fileID 形如 cloud://env-id.xxxx/status-images/xxx.jpg
```

拿到的 `fileID` 可以直接塞进 `<image src="{{fileID}}">`，小程序会自动处理。但它不是永久的 HTTP 链接，如果要给外部用，得调 `wx.cloud.getTempFileURL` 换临时地址，临时地址有有效期。

这个项目里我把 `fileID` 存进 `user_status.images` 数组，展示时直接用 `fileID`，不存临时链接。临时链接过期之后存下来就是死链，这个坑我见别人踩过。

## 一次「确认接受」走了哪几步

拿成员点「确认接受」举例，完整走一遍：

1. 任务详情页调用 `api.updateTask(taskId, { status: 'confirmed' })`。
2. `utils/api.js` 把它包成 `wx.cloud.callFunction({ name: 'updateTask', data: { taskId, updates } })`。
3. 云函数 `updateTask` 用 `OPENID` 查 `users`，拿到当前用户。
4. 查出任务，判断当前用户是不是主任、创建人或被指派人。
5. 检查这次请求是不是只改了 `status`，不是的话非主任会被拒绝。
6. 通过字段白名单拼出更新数据，写回 `tasks`。
7. 返回 `{ code: 0, data: { taskId, updated }, message: '任务更新成功' }`。
8. 前端拿到 `code === 0`，刷新详情页。

创建任务的链路多一步。`createTask` 写完 `tasks` 之后，会异步调 `sendNotification`：

```js
sendTaskNotifications(taskId, taskData.assignedTo, taskData).catch(err => {
  console.error('[createTask] 发送通知失败:', err);
});
```

注意这里是异步的，不等通知结果就返回。任务创建成功与否，不取决于通知有没有发出去。这个设计是对的，通知失败不该让主任重新建一遍任务。

`sendNotification` 现在做的是往 `notifications` 集合写记录，不是发微信订阅消息。前端在 `app.js` 和「我的」页读未读数，给 tab 加红点。这条链路看起来简单，中间换过一次方案，原因写在踩坑那篇里。

## 免费额度和什么时候要钱

云开发有免费额度，覆盖云函数调用次数、数据库读写、存储容量和 CDN 流量。具体数字会变，控制台里能看到当前用量，我不建议背数字。

四十个人的内部工具，正常用很难超出免费额度。真正会涨的是存储：状态上报的截图、机位图片，时间长了会堆。项目里限制了每次最多传三张，就是为了别让它无节制地长。

超出免费额度之后是按量付费。对学生项目来说，先看账单再决定要不要优化，不要一上来就为了省几块钱把架构搞复杂。

## 什么时候云开发不够用

三种情况我会考虑换：

要做 Web 管理端。云开发的前端能力主要面向小程序，Web 端要另接一套鉴权，登录态打不通。

要跑复杂的定时任务或长任务。云函数有超时限制，定时器最小粒度也有限制，跑批量报表、爬大量数据不合适。

要连外部系统。比如接支付回调、接第三方 ERP，云开发的网络和域名限制会比较麻烦。

这个项目目前还没碰到这三条。等真碰到了，再考虑搬到云托管或者自己的服务器。搬的时候云函数里的业务代码大多能复用，主要是把 `cloud.getWXContext()` 换成自己的鉴权，把数据库换成 MySQL 之类的东西。

先写这么多。下一篇写踩过的坑，权限、订阅消息、定时器，都是文档里没写清楚、实际撞上的部分。
