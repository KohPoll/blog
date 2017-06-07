title: Node.js 命令行工具检查更新的正确姿势
date: 2017-06-06
tags:
---

随着 Node.js 的“走红”，使用 Node.js 开发命令行工具越来越简单。一个成熟的命令行工具应该从一开始就要考虑好之后的版本更新如何“优雅”的告知用户。最好的方法当然是当用户在终端执行命令时，将相关信息提示给用户。

这篇文章将给出一个易用、高效、可定制的方法。源码在这里：<https://github.com/kohpoll/pkg-updater>，欢迎大家顺手点赞。接下来我将讲解其实现思路。

<!-- more -->

## 使用
我们先简单看看这个 npm 包的使用方法：

```javascript
const updater = require('pkg-updater');
const pkg = require('./package.json'); // 命令行工具自己的 package 信息

updater({'pkg': pkg}) .then(() => { /* 在这里启动命令行工具 */ });

updater({
  'pkg': pkg,  
  // 自定义 registry
  'registry': 'http://xxx.registry.com',
  // 自定义请求的 dist-tag，默认是 latest
  'tag': 'next',
  // 自定义检查间隔，默认是 1h
  'checkInterval': 24 * 60 * 60 * 1000,
  // 自定义更新提示信息
  'updateMessage': 'package update from <%=current%> to <%=latest%>.',
  // 自定义强制更新的版本更新级别，默认是 major
  'level': 'minor'
}).then(() => { /* 在这里启动命令行工具 */ });

updater({
  'pkg': pkg,
  // 完全自定义版本更新时的逻辑
  'onVersionChange': function* (opts) {

  }
}).then(() => { /* 在这里启动命令行工具 */ });
```

效果如图：

