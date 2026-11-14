---
title: 一个任务从发布到验收，中间转了几手
date: 2026-09-05 21:00:00
categories: [微信小程序, 任务派发]
tags: [云开发, 云数据库, 状态机, 权限设计]
description: 任务状态只有五个值，但谁能改、什么时候能改、两个人同时改会怎样，才是真正容易出问题的地方。
comments: true
toc: true
---

标题里的「验收」其实是我后来加的。代码里没有单独的验收状态，最后一步就是成员点「标记完成」，主任事后可以改回来。写这篇文章的时候我翻了 `updateTask` 的代码，发现真正麻烦的不是状态有几种，而是谁能改、在什么情况下能改，以及两个人同时改会发生什么。

## 五个状态

任务状态定义在 `app.js` 的 `taskStatusMap` 里：

```js
taskStatusMap: {
  pending:    { label: '待确认', color: '#FF9800' },
  confirmed:  { label: '已确认', color: '#2196F3' },
  in_progress:{ label: '进行中', color: '#4CAF50' },
  completed:  { label: '已完成', color: '#9E9E9E' },
  cancelled:  { label: '已取消', color: '#F44336' }
}
```

一条任务的正常路径是：主任创建后是 `pending`，被指派的成员点「确认接受」变成 `confirmed`，到现场点「标记开始」变成 `in_progress`，做完点「标记完成」变成 `completed`。主任可以在任何未结束的状态下直接取消，变成 `cancelled`。

还有一条不太起眼的回流：成员在 `in_progress` 时点「申请调整交付状态」，任务会回到 `pending`，等主任重新处理。

## 为什么不用一个 isDone

最开始我想偷懒，给任务加一个 `isDone` 布尔值，做完设 `true`。写了两天就发现不够用。

布尔值只有两个值，但实际需要区分「刚发出去还没人理」「成员已经答应了」「人已经在现场了」「做完了」。这四种情况对主任的意义完全不同：待确认要催人，已确认不用管，进行中要看进度，已完成要统计。一个 `isDone` 表达不了中间态，最后一定会再加一个 `isStarted`、一个 `isAccepted`，变回一堆布尔值。

用字符串枚举之后，前端筛选、统计计数、状态标签颜色都能直接按同一个字段走。`status-badge` 组件只认 `task.status`，不需要判断多个布尔值。

## 谁能改

权限判断在 `updateTask` 里，逻辑不复杂，但值得抄出来看：

```js
const isCreator = task.creatorId === user._id;
const isAssigned = task.assignedTo && task.assignedTo.includes(user._id);
const isDirector = userLevel >= ROLE_LEVEL.photo_director;

const onlyStatusChange =
  Object.keys(updates).length === 1 && 'status' in updates;

if (!isDirector && !isCreator && !(onlyStatusChange && isAssigned)) {
  return Response.permissionDenied('你没有权限修改此任务');
}
```

翻译一下：主任什么都能改；创建者能改；被指派的人只有在「这次请求只改 status 一个字段」时才能改。如果被指派的人想顺手改标题，会被拦下来，提示「仅主任可编辑任务详情」。

这个限制是必要的。成员需要能确认、开始、完成，但不需要能改任务内容。把「能改状态」和「能改详情」拆开，比笼统给一个「能编辑」权限安全得多。

## 那张没被用上的转换表

代码里其实写了一张成员合法转换表：

```js
const allowedMemberStatus = {
  pending_to_confirmed: '确认接受',
  confirmed_to_in_progress: '开始任务',
  in_progress_to_completed: '完成任务',
  completed_to_pending: '申请调整'
};
```

看起来挺像回事，问题是它后面没有被任何地方引用。真正的校验只做了「非主任不能改状态以外的字段」，没有校验「你改的这个状态是不是从当前状态合法走过来的」。

也就是说，一个被指派的成员如果绕过前端，直接调 `updateTask` 把 `pending` 改成 `completed`，云函数会放行。前端之所以看起来正常，是因为任务详情页按状态藏了按钮：`pending` 只显示「确认接受」，`confirmed` 只显示「开始」，`in_progress` 才显示「完成」。

前端藏按钮不是权限。只要接口能被直接调用，服务端就必须自己校验。这是我现在最想补的一块，补起来也不难：

```js
const TRANSITIONS = {
  pending: ['confirmed', 'cancelled'],
  confirmed: ['in_progress', 'cancelled'],
  in_progress: ['completed', 'pending', 'cancelled'],
  completed: ['pending'],
  cancelled: []
};

const allowed = TRANSITIONS[task.status] || [];
if (!allowed.includes(updates.status)) {
  return Response.paramError('当前状态不支持这个操作');
}
```

