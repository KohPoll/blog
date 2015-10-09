title: 谈谈响应式
date: 2015-10-09 17:54:50
tags:
---


这是自己关于PC上响应式网站的一次实践的总结。尽管在响应式满天飞的当下，这个文章看起来好像有点“过时”，但是，在实践及总结的过程中，理解了很多原先并不明确的知识，有一些自己的心得和体会，希望能对有做响应式的同学有帮助，如有不正确的地方跪求指出。

# 我们为什么要做响应式

随着手机端、Pad端用户的增长，我们一般会分别对手机端、Pad端用户单独开发一套页面，这样我们能更有针对性的对这些终端进行优化实现。看起来好像确实没有必要去做响应式。但是有时可能资源不足，直接用PC页面响应到pad也是有可能的。另外，在PC端，用户的屏幕尺寸其实还是参差不齐，会有一些小屏幕需要考虑（主要是1024x768）。

其实做不做响应式，可能很多时候还是和具体产品有关。毕竟目前大屏幕是主流，完全按1190布局其实问题也不大。但是如果我们能花些时间，让尽可能多的用户都可以很好的浏览我们的站点也是一件有意义的事情。那么，我下面说到的内容，更多的将会基于PC端的响应式来谈。

# 响应式的好伙伴之一——媒体查询

想必大家都知道要进行响应式设计必定离不开media query（媒体查询）。下面我就对media query进行一个简要的回顾，以及我觉得应该怎么样来更合理的使用它。

## 语法

实际上，早在css2中就已经支持了为不同媒体类型指定不同样式。比如下面的代码就可以分别对屏幕和打印设备指定不同的样式：

```html
<link rel="stylesheet" href="screen.css" media="screen" />
<link rel="stylesheet" href="print.css" media="print" />
```

```css
@media screen { /* style */ }
@media print { /* style */ }
```

而css3的media query对上面的查询进行了增强，可以让我们根据设备的特性来设定不同的样式。媒体查询由媒体类型和一个或多个检测媒体特性的条件表达式组成，可以检测的媒体特性包括：width（视口宽度）、height（视口高度）等等。

比如下面的media query：

```css
@media screen and (max-width: 800px) { /* style here */ }
```

`@media`是查询的开始；`screen`是媒体类型（可以省略，默认是`all`）；`(max-width: 800px)`是检测媒体特性的表达式，会跟进具体情况被解析为`true`或`false`；`and`是逻辑操作符（表示”且“），其他的操作符包括：表示”或“的`,`、表示”非“的`not`。

更复杂的media query可以是下面这样：

```css
@media not screen and (orientation: portrait) and (min-width: 800px), projection { /* style here */ }
```

更多细节可以参考：http://www.w3.org/TR/css3-mediaqueries/

## max-width和min-width

上面我们看到media query中出现了`min-width`的写法，这里我们来详细说明下`min-width`、`max-width`以及`width`的含义。

media query支持的媒体特性其实有很多，比如设备方向（orientation）、宽高比（aspect-ratio）等等。但是，我们最常用的是视口宽度（viewport width）。这里需要强调的是`width`表示的是视口宽度（包括滚动条在内），实际上就是`window.innerWidth`的值。

`max-width`表示的含义是小于等于，比如`(max-width: 990px)`表示的就是视口宽度小于等于990px；而`min-width`表示的含义是大于等于，比如`(min-width: 990px)`表示的是视口宽度大于等于990px。比如，下面的代码就表示当视口宽度大于等于767px并且小于等于1006px时应用样式`background-color:blue`：

```css
@media (min-width: 767px) and (max-width: 1006px)  { background-color: blue; }
```

## 额外说明

前面我们提到`width`实际上是包含了滚动条宽度的视口宽度，而一般视觉设计师产出的设计稿是没有考虑滚动条的，比如，1190其实就是指布局容器是1190px。这样我们在书写media query时要特别注意。

假设有这样的media query，`@media (max-width: 1190px) { /* style here */ }`，表示当窗口小于等于1190px时应用相关样式（实际上此时应用的样式就是990版本）。

现在考虑这样的情况：浏览器窗口宽度刚好在1191px并且含有垂直滚动条。此时我们的“查询条件”尚未满足，还不能应用990版本的样式，而且由于这里的`width`是包含了滚动条的宽度，最后留给我们的布局空间实际上只有1177px（减去chrome下的14px滚动条宽度，其他浏览器也类似），根本放不下1190px的容器，于是会出现横向滚动条。

所以，我们在书写`min-width`或是`max-width`时需要将滚动条的宽度考虑进去，另外，如果页面中还有像sidebar这样会占据布局空间的元素也应该一并考虑，避免空间不够发生sidebar遮挡主体元素或是横向滚动条的出现。实际增加的数值可以稍微多几像素，以达到提前响应。

## media query推荐书写顺序

前面我们看到media query的“查询条件”可以包含很复杂的逻辑运算，就实际使用上来看，我觉得完全没有必要。复杂的逻辑运算会导致不容易debug、后期维护困难等问题。如果我们将media query和css的“层叠”机制（样式表中，后面的样式覆盖前面的样式）进行结合，基本上不用额外的逻辑运算符，只用一个查询表达式就能满足需求，而且样式十分容易理解。