![兼容版本更新](https://img.alicdn.com/tfs/TB1d.GdRFXXXXbSXFXXXXXXXXXX-752-244.png)

![不兼容版本更新](https://img.alicdn.com/tfs/TB1YA49RFXXXXcIXFXXXXXXXXXX-1120-298.png)

## 实现
使用方法很简单，我们一起来看看其实现方法。

### 需求
我们先来梳理下需求，一个命令行检查更新器应该至少提供如下功能：
  - 能从远程获取最新版本
  - 能根据检查结果进行提示
  - 在版本不兼容时可以直接退出，强制用户升级程序

### 获取版本
获取最新版本这个功能看起来很简单，就是发送一个请求从“某处”获取信息。但是有一些问题需要我们考虑：
  - 从哪里获取版本信息？
  - 获取版本信息的策略是怎样的？（什么时候获取？获取的信息如何处理？）

#### 从哪里获取版本信息
我们的命令行工具一般都是使用 npm 进行分发，最简便的方法就是直接通过 registry 获取。通过请求 `https://registry.npmjs.org/{name}/{dist-tag}` 就可以得到 package 对应 tag 的版本信息。结果类似下面这样：

```json
// https://registry.npmjs.org/co/latest
{
  "name": "co",
  "version": "4.5.0"
}
```

在实际实现时，我们应该允许调用者自定义 registry 地址、请求的 dist-tag 等，这样可以有更多的定制性。

#### 获取版本信息的策略
首先想到的方法是用户每次执行命令时都去获取一次版本信息，这样的获取策略应该是最简单和实时的。

但是这个策略其实并不合适：
  - 每次执行命令都要去发请求进行检查，如果网络延迟，会阻塞命令执行，影响用户体验
  - 工具的版本更新其实并不会很频繁，没有必要进行实时检查
  - 网络请求的影响因素很多，不能保证每次都成功，应该提供本地缓存机制来存储请求成功的结果，避免版本信息的不可用

综合上面的几点，我们设计如下的获取策略：
  - 将发送网络请求获取版本信息的逻辑放在一个独立的后台进程去执行，保证不阻塞主命令执行
  - 请求成功后将版本信息、检查时间缓存到用户机器
  - 每次执行命令时，只是读取本地缓存下来的版本信息，不去发送网络请求
  - 根据缓存下来的检查时间和当前时间，在一个间隔之内不去额外创建后台检查进程

将上面的策略翻译成代码大概就是下面这样：

```javascript
// 读取本地缓存的检查结果
const checkInfo = yield updater.readCheckInfo(opts);
const lastCheck = checkInfo.lastCheck;
const lastVersion = checkInfo.lastVersion;

// 根据版本信息提示用户
// ...

// 在时间间隔内，直接返回
if (Date.now() - lastCheck < opts.checkInterval) {
  return;
}

// 创建后台检查进程
try {
  require('child_process').spawn(
    process.execPath,
    [require('path').join(__dirname, '_check.js'), JSON.stringify({
      'pkg': opts.pkg, // package 信息
      'tag': opts.tag, // 检查的 dist-tag
      'logFile': opts.logFile, // 缓存文件路径
      'registry': opts.registry // registry 地址
    })],
    {'stdio': ['ignore', 'ignore', 'ignore'], 'detached': true}
  ).unref();
} catch(e) {}
```

后台进程执行的 `_check.js` 文件也很简单，如下所示：

```javascript
const opts = JSON.parse(process.argv[2]);

let lastVersion = '';
try {
  // 发送请求获取最新版本
  const url = normalizeUrl(opts.registry + '/' + opts.pkg.name + '/' + (opts.tag || 'latest'));
  const res = yield got.get(url, {
    'json': true,
    'timeout': 60 * 1000
  });
  if (res && res.body && res.body.version) {
    lastVersion = res.body.version;
  }
} catch(e) {}

// 如果获取失败了，最新版本就是当前版本（package.version）
if (!lastVersion) {
  lastVersion = opts.pkg.version;
}

let data = yield util.readJson(opts.logFile);
if (!data[opts.pkg.name]) {
  data[opts.pkg.name] = {};
}
data[opts.pkg.name].lastVersion = lastVersion; // 最新版本
data[opts.pkg.name].lastCheck = Date.now();    // 检查时间

// 写入缓存
yield util.writeJson(opts.logFile, data);
```

### 提示
当版本更新了，我们应该在终端提示用户。这里有两个问题：
  - 提示文案的问题
  - 提示文案显示间隔的问题（一直显示？每隔一段时间显示？）

这里我们采取的策略是：
  - 提供默认提示文案，清晰的说明当前版本、最新版本、更新方法，允许调用者自定义提示文案
  - 只要有更新就一直显示提示文案，因为我们希望用户经常的进行更新

实现代码大概如下：

```javascript
// 比对版本
const type = updater.diffType(opts.pkg.version, lastVersion, opts.level);
if (type) {
  // 根据模板渲染提示信息
  const str = updater.template(opts.updateMessage || updater.defaultOpts.updateMessage)({
    'colors': updater.colors,
    'name': opts.pkg.name,
    'current': opts.pkg.version,
    'latest': opts.lastVersion,
    'command': 'npm i -g ' + opts.pkg.name
  });
  // 进行提示
  console.log(
    updater.boxen(str, {
      'padding': 1,
      'margin': 1,
      'borderStyle': 'classic'
    })
  );
}
```

### 强制升级
对于 npm 模块来说，版本 a.b.c 的更新一般有三种情况：
  - patch：c 位，小版本更新，一般是 bug 修复
  - minor：b 位，中版本更新，一般增加新功能、bug 修复
  - major，a 位，大版本更新，一般是不兼容的升级

我们希望当远程版本的更新如果是 major 形式，命令行工具将直接退出，强制用户进行升级后才能使用。这可以保证我们推送一个大版本后，所有的用户都能够马上更新掉，而不是继续使用老版本，造成版本碎片的问题。

实现代码大致如下：

```javascript
// 比对版本
const type = updater.diffType(opts.pkg.version, lastVersion, opts.level);
if (type) {
  // 根据模板渲染提示信息
  const str = updater.template(opts.updateMessage || updater.defaultOpts.updateMessage)({
    'colors': updater.colors,
    'name': opts.pkg.name,
    'current': opts.pkg.version,
    'latest': opts.lastVersion,
    'command': 'npm i -g ' + opts.pkg.name
  });
  // 进行提示
  console.log(
    updater.boxen(str, {
      'padding': 1,
      'margin': 1,
      'borderStyle': 'classic'
    })
  );

  // 不兼容的更新，直接让进程退出
  if (type == 'incompatible') {
    process.exit(1);
  }
}
```

## 总结
命令行检查更新看似简单，其实仔细思考，还是有很多细节。希望这篇文章对你有所启发。
