title: 纯 css 实现容器动态宽高相等
date: 2015-08-31
tags:
---

假设有这样的需求：不知道容器的宽度（容器宽不能写死，动态变化），如何只通过css让容器的高总是等于容器的宽？

可能大家会想到用`height:100%`，当然这肯定是不行的，`height`的百分比是根据父容器计算的，不是当前容器，根本满足不了我们的需求。

方法其实很简单，但是很巧妙。

我们需要知道的基础是，`padding-top`、`padding-bottom`的百分比是根据父容器的`width`（宽度）计算的，而不是`height`（高度）。

有了这个基础，我们就可以来实现了。我们的html结构是下面这样。

```html
<div class="container">
    <div class="content">
        <!-- content here -->
    </div>
</div>
```

首先，我们需要一个子元素来将容器的高度撑开，样式如下：

```css
.container:before {
    content: " ";
    display: block;
    padding-top: 100%; /*利用padding来撑开*/
}
```

然后，我们让实际的内容完全占满容器，样式如下：

```css
.container {
    position: relative;
}
.container .content {
    position: absolute;
    top: 0; bottom: 0; right: 0; left: 0;
}
```

这样就只用样式实现了高总是等于宽的容器。

因为`padding-top`的百分比是基于父容器`width`的计算，我们完全可以实现宽高是各种比例的容器。比如，高是宽的2倍、高是宽的1/2，则只需要修改样式为：

```css
.container:before {
    content: " ";
    display: block;
    
    /* 2倍 */
    padding-top: 200%;
    /* 0.5倍 */
    padding-top: 50%;
}
```

实际的代码示例：<http://runjs.cn/code/jqer8q5i>

以上。