考虑下面的css样式：

```css
.w { margin: 0 auto; }

/* 默认，750样式 */
.w {
  width: 750px;
  background-color: blue;
}
/* 大于990px+20px时，应用990样式，覆盖前面的样式（第一个断点） */
@media (min-width: 1010px) {
  .w {
    width: 990px;
    background-color: green;
  }
}
/* 大于1190px+20px时，应用1190样式，覆盖前面的样式（第二个断点） */
@media (min-width: 1210px) {
  .w {
    width: 1190px;
    background-color: red;
  }
}
```

正是因为css的层叠机制（后面的样式总会覆盖前面的样式），我们可以在每个断点下将前面的样式覆盖掉来实现不同断点间的样式切换。

可能有人会提出疑问”为什么会先写最小的那个断点？而不是反过来？”。其实是完全可以反过来的，下面的代码就是反过来书写的css（从最大断点写起）：

```css
.w {
  width: 1190px;
  background-color: red;
}
/* 小于1190px+20px时，应用990样式 */
@media (max-width: 1210px) {
  .w {
    width: 990px;
    background-color: green;
  }
}
/* 小于990px+20px时，应用750样式 */
@media (max-width: 1010px) {
  .w {
    width: 750px;
    background-color: blue;
  }
}
```

之所以我比较倾向于从最小的断点写起，是因为断点和样式对应关系比较清晰（下面的查询条件为了解释清晰，减掉了提前响应的20px）：条件是`min-width:990px`时样式刚好就是990版本；而从最大断点开始写起，条件是`max-width:1190px`时样式却是990版本的。另外，从最小断点开始写起，也方便我们对样式进行统一兼容处理（后面“兼容处理方案”部分会说到）。

# 响应式设计的好伙伴之二——matchMedia

上面我们详细说明了响应式设计中的样式问题，但是随着页面交互变得复杂，仅仅用样式可能没法很好的完成需求，这时就需要借助js了。比如，你想在某个断点到达时使用js来做些处理。这就引出了我们要介绍的响应式设计的第二个好伙伴，`matchMedia`方法。

`matchMedia`是`window`对象上的一个“全局”方法，可以接受一个media query字符串作为参数，然后返回一个`media query list`对象，该对象拥有`matches`、`media`属性及`addListener`、`removeListener `方法。考虑下面的实例代码：

```js
var mql = window.matchMedia('(max-width: 1010px)');
console.log(mql, mql.matches, mql.media);
function handler(mql) {
    if (mql.matches) {
        // 视口小于等于1010px了，进行相应处理
    } else {
        // 视口大于等于1010px了，进行相应处理
    }
}
mql.addListener(handler);
handler(mql);
```

我们传入了`(max-width: 1010px)`字符串作为参数，返回了一个`media query list`对象，该对象的`matches`属性是一个布尔变量，告知传入的“媒体查询条件”是否满足；`media`属性就是咱们传人的“媒体查询条件”；重点是`addListener`方法（注意，不是`addEventListener`方法哦），该方法会注册一个在媒体特性变更时得到执行的回调函数，回调函数会接受一个`media query list`对象作为参数（类似注册事件时的事件回调函数接受的`event`对象）。

上面例子里的回调函数`handler`在窗口宽度变化时，会得到2次回调，一次是窗口宽度小于等于1010px了（此时查询条件满足，所以`matches`为`true`），一次是窗口宽度大于1010px了（此时查询条件不满足，所以`matches`为`false`）。顺带一提，我们初始时主动调用了一次`handler`函数，主要是因为刚进页面时，窗口并没有改变，回调不会执行，此时主动调用`handler`，能保证相应的处理逻辑得到执行。这就类似我们绑`resize`事件时，也会主动调用一次相关处理函数一样，保证初始状态时处理正确。

有了这个函数，我们就可以在响应式的各个断点，通过js来进行一些额外处理了。

# 响应式设计兼容处理方案

我们有了能对不同窗口大小进行样式定制的媒体查询，有了能在不同断点到达时执行的js回调函数，一切看起来很美好。但是，有几个问题我们没有很好的进行解决。

首先是低级浏览器（主要是IE）怎么处理？根据caniuse的结果，http://caniuse.com/#feat=css-mediaqueries http://caniuse.com/#feat=matchmedia css3的media query仅IE9以上支持，而matchMedia仅IE10以上支持，不可能不做兼容。

然后是前面介绍的media query也好、matchMeda也好，提供的功能都有一些不太自然的地方。比如，每次书写max-width还要默念这是小于等于的意思，还要记得处理滚动条或是页面上的sidebar类元素的宽度；matchMedia传递的居然也是一个media query，怎么就不能传类似`{min: 10, max: 100}`这样更自然的东西等等。那么下面就来展示下我的解决方法。

## 样式上的处理

