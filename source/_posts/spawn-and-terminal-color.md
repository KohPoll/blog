title: 在 spawn 的子进程中保持命令行颜色
date: 2016-09-15
tags:
---

最近在用子进程运行 gulpfile.js 的时候发现终端上的颜色全部没有了，很是奇怪。经过一些研究，最终解决了问题，同时也总结了一些相关知识。希望对你有帮助。

<!-- more -->

## 终端颜色

相信大家都知道，我们平常使用的 terminal 是可以输出各种颜色的，并不仅仅只是充满 geek 味道的黑绿。这些颜色其实是通过一种叫做 `ANSI/VT100` 控制序列来标记的，这些字符本身不可见，会被 shell 解析后使用合适的颜色来渲染。

比如，命令 `echo -e "\033[31m red \033[39m"` 就可以在终端打印红色的文字。

其中的 `\033[31m`、`\033[39m` 就是特殊的控制序列，`\033[31m` 表示红色的前景（文字）色，`\033[39m` 表示默认的前景（文字）色。而选项 `-e` 则是让 `echo` 启用反斜杠控制字符的转换（默认是不转换的）。

实际上，我们还可以使用 `echo -e "\x1b[31m red \x1b[39m"` 来输出红色文字。`\x1b` 是十六进制表示，刚好等于八进制表示的 `033`。

除了前景色，还可以通过序列来表示背景色。更多的序列，可以参考：<http://misc.flogisoft.com/bash/tip_colors_and_formatting>。

那在 Node 中我们该如何打印带颜色的内容呢？其实很简单，我们只要使用一样的控制序列就可以了。

比如下面的代码，都可以打印红色文字。

```javascript
console.log('\u001b[31m red \u001b[39m');
console.log('\033[31m red \033[39m'); // 这行代码在 strict 模式下将会报错
console.log('\x1b[31m red \x1b[39m');
```

但是这么多的控制序列很难记住，写起来很麻烦，给别人看更是如天书一般难懂。我推荐直接使用 [chalk](https://github.com/chalk/chalk) 这样的 npm 包，代码瞬间简洁、清晰了许多：

```javascript
const chalk = require('chalk');
console.log(chalk.red('red'));
```

`chalk` 实际上直接使用了 [ansi-styles](https://github.com/chalk/ansi-styles) 这个包，从其源码中可以看到，原理就是对字符串增加特殊的控制序列：<https://github.com/chalk/ansi-styles/blob/master/index.js>

## spawn

我在之前的[文章](http://kohpoll.github.io/blog/2016/04/25/about-the-node-child-process/)中介绍过 Node 子进程的用法，感兴趣的可以进行阅读。

我比较喜欢使用 `spawn` 的方式，因为它可以通过 stream 的方式操作子进程的输出，非常方便。比如下面的代码：

```javascript
const log = [];
const path = require('path');
const child = require('child_process').spawn(
  'node',
  [
    require.resolve('gulp/bin/gulp'),
    '--gulpfile', path.join(__dirname, 'gulpfile.js'),
    '--cwd', process.cwd()  
  ],
  {
    'stdio': 'pipe',
    'cwd': process.cwd()
  }
);
child.stdout.pipe(process.stdout);
child.stdout.on('data', (data) => {
  // collect the data
  log.push(data.toString('utf8'))
});
```

这里指定 `stdio: 'pipe'` 后，我们可以操作子进程的 `stdout`，比如将它的输出收集起来然后写入数据库做记录之类的。

如果尝试运行上面的脚本会发现，之前直接通过 `gulp` 执行 `gulpfile.js` 时漂亮的颜色消失了，全部变成默认颜色了。

## process.stdout.isTTY

实际上，在 Node 中执行的进程我们可以通过 `process.stdout.isTTY` 这个属性来判断它是否在终端（terminal）终端环境中执行。

通过 `spawn` 并配置了 `stdio: 'pipe'` 开启的子进程，`process.stdout.isTTY` 属性会是 `undefined`。

比如下面的代码：

```javascript
// parent.js
const child = require('child_process').spawn(
  'node', ['./child.js'],
  {'stdio': 'pipe'}
);
child.stdout.pipe(process.stdout);

// child.js
console.log('process.stdout.isTTY=', process.stdout.isTTY);
```

在终端执行 `node parent.js` 会输出 `process.stdout.isTTY= undefined`。

单独执行 `node child.js` 会输出 `process.stdout.isTTY= true`。

`gulp` 背后的颜色功能，实际上使用的是 `chalk`。

`chalk` 会使用 [supports-color](https://github.com/chalk/supports-color) 做是否支持终端颜色的判断。从其源码中我们看到，它正是通过 `process.stdout.isTTY` 来进行判断的，如果该属性不为 `true`，则认为不支持：<https://github.com/chalk/supports-color/blob/master/index.js#L41>

因此，由于被认为是在不支持终端颜色的环境中执行，我们之前代码中的所有输出就都不再带有颜色了。

另外我们还看到，`supports-color` 还会检查命令行参数中是否有 `--color` 选项，如果有的话就直接启用，不再检查 `proess.stdout.isTTY`。这正是我要的解决方案，于是，将代码修改成如下后，问题得到解决：

```javascript
const log = [];
const path = require('path');
const child = require('child_process').spawn(
  'node',
  [
    require.resolve('gulp/bin/gulp'),
    '--gulpfile', path.join(__dirname, 'gulpfile.js'),
    '--cwd', process.cwd(),
    '--color' // preserve the terminal color  
  ],
  {
    'stdio': 'pipe',
    'cwd': process.cwd()
  }
);
child.stdout.pipe(process.stdout);
child.stdout.on('data', (data) => {
  // collect the data
  log.push(data.toString('utf8'))
});
```

## stdio: inherit

实际上，我们也可以指定 `stdio: 'inherit'` 来让子进程直接使用父进程的 IO。这时子进程的 `process.stdout.isTTY` 会是 `true`。这是因为直接使用了父进程的 IO，所以它实际上还是在终端环境中执行的。

但是，这个方法会导致子进程不再拥有 `stdout` 属性，无法通过代码获取到子进程的任何标准输出，所以我最终并没有采用。

## 参考资料
  - http://misc.flogisoft.com/bash/tip_colors_and_formatting
  - https://github.com/chalk/chalk
  - https://nodejs.org/dist/latest-v4.x/docs/api/process.html#process_process_stdout
  - https://nodejs.org/dist/latest-v4.x/docs/api/child_process.html#child_process_options_stdio
  - https://nodejs.org/dist/latest-v4.x/docs/api/child_process.html#child_process_child_stdout

本文采用 [知识共享署名 3.0 中国大陆许可协议](http://creativecommons.org/licenses/by/3.0/cn)，可自由转载、引用，但需署名作者且注明文章出处 。