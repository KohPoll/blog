title: 你可能不知道的 Promise
date: 2016-05-02
tags:
---

开始之前，我们先来一个“脑筋急转弯”。假设 `later` 函数调用后返回一个 promise 对象，下面这 4 种写法有什么区别？

```javascript
// #1
later(1000)
  .then(function() {
    return later(2000);
  });

// #2
later(1000)
  .then(function() {
    later(2000);
  });

// #3
later(1000)
  .then(later(2000));

// #4
later(1000)
  .then(later);
```

大家可以先实际去运行代码看看输出结果：[#1](https://repl.it/CMAl/0)、[#2](https://repl.it/CMAm/0)、[#3](https://repl.it/CMAn/0)、[#4](https://repl.it/CMAo/0)，想想为什么是这样的输出，然后回来继续阅读。

## Promise 是什么

Promise 是对异步处理的一种抽象。在 JavaScript 中，我们通常使用回调函数来进行异步处理：

```javascript
later(1000, function(err) {
  if (err) {
    // 失败时的处理
  } else {
    // 成功时的处理
  }
});
```

Promise 将异步处理进行抽象，并形成规范化的接口：

```javascript
later(1000)
  .then(function() {
    // 成功时的处理
  })
  .catch(function(err) {
    // 失败时的处理
  });
```

基于固定、统一的接口编程，我们可以将复杂的异步处理模式化。

## 创建 Promise 对象

  - `new Promise(function fn(resolve, reject) {})` 会返回一个状态为 pending 的 promise 对象
  - 在 fn 中指定处理逻辑
    - 处理成功时，调用 resolve(结果)，promise 对象状态变更为 fulfilled
    - 处理失败时，调用 reject(Error 对象)，promise 对象状态变更为 rejected

```javascript
const later = function(timeout) {
  return new Promise(function fn(resolve, reject) {
    timeout = parseInt(timeout, 10);
    if (!timeout) {
      return reject(new Error('Invalid Timeout'));
    }
    return setTimeout(function() {
      resolve('later_' + timeout);
    }, timeout);
  });
};
```

获取到 promise 对象后，我们可以为其添加处理方法：
  - 当对象被 resolve 时的处理方法（onFulfilled）
  - 当对象被 reject 时的处理方法（onRejected）

```javascript
later(1000)
  .then(function onFulfilled(data) {})
  .catch(function onRejected(err) {});

later(1000)
  .then(
    function onFulfilled(data) {},
    function onRejected(err) {}
  );
```

## 在 onFulfilled 函数内可以做什么

```javascript
later(1000).then(function() {
  // 在这里能做些什么特别的？
});
```

  - 返回一个 promise 对象
  - 返回一个同步值（什么也不返回，那就是返回 undefined）
  - throw 一个 `Error` 对象

### 返回一个 promise 对象

返回一个 promise 对象就是在做异步操作的串行化。

```javascript
later(1000)
  .then(function(data) {
    // data = later_1000
    return later(2000);
  })
  .then(function(data) {
    // data = later_2000
  });
```

上面的代码会在 1 秒后获得第一个 promise 对象的结果，然后再过 2 秒后获得第二个 promise 对象的结果。

### 返回一个同步值

返回一个同步值可以将同步代码 “promise 化”。

```javascript
later(1000)
  .then(function(data) {
    // data = later_1000
    if (data != 'later_1000') {
      return later(1000);
    }
    return data;
  })
  .then(function(data) {
    // data = later_1000
  });
```

上面的代码保证最后获得的结果总是 `later_1000`（只是等 1 秒还是 2 秒的区别）。更实用的例子是异步获取某个数据（查询 db），我们可以先从本地 cache 查询，查到直接返回同步值，否则返回一个查询 db 的 promise 对象，最终都会获得正确的数据。

函数什么都不返回等于返回了 `undefined`，所以小心下面的写法：

```javascript
later(1000)
  .then(function(data) {
    // data = later_1000
    later(2000);
  })
  .then(function(data) {
    // data = undefined
  });
```

上面的代码在 1 秒后取到第一个 promise 对象的结果，然后不会等待第二个 promise 对象的结果，马上就执行到了第二个 `.then()`。

### （同步）抛出一个 Error 对象

在 `.then()` 里面抛出一个 `Error` 对象可以让错误处理更加方便。

```javascript
later(1000)
  .then(function(data) {
    if (data == 'later_1000') {
      throw new Error('later_1000 invalid');
    }
    if (data == 'later_2000') {
      return data;
    }
    return later(3000);
  })
  .then(function(data) {
  })
  .catch(function(err) {
    // 捕获到错误 Error('later_1000 invalid');
  });
```

只要我们调用 `catch` 添加了 `onRejected` 回调处理，在 `then()` 里面 throw 出的任何同步错误都会在 `catch()` 里面被捕捉到（比如：不小心访问了未定义值啊、JSON.parse 错误啊等等），这让问题定位非常方便。

能捕获到的是“同步”错误，请小心下面的代码：

```javascript
later(1000)
  .then(function(data) {
    setTimeout(function() {
      throw new Error('the err can not catch');
    }, 1000);
  })
  .then(function(data) {
    // data = undefined
  })
  .catch(function(err) {
    // 捕获不到错误
  });
```

另外需要注意 `.then(null, onRejected)` 并不完全等价于 `.catch(onRejected)`：

```javascript
later(1000)
  .then(function(data) {
    throw new Error('this is err');
  })
  .catch(function(err) {
    // 捕获到错误 Error('this is err');
  });

later(1000)
  .then(
    function(data) {
      throw new Error('this is err');
    },
    function(err) {
      // 捕获不到错误 Error('this is err');
    }
  );
```

### 最佳实践

我们总结下这一部分的最佳实践：
  - 总是在 `.then()` 里面使用 `return` 来返回 promise 对象或者同步值
  - 总是在 `.then()` 里面 `throw` 同步的 `Error` 对象
  - 总是使用 `.catch()` 来捕获错误

## 可以给 `.then()` 函数传递什么

目前为止，我们看到给 `.then()` 传递的都是函数，但是其实它可以接受非函数值：

```javascript
later(1000)
  .then(later(2000))
  .then(function(data) {
    // data = later_1000
  });
```

给 `.then()` 传递非函数值时，实际上会被解析成 `.then(null)`，从而导致上一个 promise 对象的结果被“穿透”。于是，上面的代码等价于：

```javascript
later(1000)
  .then(null)
  .then(function(data) {
    // data = later_1000
  });
```

为了避免不必要的麻烦，建议总是给 `.then()` 传递函数。

## 一些编码技巧

### 快速创建 promise 对象

上面我们主要通过 `new Promise(fn)` 的方式来创建 promise 对象，实际上有一个快捷方法 `Promise.resolve(value)` 可以方便的创建 promise 对象。

`Promise.resolve` 的使用场景主要包括：
  - 用最少的代码快速创建一个 promise 对象
  - 在 promise 化的 API 接口中将同步代码 promise 化，更好的捕捉同步代码产出的错误

下面两种写法是等价的，显然使用 `Promise.resolve` 更加简练：

```javascript
new Promise(function(resolve, reject) {
  resolve('value');
}).then(function(data) {});

Promise.resolve('value').then(function(data) {});
```

在 promise 化的 API 中，将同步代码 promise 化可以统一的在 `.catch()` 中捕获异常：

```javascript
function apiReturnPromise() {
  return Promise.resolve().then(function() {
    someFuncMayThrowError();
    return 'xyz';
  }).then(function(data) {
    return doAsync(data);
  });
}
apiReturnPromise()
  .then(function(data) {})
  .catch(function(err) {
    // 可以一致的捕捉到错误
  });
```

### 如何解决 promise 对象间的依赖

实际编码中我们可能经常遇到一个 promise 对象依赖另一个 promise 对象的执行，并且我们两个 promise 对象的结果都需要的情况：

```javascript
later(1000)
  .then(function(dataA) {
    return later(2000);
  })
  .then(function(dataB) {
    // 我们同时需要 dataA 和 dataB
  });
```

前面我们说到在 `.then()` 里可以返回 promise 对象，这个 promise 对象实际上是可以调用自己的 `.then()` 的。下面的代码可以满足需求：

```javascript
later(1000)
  .then(function(dataA) {
    return later(2000).then(function(dataB) {
      return dataA + ':' + dataB;
    });
  })
  .then(function(data) {
    // data = later_1000:later_2000
  });
```

## 异步处理中的并行和串行

在异步处理中，经常需要结合 for 循环来进行批量处理。因为处理都是异步的，这个过程就相应的被分为了两类：
  - 并行：一批异步操作同时执行
  - 串行：一批异步操作挨个执行（一个操作完成后下一个操作才继续）

promise 对象是对异步操作的一个抽象表示，对 promise 对象进行不同的操作可以达到异步处理的并行和串行。

### 并行

`Promise.all` 方法可以接受一个数组（数组里面的元素是 promise 对象）。当数组内所有 promise 对象变为 fulfilled 状态时，才调用 `.then()`；数组内有任一个 promise 对象变为 rejectec 状态时，调用 `.catch()`。

```javascript
const promises = [1000, 2000, 3000].map(function(timeout) {
  return later(timeout);
});
Promise.all(promises)
  .then(function(data) {
    // data = ['later_1000', 'later_2000', 'later_3000']
  })
  .catch(function(err) {
  });
```

数组内 promise 对象所表示的异步操作是同时执行的，并且最后的结果和传递给 `Promise.all` 的数组的顺序是一致的。所以，3 秒钟后我们取得的结果是一个值为 `['later_1000', 'later_2000', 'later_3000']` 的数组。

### 串行

在 `.then()` 里面返回一个 promise 对象就是一种串行。我们需要构造一个类似下面这样的 promise 对象：

```javascript
promise1
  .then(function() { return promise2; })
  .then(function() { return promise3; })
  .then(...);
```

将一个数组转换为一个值，使用 reduce 可以实现：

```javascript
[1000, 2000, 3000]
  .reduce(function(promise, timeout) {
    return promise.then(function() {
      return later(timeout);
    });
  }, Promise.resolve())
  .then(function(data) {
    // data = later_3000
  })
  .catch(function(err) {
  });
```

上面代码将以下面的方式来执行：

```javascript
Promise.resolve()
  .then(function() { return later(1000); })
  .then(function() { return later(2000); })
  .then(function() { return later(3000) })
  .then(function(data) {
    // data = later_3000
  })
  .catch(function(err) {
  });
```

每个 promise 对象表示的异步操作依次执行，最终结果将会在 6 秒后取得（只能取到最后一个 promise 对象的结果，如果都需要的话，需要单独进行处理存储）。

## 参考答案

### 问题 1

```javascript
later(1000)
  .then(function() {
    return later(2000);
  })
  .then(done);
```

执行结果：

```bash
later(1000)  later(2000)            done(later_2000)
|-----1s-----|----------2s----------|
```

### 问题 2

```javascript
later(1000)
  .then(function() {
    later(2000);
  })
  .then(done);
```

执行结果：

```bash
later(1000)  done(undefined)           
|-----1s-----|
             later(2000)
             |----------2s----------|          
```

### 问题 3

```javascript
later(1000)
  .then(later(2000))
  .then(done);
```

执行结果：

```bash
later(1000)  done(later_1000)
|-----1s-----|
later(2000)
|----------2s----------|   
```

### 问题 4

```javascript
later(1000)
  .then(later)
  .then(done);
```

执行结果：

```bash
later(1000)  later(later_1000)
|-----1s-----|
             throw new Error('Invalid Timeout')
```

## 参考资料

  - <http://liubin.org/promises-book/>
  - <https://pouchdb.com/2015/05/18/we-have-a-problem-with-promises.html>

本文采用 [知识共享署名 3.0 中国大陆许可协议](http://creativecommons.org/licenses/by/3.0/cn)，可自由转载、引用，但需署名作者且注明文章出处 。