关于样式上的处理，最简单的兼容方法就是在`html`上增加代表不同断点的class（比如：w990、w1190等），然后所有相关样式都在这个断点class的”context“下书写，比如我们上面例子的”兼容“写法如下：

```css
.w {
  width: 750px;
  background-color: blue;
}
@media (min-width: 1010px) {
  .w {
    width: 990px;
    background-color: green;
  }
}
.w990 .w {
  width: 990px;
  background-color: green;
}
@media (min-width: 1210px) {
  .w {
    width: 1190px;
    background-color: red;
  }
}
.w1190 .w {
  width: 1190px;
  background-color: red;
}
```

那么问题来了，样式中写在media query里和写在class里的重复代码怎么办，我们不可能每次都不停的去复制粘贴啊。最简单的方法就是通过像less之类的预处理器来解决。比如定义如下的mixin：

```less
.mq(@w) {
  @fix: 20;
  @m: (@w + @fix);

  @media (min-width: ~"@{m}px") { ._ }
  .w@{w} { ._ }
}
```

这里大家也应该明白了为什么我们的断点会从小开始书写了，因为这让我们定义mixin时减少了很多代码，如果断点从大开始书写，因为断点条件和实际样式没有“一一对应”，我们的mixin将会变成下面这样（每多一个断点将会多一个when条件）：

```less
@fix: 20;
.mq(@w) when (@w = 990) {
  @m: (1190 + @fix);

  @media (max-width: ~"@{m}px") { ._ }
  .w990 { ._ }

}
.mq(@w) when (@w = 750) {
  @m: (990 + @fix);

  @media (max-width: ~"@{m}px") { ._ }
  .w750 { ._ }

}
```

好了，有了上面的mixin，我们的样式将会简化成这样：

```less
.w {
  width: 750px;
  background-color: blue;
}
& { .mq(990); ._() {
  .w {
    width: 990px;
    background-color: green;
  }
}}
& { .mq(1190); ._() {
  .w {
    width: 1190px;
    background-color: red;
  }
}}
```

最终生成的代码就和上面的”兼容“代码一样，但是代码量少了很多，只用维护一处代码即可。而且，我们只需要关注750、990这些断点就行，背后需要考虑的滚动条宽度（即变量fix）、media query和class兼容写法都集中在一处管理了。

## 脚本上的处理——Respond

脚本上的处理实际上包含两部分，一部分是对不支持media query的浏览器的html元素增加断点对应的class，另一部分就是`matchMedia`的兼容处理了。为html元素增加断点class，可以通过绑定resize事件，然后取得视口宽度，再逐一和断点比较后决定加哪一个class。`matchMedia`的兼容处理就稍微有些麻烦，需要维护各个查询条件对应的listener，然后在resize时逐一看是不是状态改变了然后进行回调。

这里面的细节可能需要新开一篇文章来分析，这里直接给出我的实现脚本[Respond](https://github.com/KohPoll/respond/blob/master/respond.js)。以供参考。

引入脚本后，我们在`window`上就拥有了一个`Respond`对象，该对象提供以下属性及方法：

- `hasMediaQueries`  浏览器是否支持media query
- `hasAddListener` 浏览器是否支持media query list的addListener
- `init(breakpoints, fix) `初始化响应式断点。breakpoints是一个数组，表示断点；fix用于处理滚动条等的额外宽度（注意：这里的值应该和样式的fix一致）
- `match(condition)` 判断当前视口宽度是否满足给定条件，条件是拥有`min`和`max`的一个对象，`min`和`max`值完全不用考虑fix值，内部会处理，再也不用写蛋疼的media query了。
- `addListener(condition, handler)` 添加给定条件的回调函数，并在初始时调用一次`handler`保证初始状态正确处理，然后返回一个对象，对象拥有`removeListener`方法，用于移除注册的回调函数。

有了`Respond`，我们可以这样来初始化我们的响应式页面（建议在head里面就引入此脚本并进行初始化，这样断点class可以一开始就处理好）：

```js
Respond.init(
    [
        750
        ,990
        ,1190
    ],
    20
);
```

以及这样来添加断点处的回调：

```js
function _log(w) {
    return function (data) {
        if (data.matches) {
            document.getElementById('text').innerHTML = 'w' + w;
        }
    };
}
Respond.addListener({min: 0, max: 990}, _log(750));
Respond.addListener({min: 990, max: 1190}, _log(990));
Respond.addListener({min: 1190}, _log(1190));
```

更多细节请直接查看demo：http://kohpoll.github.io/respond/demo/demo.html

于是，我们终于讲完了。其实上面的方案可能并不完美，真正的响应式涉及的东西要远比上面复杂很多，但是综合考虑下来，它应该是一个比较“可用”的PC端方案。那么，祝大家都能好好响应。

参考资料：
- http://www.w3.org/TR/css3-mediaqueries/
- http://dbaron.org/log/20110422-matchMedia
- http://caniuse.com/#feat=css-mediaqueries 
- http://caniuse.com/#feat=matchmedia
- http://mux.alimama.com/posts/686
- https://github.com/kissygalleryteam/responsive
- https://github.com/paulirish/matchMedia.js