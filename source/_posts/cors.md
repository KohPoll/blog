title: 使用cors
date: 2015-09-01
tags:
---

最近使用了跨域ajax来发起post请求，对cors进行了一些学习，下面是学习的一些记录。

<!-- more -->

web开发中最常见的操作就是在进行接口调用。发起异步接口调用，根据返回来进行相关操作，这差不多就是目前前端开发中最常见的操作了。

但是，通常的接口调用存在一定的局限性。xhr发起的异步接口，必须遵循浏览器的”同源策略“，而jsonp请求只能支持跨域GET请求。于是，cors（Cross-Origin Resource Sharing）就产生了。

遵循一定”沟通“原则的前提下，cors允许不同域名的客户端和服务器之间进行交互。比如：a.com下的一个页面可以向b.com发起ajax请求（甚至是post请求）并取得响应结果。

# 交互流程

我们先暂时忽略浏览器发起cors请求的Javascript方法，来看看浏览器端和服务器端在cors请求中是怎么进行一问一答来进行交互的。

为了方便说明，我们假设发起请求的页面地址为：`http://a.com/page.html`，请求的地址为：`http://api.com/cors`

## 基本支持

客户端请求：

```javascript
GET /cors HTTP/1.1
Origin: http://a.com/page.html
Host: api.com
```

服务端响应：

```javascript
Access-Control-Allow-Origin: http://a.com
```

可以看到，在请求端，跨域请求会带上一个`Origin`请求头（浏览器强制进行添加），服务端在响应时必须提供`Access-Control-Allow-Origin`响应头，用来告知客户端此次跨域请求所允许的来源。如果不提供这个头或者说提供的来源和请求的`Origin`不一致，将导致请求失败。这个响应头可以直接指定`*`，表示任何来源都支持。

如果我不想写`*`，并且要支持多个来源呢？答案是不可以。`Access-Control-Allow-Origin`响应头只允许指定一个值，要么写死的地址，要么`*`。

## cookie支持

默认情况下，跨域请求是不会携带cookie的。如果需要，客户端在发起请求时，可以指定`XMLHttpRequest`的实例对象的`withCredentials`属性为`true`。同时，服务端在进行响应时，必须提供`Access-Control-Allow-Credentials: true`响应头，否则将导致请求失败。另外，需要注意的是，当指定`Access-Control-Allow-Credentials: true`后，`Access-Control-Allow-Origin`响应头将不能指定`*`，否则请求也会失败。

## 更多定制

实际上，在发起请求时，一般会将请求区分成2种类型：

  * ”简单“请求
    * GET、HEAD、POST（Content-Type必须是：application/x-www-form-urlencoded、multipart/form-data、text/plain之一）
    * 没有设置自定义的请求头（比如：X-Custom-Header: foo这种）
  * “复杂”请求

在发起请求时，客户端首先会检查该次请求是一个”简单“请求还是”复杂“请求，如果是”简单“请求，就直接发起请求；如果是”复杂“请求，客户端会先发起一个`OPTION`请求作为预检查请求，如果预检查请求通过，才会发起实的请求。

假设我们现在发起一个`PUT`请求，并设置两个自定义头，分别为：`x-custom-a: 1`、`x-custom-b: 2`，实际的交互流程将会如下：

客户端预检查请求：

```javascript
OPTIONS /cors HTTP/1.1
Origin: http://a.com/page.html
Access-Control-Request-Method: PUT
Access-Control-Request-Headers: x-custom-a, x-custom-b
Host: api.com
```

服务端响应预检查请求：

```javascript
Access-Control-Allow-Origin: http://a.com
Access-Control-Allow-Headers: x-custom-a, x-custom-b
Access-Control-Allow-Methods: GET, POST, PUT
```

可以看到，预检查请求会将请求方法（`Access-Control-Request-Method: PUT`）、自定义请求头（`Access-Control-Request-Method: x-custom-a, x-custom-b`）进行传递，服务端需要”回答“请求方法和请求头是否是”允许“的，通过制定：`Access-Control-Allow-Headers: x-custom-a, x-custom-b`及
`Access-Control-Allow-Methods: GET, POST, PUT`。

一旦两端提供的”情报“不一致，将导致请求失败。如果成功对应上了，客户端就会发起实际的请求了。如下：

客户端实际请求：

```javascript
PUT /cors HTTP/1.1
Origin: http://a.com/page.html
Host: api.com
```

服务端响应实际请求：

```javascript
Access-Control-Allow-Origin: http://a.com
```

# 发起cors请求

浏览器对cors的支持情况还算令人满意，详细支持情况可以参见这里：<http://caniuse.com/#search=cors>。

chrome、firefox、safari使用[XMLHttpRequest2](http://www.html5rocks.com/en/tutorials/file/xhr2/)来发请求，而IE8+则使用[XDomainRequest](http://blogs.msdn.com/b/ieinternals/archive/2010/05/13/xdomainrequest-restrictions-limitations-and-workarounds.aspx)。

我们可以直接使用jQuery的`ajax`方法，实例如下：

”简单“请求：

```javascript
$.ajax({
    method: 'get',
    url: 'http://b.com:3000',
    success: function(res) { // 服务端需要响应：Access-Control-Allow-Origin
        alert(res);
    },
    error: function() {

    }
});
```

携带cookie：

```javascript
$.ajax({
    method: 'get',
    url: 'http://b.com:3000',
    xhrFields: {
        withCredentials: true // 服务端需要响应：Access-Control-Allow-Credentials: true
    },
    success: function(res) {
        alert(res);
    },
    error: function() {

    }
});
```

”复杂“请求：

```javascript
$.ajax({
    url: 'http://b.com:3000',
    method: 'put', // 服务端需要响应：Access-Control-Allow-Methods
    headers: { // 服务端需要响应：Access-Control-Allow-Headers
        'x-custom-a': 1,
        'x-custom-b': 2
    },
    success: function(res) {
        alert(res);
    },
    error: function() {

    }
});
```

# 参考资料

- http://www.html5rocks.com/en/tutorials/cors/
- https://developer.mozilla.org/en-US/docs/Web/HTTP/Access_control_CORS

本文采用 [知识共享署名 3.0 中国大陆许可协议](http://creativecommons.org/licenses/by/3.0/cn)，可自由转载、引用，但需署名作者且注明文章出处 。
