title: 如何优雅的在 koa 中处理错误
date: 2016-03-15
tags:
---

> 软件开发时，有 80% 的代码在处理各种错误。
>  <p style="text-align:right">——某著名开发者</p>

想让自己的代码健壮，错误处理是必不可少的。这篇文章将主要介绍 koa 框架中错误处理的实现（其实主要是 co 的实现），使用 koa 框架开发 web 应用时进行错误处理的一些方法。

<!-- more -->

## 基础

在 Node 中，错误处理的方法主要有下面几种：
  - 和其他同步语言类似的 throw / try / catch 方法
  - callback(err, data) 回调形式
  - 通过 EventEmitter 触发一个 error 事件

第一种使用 catch 来捕获错误，十分易用，其他两种在捕获错误时多多少少都有些别扭。

但是 koa 通过十分巧妙的”黑魔法“让我们可以使用 catch 来捕获异步代码中的错误。比如下面的例子：

```javascript
const fs = require('fs');
const Promise = require('bluebird');

let filename = '/nonexists';
let statAsync = Promise.promisify(fs.stat);
try {
  yield statAsync(filename);
} catch(e) {
  // error here
}
```

在 koa 中，推荐统一使用 throw / try / catch 的方式来进行错误的触发和捕获，这会让代码更加易读，防止被绕晕。

## 原理

上面我们说了 koa 中可以使用 try / catch，我们就来分析下它是如何做到的。koa 基于 co，所以，我们其实主要是分析 co 的实现。（注：这一部分比较偏原理，不关心的可以跳过。）

首先，我们来看看什么是 generator。

```javascript
function* gen() {
  var a = yield 'start';
  console.log(a);
  var b = yield 'end';
  console.log(b);
  return 'over';
}
var it = gen();
console.log(it.next()); // {value: 'start', done: false}
console.log(it.next(22)); // 22 {value: 'end', done: false}
console.log(it.next(333)); // 333 {value: 'over', done: true}
```

带有 `*` 的函数声明表示是一个 generator 函数，当执行 `gen()` 时，函数体内的代码并没有执行，而是返回了一个 generator 对象。

generator 函数通常和 yield 结合使用，函数执行到每个 yield 时都会暂停并返回 yield 的右值。下次调用 next 时，函数会从 yield 的下一个语句继续执行。等到整个函数执行完，next 方法返回的 done 字段会变成 true，并且将函数返回值作为 value 字段。

第一次执行 `next()` 时，走到 `yield 'start'` 后暂停并返回 `yield` 的右值 `'start'`。注意，此时`var a = ` 这个赋值语句其实还没有执行。

第二次执行 `next(22)` 时，从 `yield 'start'` 下一个语句执行。于是执行 `var a = ` 这个赋值语句，而表达式 `yield 'start'` 的值就等于传递给 `next` 函数的参数值 `22`，所以，`a` 被赋值为 `22`。然后继续往下执行到 `yield 'end'` 后暂停并返回 `yield` 的右值 `'end'`。

第三次执行 `next(333)` 时，从 `yield 'end'` 下一个语句执行。此时执行 `var b = ` 这个赋值语句，表达式 `yield 'end'` 的值等于传递给 `next` 函数的参数 `333`，`b` 被赋值为 `333`。继续往下执行到 `return` 语句，将 `return` 语句的返回值作为 `value` 返回，因为函数已经执行完毕，`done` 字段标记为 `true`。

可以看到 generator 就是一种迭代机制，就像一只很懒的青蛙，戳一下（调用 `next`）动一下。

generator 对象还有一个 `throw` 方法，可以在 generator 函数外面抛出异常，然后在 generator 函数里面捕获异常。有点绕？我们来看一个实例：

```javascript
function *gen() {
  try {
    yield 'a';
    yield 'b';
  } catch(e) {
    console.log('inside：', e); // inside： [Error: error from outside]
  }
}

var it = gen();
it.next();

console.log(it.throw(new Error('error from outside'))); // { value: undefined, done: true }
```

我们执行一次 `next`，会运行到 `yield 'a'` 这里然后暂停，这一句刚好在 try 的返回内，因此 `it.throw` 抛出的错误我们可以 catch 到。并且看到 `throw` 返回的 `done` 字段是 `true`，说明后面的 `yield 'b'` 已经不会再执行了。

如果我们不调用 `next`，或者连续调用三次 `next`，`yield` 代码不在 `try` 返回里面，会导致报错。co 的错误处理其实正是利用了这个 `throw` 方法。

下面我们来看看 co 的核心代码：

