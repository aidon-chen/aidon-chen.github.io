---
title: 微信开发者工具我用得最顺的几处，顺便聊两个脚本
date: 2026-08-10 20:00:00
categories: [微信小程序, 工具链]
tags: [微信开发者工具, 小程序原生, 自动化, 环境搭建]
description: 用了两个月开发者工具后留下来的几个固定操作，以及两个把重复劳动压掉的脚本。
comments: true
toc: true
---

六月底开始写视觉中心那个任务派发小程序，这是我第一次正经做小程序。之前写过网页和 Python 脚本，进到开发者工具里还是有点懵：项目结构、AppID、云函数目录，每一样都要先弄明白。用了两个月，工具本身没多复杂，真正省时间的是几个固定习惯，还有两个后来补的脚本。

## 先把项目结构认清楚

我导入的是仓库根目录，因为这个项目没有单独的 `miniprogram/` 子目录。判断方法很简单：`app.js`、`app.json`、`project.config.json` 在哪一层，开发者工具就导入哪一层。选错一层，最常见的报错是「app.json 未找到」。

`project.config.json` 里有三个字段值得先看：

```json
{
  "appid": "wx8fdb0c0acb28ee1a",
  "cloudfunctionRoot": "cloudfunctions/",
  "libVersion": "2.32.3"
}
```

`appid` 别用测试号。云开发环境是绑在正式 AppID 下的，测试号连不上你账号里的环境。`cloudfunctionRoot` 决定了哪个目录会被识别成云函数目录，右键上传时能不能看到「上传并部署」全靠它。基础库版本我用的是 2.32.3，因为隐私协议那套接口在这个版本之后才稳定。

前端用了 TDesign 小程序组件库，所以第一次跑之前必须做一次「工具 → 构建 npm」。这一步不做，编译会报 `tdesign-xxx 未找到`，而且报错位置通常在 wxml 里，看起来像是组件名字写错了，容易绕远路。后来只要 `package.json` 动过，或者拉了别人的代码，我都会重新构建一次。

## 模拟器、真机和云开发控制台

日常改页面，模拟器够用了。它的好处是改完重新编译很快，样式马上能看到。但有三类问题模拟器看不出来：

第一类是字体和布局。项目后来把原生导航栏换成了自定义导航栏，模拟器上一切正常，真机上华为把系统字体调大之后，标题直接溢出。这个问题是看真机截图才发现的，后来单独改了自定义导航栏的标题布局。

第二类是授权。位置、相册、隐私协议这些弹窗，模拟器和真机的表现不一样。真机预览扫一次码，比在模拟器里猜半天靠谱。

第三类是云开发的真实身份。云函数里拿的是 `cloud.getWXContext().OPENID`，模拟器里也有 openid，但换到真机上才是真实用户。权限相关的改动，我一般会在真机上再走一遍。

调试云函数时，我不太看 Network 面板。`wx.cloud.callFunction` 的调用记录在云开发控制台的「云函数 → 日志」里更清楚，入参、返回、报错都在那儿。Network 面板我主要用来看静态资源和普通的 `wx.request`。

Storage 面板倒是用得很多。这个项目有两个键经常要手动改：

- `td_envOverride`：本地临时切云环境，值写 `dev`、`test` 或 `prod`；
- `td_taskFilter`：首页筛选条件的持久化，调筛选逻辑时经常要先清掉它，不然看到的是上次的筛选结果。

改完云环境或者清完缓存，记得重新编译一次。开发者工具偶尔会出现样式没刷新、页面还是旧代码的情况，我遇到的几次都是「清缓存 → 全部清除」之后恢复的。

## 两个脚本

项目里有两类重复劳动：切测试开关、部署云函数。前者是十几个文件里的 `enabled` 字段，后者是十几个云函数目录，手动点右键能点到手酸。

第一个脚本是 `scripts/switch-mode.js`。它做两件事：把 13 个开关文件里的 `enabled` 改成目标值，然后调开发者工具 CLI 部署 12 个带 `testConfig.js` 的云函数。用法很直白：

```bash
node scripts/switch-mode.js test
node scripts/switch-mode.js release
node scripts/switch-mode.js test --no-deploy
```

测试模式和发布模式的差别不只是前端一个布尔值。云函数端也有各自的 `testConfig.js`，前端把 `enabled` 打开了、云端没打开，调试角色照样提不了权；反过来云端开着、前端关了，测试入口又不会显示。所以这个脚本必须把两边一起改。

写的时候踩了一个很蠢的坑。`config/test.js` 的注释里也出现了 `enabled: xxx` 的字样，最初的正则取了第一个匹配，结果改的是注释，真正的字段没动。后来改成取最后一个匹配：

```js
const matches = [...content.matchAll(/enabled:\s*(true|false)/g)];
const last = matches[matches.length - 1];
```

这件事之后我对「正则改配置文件」一直有点警惕。能不用正则就不用，非要用的话，先打印出准备改哪一段，再执行。

第二个脚本是 `scripts/deploy-functions.js`。它按环境部署云函数，默认走 CloudBase CLI：

```bash
node scripts/deploy-functions.js --env test
node scripts/deploy-functions.js --env test --name createTask
node scripts/deploy-functions.js --env test --dry-run
```

脚本强制要求传 `--env`，不给默认值。这是故意的：云开发项目最容易出的事故就是把测试函数部署到生产环境，或者反过来。多个环境共用一套代码时，「默认部署到哪」这个问题不该有默认答案。

它也支持用环境变量覆盖部署命令：

```bash
TD_DEPLOY_CMD='"C:/Program Files (x86)/Tencent/微信web开发者工具/cli.bat" cloud functions:deploy --name {name} --env {envId}' \
node scripts/deploy-functions.js --env test
```

开发者工具 CLI 本身还能开项目、自动预览，不过我用得最多的是云函数部署。`--remote-npm-install` 这个参数挺关键，它让云端自己装依赖，不用把 `node_modules` 一起传上去，部署体积小很多。

## 几个当时卡了半天的坑

`Env Not Exists (-501000)`。这个报错一般不是代码问题，是 `config/env.js` 里的环境 ID 写错了，或者那个环境不在你账号下。项目早期三个环境 ID 都是占位符，后来统一改成真实 ID 才好。遇到这个错，先去云开发控制台确认环境名，再回头对配置。

`tdesign-xxx 未找到`。前面说过，构建 npm。补充一点：如果构建完还是找不到，先清缓存再构建一次，顺序反了有时候不生效。

测试开关忘了关。发布前最怕某次调试把 `enabled: true` 提交上去。这个项目后来在发布清单里专门列了一条：检查 `config/test.js` 和 12 个云函数 `testConfig.js`。脚本能降低遗漏概率，但不能替代最后那一眼。

真机白屏。我遇到过一次，重新编译、重新扫码就好了，没定位到具体原因。后来养成的习惯是：真机预览前先确认云函数已经部署，再确认当前环境是不是 `dev`，这两个都没问题再怀疑代码。

## 最后

开发者工具本身的功能不算多，真正影响效率的是把重复操作固定成脚本。后面给湖里 Lakes 写的 `start-all.sh`，思路是一样的：把「先起这个、再起那个」的命令收进一个文件，下次不用回忆。

任务派发项目本身也写了一篇，讲为什么做、怎么收需求：{% post_link mp-task-dispatch-why %}。
