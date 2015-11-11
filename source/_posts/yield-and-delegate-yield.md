title: 'yield和yield*'
date: 2015-11-11 11:29:32
tags:
---


最近在看`koa`的中间件实现时，总是看到`yield* next`这种用法，觉得很困惑。下面是学习成果。

# 代理yield

我们先来看看原生的代理`yield`（delegating yield）的含义和用法。

首先是下面的代码（普通的`yield`）：

```javascript
function* outer() {
    yield 'begin';
    yield inner();
    yield 'end';
}

function* inner() {
    yield 'inner';
}

var it = outer(), v;

v = it.next().value;
console.log(v);            // -> 输出：begin

v = it.next().value;
console.log(v);            // -> 输出：{}
console.log(v.toString()); // -> 输出：[object Generator]

v = it.next().value;
console.log(v);            // -> 输出：end
```

然后是下面的代码（代理`yield`）：

```javascript
function* outer() {
    yield 'begin';

    /*
     * 这行等价于 yield 'inner';就是把inner里面的代码替换过来
     * 同时获得的rt刚好就是inner的返回值
     */
    var rt = yield* inner();
    console.log(rt);

    yield 'end';
}

function* inner() {
    yield 'inner';
    return 'return from inner';
}

var it = outer(), v;

v = it.next().value;
console.log(v);            // -> 输出：begin

v = it.next().value;
console.log(v);            // -> 输出：inner
// -> 输出：return from inner

v = it.next().value;
console.log(v);            // -> 输出：end
```

根据[文档](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/yield*)的描述，`yield*`后面接受一个`iterable object`作为参数，然后去迭代（`iterate`）这个迭代器（`iterable object`），同时`yield*`本身这个表达式的值就是当迭代器迭代完成时（`done: true`）的返回值。

啊，好绕。我自己的理解就是，`yield*`一般用来在一个generator函数里“执行”另一个generator函数，并可以取得其返回值。

# yield和co

我们再来看看`co`中`yield`的用法，代码如下：

```javascript
co(function* () {

  // yield promise
  var a = yield Promise.resolve(1);
  console.log(a); // -> 输出：1

  // yield thunk
  var b = yield later(10);
  console.log(b); // -> 输出：10

  // yield generator function
  var c = yield fn;
  console.log(c); // -> 输出：fn_1

  // yield generator
  var d = yield fn(5);
  console.log(d); // -> 输出：fn_5

  // yield array
  var e = yield [
    Promise.resolve('a'),
    later('b'),
    fn,
    fn(5)
  ];
  console.log(e); // -> 输出：['a', 'b', 'fn_1', 'fn_5']

  // yield object
  var f = yield {
    'a': Promise.resolve('a'),
    'b': later('b'),
    'c': fn,
    'd': fn(5)
  };
  console.log(f); // -> 输出：{a: 'a', b: 'b', c: 'fn_1', d: 'fn_5'}

  function* fn(n) {
    n = n || 1;
    var a = yield later(n);
    return 'fn_' + a;
  }

  function later(n, t) {
    t = t || 1000;
    return function(done) {
      setTimeout(function() { done(null, n); }, t);
    };
  }

}).catch(function(e) {
  console.error(e);
});
```

等等！为什么`co`可以直接`yield`一个`generator`而不用`yield*`，更奇怪的是居然可以直接`yield`一个`generator function`？

实际上，`co`里面做了封装，在`co`的`toPromise`代码里有这样的判断：

```javascript
if (isGeneratorFunction(obj) || isGenerator(obj)) return co.call(this, obj);
```

所以，其实我们的代码：

```javascript
// yield generator function
var b = yield fn;
// yield generator
var c = yield fn(5);
```

实际上会变成：

```javascript
// this是co的实例
var b = yield co.call(this, fn);
var c = yield co.call(this, fn_generator);
```

而`co`方法最后是返回一个`Promise`的，所以，`yield`这个`Promise`后，保证了正常结果。

其实，我们也可以直接用“原生”的语法写成：

```javascript
var b = yield* fn();
var c = yield* fn(5);
```

因为`yield*`只接受一个`generator`，所以，我们需要调用`generator function`来返回一个`generator`。

所以，`co`背后是封装了很多黑魔法的，这多创建了闭包，会有一点点点点的内存消耗。不是我打错了，根据这里的讨论：<https://github.com/koajs/compose/issues/2>，确实没有非常要命的影响。而我们的`TJ`大神非常不喜欢在`yield`和`代理yield`间切换，他觉得本来就应该是语言自己去帮我做代理的，反正我就是不想写`yield*`。

不过嘛，这个`issue`最后的结果还是将`next`实现成了一个`generator`，所以我们才会在各种`koa`的中间件里面看到`yield* next`这种写法。

# yield*的作用

好了，既然这样，那`yield*`到底有啥用啊？还是有用处的。具体有下面这几点：

  - 用原生语法，抛弃`co`的黑魔法，换取一点点点点性能提升
  - 明确表明自己的意图，避免混淆
  - 调用时保证正确的`this`指向

第一点上面有提到，就不展开说了。

第二点我们来解释下。看看下面的代码，你能知道`yield`后面的到底是个什么鬼吗？

```javascript
var v = yield later();
```

如果你不看`later`的实现，实际上你并不知道。`later`方法完全可以实现成返回一个`Promise`，返回一个`thunk`，返回一个数组、对象，或者就是返回`generator`。

但是下面这样的代码：

```javascript
var v = yield* later();
```

我们就很自信的知道`later`肯定是返回一个`generator`了。

第三点也是`co`黑魔法的一个大坑。

比如我们有下面这样的代码：

```javascript
function later(n, t) {
  t = t || 1000;
  return function(done) {
    setTimeout(function() { done(null, n); }, t);
  };
}
function Runner() {
  this.name = 'runner';
}
Runner.prototype.run = function* (t) {
  var r = yield later(this.name, t);
  return 'run->' + r;
};

var runner = new Runner();
var result = yield runner.run;
console.log(result); // -> 输出：run->undefined
```

你会发现，`result`是`run->undefined`，这说明`this`的指向不对了。这是因为`co`实际上是类似这样执行上面的代码的：

```javascript
var runner = new Runner();
var result = yield co(function {
    runner.run.call(this); // 这里的this变成了co的实例...
});
```

破解大法也很简单，就是使用原生语法：

```javascript
var result = yield* runner.run();
console.log(result); // -> 输出：run->runner
```

那么，我就说完了。如果各位没有了解过`co`的黑魔法，看的可能会有点累，推荐大家看看参考资料的第一条。

以上。

参考资料：

  - <http://purplebamboo.github.io/2014/05/24/koa-source-analytics-2/>
  - <http://www.jongleberry.com/delegating-yield.html>
  - <https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/yield*>