```javascript
function co(gen) {
  var ctx = this;
  var args = slice.call(arguments, 1);

  // 统一返回一个整体的 promise
  return new Promise(function(resolve, reject) {
    // 如果是函数，调用并取得 generator 对象
    if (typeof gen === 'function') gen = gen.apply(ctx, args);
    // 如果根本不是 generator 对象（没有 next 方法），直接 resolve 掉并返回
    if (!gen || typeof gen.next !== 'function') return resolve(gen);

    // 入口函数
    onFulfilled();

    function onFulfilled(res) {
      var ret;
      try {
        // 拿到 yield 的返回值
        ret = gen.next(res);
      } catch (e) {
        // 如果执行发生错误，直接将 promise reject 掉
        return reject(e);
      }
      // 延续调用链
      next(ret);
    }
    function onRejected(err) {
      var ret;
      try {
        // 如果 promise 被 reject 了就直接抛出错误
        ret = gen.throw(err);
      } catch (e) {
        // 如果执行发生错误，直接将 promise reject 掉
        return reject(e);
      }
      // 延续调用链
      next(ret);
    }
    function next(ret) {
      // generator 函数执行完毕，resolve 掉 promise
      if (ret.done) return resolve(ret.value);
      // 将 value 统一转换为 promise
      var value = toPromise.call(ctx, ret.value);
      // 将 promise 添加 onFulfilled、onRejected，这样当新的promise 状态变成成功或失败，就会调用对应的回调。整个 next 链路就执行下去了
      if (value && isPromise(value)) return value.then(onFulfilled, onRejected);
      // 没法转换为 promise，直接 reject 掉 promise
      return onRejected(new TypeError('You may only yield a function, promise, generator, array, or object, '
        + 'but the following object was passed: "' + String(ret.value) + '"'));
    }
  });
}
```

假设有下面的代码，让我们一起推演下执行流程：

```javascript
co(function* gen() {
  var a = yield Promise.resolve('a 值');
  console.log(a);
  try {
    var b = yield Promise.reject(new Error('b 错误'));
    var c = yield Promise.resolve('c 值');
    console.log(b, c);
  } catch(e) {
    console.log('error', e);
  }
  return 'over';
}).then(function (value) {
  console.log(value);
}).catch(function (err) {
  console.error(err.stack);
});
```

约定：`Promise.resolve('a 值')` 生成的是 promiseA；`Promise.reject(new Error('b 错误'))` 生成的是 promiseB。

首先传入 co 的 gen 函数会被执行，获取到 generator 对象。对应代码：`if (typeof gen === 'function') gen = gen.apply(ctx, args);`。

然后调用 `onFulfilled` 函数。开启整个执行过程。

第一次执行 `ret = gen.next(res)`，走到 `yield Promise.resolve('a 值')` 后暂停并返回 `yield` 的右值，此时 `ret` 等于 `{value: PromiseA, done: false}`。

然后执行 `next(ret)`，将 `ret.value` 转换为 Promise，执行 `value.then(onFulfilled, onRejected)`，也就是 `PromiseA.then(onFulfilled, onRejected)`。当我们的 PromiseA 被 resolve 后，又再次执行 `onFulfilled`，并传入 resvole 的值，也就是：`onFulfilled('a 值')`。

于是第二次执行 `ret = gen.next('a 值')`（此时的 `res` 就等于 `a 值`），进入到 gen 函数，执行接下来的 `var a = ` 赋值语句，`yield Promise.resolve('a 值')` 的返回值等于给 `next` 传递的参数 `'a 值'`，于是变量 `a` 被赋值为 `'a 值'`。继续执行到 `yield Promise.reject(new Error('b 错误'))` 后暂停并返回 `yield` 的右值，此时 `ret` 等于 `{value: PromiseB, done: false}`。

继续执行 `next(ret)`，延续调用链。执行 `value.then(onFulfilled, onRejected)`，也就是 `PromiseB.then(onFulfilled, onRejected)`。这次 PromiseB 被 reject 掉了，于是执行 `onRejected`，并传人 reject 的错误原因，也就是：`onRejected(new Error('b 错误'))`。

于是执行到 `ret = gen.throw(new Error('b 错误'))`，而此时 `yield Promise.reject(new Error('b 错误'))` 刚好在 try 的范围内，错误被 catch 住了！接着就执行 catch 里面的打印语句 `console.log('error', e);`，一路执行到函数结束（因为再也没有 `yield` 了），将返回值赋给 `value`。最后 `ret` 等于 `{value: 'over', done: true}`。

继续执行 `next(ret)`，延续调用链。执行到 `if (ret.done) return resolve(ret.value);`，于是整体的 promise 被 resolve 掉，执行 `then` 里面的打印语句，打印出 `ret.value` 的值 `'over'`。整个流程结束。

如果我们不 try / catch 会怎样？因为 `onRejected` 里面有是这样处理的：`try { ret = gen.throw(err); } catch (e) { return reject(e); }`。我们上面说如果 `yield` 没有在 `try`里会导致 `gen.throw` 报错，于是整体 promise 被 reject，执行其 `catch` 方法，打印出 `Error('b 错误')` 的堆栈。