这段现在还没进代码，先记在这儿。

## 数据库权限规则管不了业务

云开发的数据库权限有四种模式：所有用户可读、仅创建者可读写、所有用户可读仅创建者可写、所有用户不可读写。听起来够用，实际只能回答一个问题：这条记录是不是你创建的。

任务派发的场景比这个复杂。一条任务由主任创建，指派给成员；成员要能读，还要能改状态，但不能改标题；其他部门的成员不该看到；中心主任要能看到全部。这些规则没有一条能只靠「记录是不是你创建的」表达出来。

所以这个项目的写操作全部放在云函数里。云函数用的是管理员权限，不受数据库权限规则限制，判断逻辑写在代码里。前端只调 `wx.cloud.callFunction`，不直接碰 `db.collection`。

代价是每加一个操作就要多写一个云函数分支，不能像前端直写那样一行 `add` 解决。但权限规则至少是集中的，不会散在十几个页面里。

## 两个人同时点会怎样

现在的代码是先读任务、再判断权限、最后更新：

```js
const { data: tasks } = await tasksCollection.doc(taskId).get();
const task = tasks.length ? tasks[0] : tasks;

// ...权限判断...

await tasksCollection.doc(taskId).update({ data: updateData });
```

这个写法在大多数时候没问题，因为它默认「读和写之间不会有人插进来」。实际上会。

举两个真实会发生的场景。一个是主任点「取消」的同时，成员点「标记开始」：主任先读到 `confirmed`，成员也读到 `confirmed`，然后两个 update 先后落下，谁后写谁赢。结果可能是任务已经取消了，状态却又变成 `in_progress`。

另一个是定时器。`scheduleNotify` 每 30 分钟扫一次，把 `endTime` 已过且状态是 `in_progress` 的任务自动置为 `completed`。如果成员刚好在同一时间点「标记完成」，两边都写 `completed`，状态没错，但通知会发两遍。

要修的话，最直接的是条件更新：

```js
const expectedStatus = task.status;
const res = await tasksCollection
  .where({ _id: taskId, status: expectedStatus })
  .update({
    data: { status: updates.status, updatedAt: new Date() }
  });

if (res.stats.updated === 0) {
  return Response.paramError('任务状态已经变了，请刷新后重试');
}
```

`where` 里带上旧状态，只有状态没变时才更新。更新影响行数为 0，说明有人先改了，让前端刷新重来。这个做法不需要事务，适合云开发基础版。

事务也可以，但 `verifyKey` 里留过一句注释：小程序云开发基础版不支持事务，所以那个函数用了两步更新。状态流转这块我倾向于条件更新，简单，够用。

## 通知和状态是绑在一起的

`createTask` 写完任务后，会异步调 `sendNotification` 给被指派的人写一条通知。`updateTask` 只在状态变成 `cancelled` 时通知。定时器负责两件事：任务开始前两小时提醒状态为 `confirmed` 的任务，以及把过期的 `in_progress` 任务自动完成。

这里有个细节：提醒只扫 `confirmed`，不扫 `in_progress`。如果成员一直没点「开始」，但人已经去了，提醒照样会发；如果他点了「开始」，反而不会收到提醒。这个逻辑当时没想清楚，后来也没改，因为实际使用中大家一般不会提前点开始。

订阅消息的限制、定时器的时间窗，这些放在 {% post_link mp-cloudbase-pitfalls %} 里讲。状态机本身和通知机制是两件事，但它们的边界经常缠在一起，改一个要顺带看另一个。

## 现在回头看

要重写的排序大概是：

第一，把转换表真正接进 `updateTask`，服务端校验合法状态流转。现在这张表只是注释级别的存在。

第二，把「申请调整」的原因存下来。前端提交了 `adjustReason`，但 `updateTask` 的字段白名单里没有这个字段，更新时被直接丢掉了。主任只能看到任务回到 `pending`，看不到成员为什么申请调整。

第三，把状态更新改成条件更新，处理并发。现在两个人同时操作，结果是随机的。

第四，给通知加去重键。同一个任务、同一个用户、同一种类型，不应该重复写。

这四条都不难，难的是当时没意识到它们是问题。第一版能跑起来，靠的是前端按钮藏得好和用户量小。用户一多、网络一慢，这些地方就会开始出毛病。