这就是“黑魔法”的神秘面纱！对 TJ 大神真是一个大写的“服”字。

## 什么错误该处理和怎么处理

接下来的问题是什么样的错误我们需要处理？怎么处理？我们可以将错误分个类：
  - 操作错误：不是程序 bug 导致的运行时错误。比如：连接数据库服务器失败、请求接口超时、系统内存用光等等。
  - 程序错误：程序 bug 导致的错误，只要修改代码就可以避免。比如：尝试读取未定义对象的属性、语法错误等等。

很显然，我们真正需要处理的是操作错误，程序错误应该马上进行修复。

那怎么处理操作错误呢？总结起来大概有下面这些方法：
  - 直接处理。这个简直是废话。举个例子：尝试向一个文件中写东西，但是这个文件不存在，那这个时候会报错吧？处理这个错误的方法就是先创建好要写入的文件。如果我们知道怎么处理错误，那直接处理就是。
  - 重试。有时候某些错误可能是偶发的（比如：连接的服务不稳定等），我们可以尝试对当前操作进行重试。但是一定要设置重试的超时时间、次数，避免长时间的等待卡死应用。
  - 直接将错误抛给调用方。如果我们不知道具体怎么处理错误，那最简单的就是将错误往上抛。比如：检查到用户没有权限访问某个资源，那我们直接 throw 一个 Error（并带上 status 是 403）比较好，上层代码可以 catch 这个错误，然后要么展示一个统一的无权限页面给用户，要么返回一个统一的错误 json 给调用方。
  - 写日志然后将错误抛出。这种情况一般是发生了比较致命的错误，没法处理，也不能重试，那我们需要记下错误日志（方便以后定位问题），然后将错误往上抛（交给上层代码去进行统一错误展示）。

## 使用中间件统一处理错误

有了上面的说明，那现在我们就来看看在 koa 里面怎么优雅的实现统一错误处理。

答案就是使用强大的中间件！

我们可以在业务逻辑中间件（一般就是 MVC 中的 Controller）开始之前定义下面的中间件：

```javascript
app.use(function* (next) {
  try {
    yield* next;
  } catch(e) {
    let status = e.status || 500;
    let message = e.message || '服务器错误';

    if (e instanceof JsonError) { // 错误是 json 错误
      this.body = {
        'status': status,
        'message': message
      };
      if (status == 500) {
        // 触发 koa 统一错误事件，可以打印出详细的错误堆栈 log
        this.app.emit('error', e, this);
      }
      return;
    }

    this.status = status;
    // 根据 status 渲染不同的页面
    if (status == 403) {
      this.body = yield this.render('403.html', {'err': e});
    }
    if (status == 404) {
      this.body = yield this.render('404.html', {'err': e});
    }
    if (status == 500) {
      this.body = yield this.render('500.html', {'err': e});
      // 触发 koa 统一错误事件，可以打印出详细的错误堆栈 log
      this.app.emit('error', e, this);
    }
  }
});
```

可以看到，我们直接执行 `yield* next`，然后 `catch` 执行过程中任何一个中间件的错误，然后根据错误的“特性”，分别进行不同的处理。

有了这个中间件，我们的业务逻辑 controller 中的代码就可以这样来触发错误：

```javascript
const router = new (require('koa-router'));

router.get('/some_page', function* () {
  // 直接抛出错误，被中间件捕获后当成 500 错误
  throw new PageError('发生了一个致命错误');
  throw new JsonError('发送了一个致命错误');

  // 带 status 的错误，被中间件捕获后特殊处理
  this.throw(403, new PageError('没有权限访问'));
  this.throw(403, new JsonError('没有权限访问'));
});
```

## 对 Error 分类

上面的代码里面出现的 `JsonError`、`PageError`，实际上是继承于 `Error` 的两个构造器。代码如下：

```javascript
const util = require('util');

exports.JsonError = JsonError;
exports.PageError = PageError;

function JsonError(message) {
  Error.call(this, message);
}
util.inherits(JsonError, Error);

function PageError(message) {
  Error.call(this, message);
}
util.inherits(PageError, Error);
```

通过继承 `Error` 构造器，我们可以将错误进行细分，从而能更精细的对错误进行处理。

## 参考资料
  - https://www.joyent.com/developers/node/design/errors
  - https://github.com/koajs/koa/wiki/Error-Handling
  - https://imququ.com/post/generator-function-in-es6.html
  - http://purplebamboo.github.io/2014/05/24/koa-source-analytics-1/
  - http://purplebamboo.github.io/2015/01/16/koa-source-analytics-4/

本文采用 [知识共享署名 3.0 中国大陆许可协议](http://creativecommons.org/licenses/by/3.0/cn)，可自由转载、引用，但需署名作者且注明文章出处 。
