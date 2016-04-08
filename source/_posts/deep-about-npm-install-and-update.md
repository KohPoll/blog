title: 深入 Node 模块的安装和发布
date: 2015-12-29
tags:
---

npm 是一个方便的 Node 模块分发、管理工具。我们平常会使用 `npm install` 安装模块，使用 `npm publish` 发布模块。事实上除了基本功能，这 2 个命令还有其他非常有用的特性。这篇文章会给大家介绍这些命令的一些“高级”用法。

## 模块的版本号

我们先来说说版本号。npm 使用的是一种叫做 [semantic version](http://semver.org/) 的规范，它的规则很简单，总结起来就是下面几条：
  - 使用 semver 的软件必须定义公开、严谨、易于理解的 API。也就是模块要提供功能给用户。
  - 版本号格式为：`X.Y.Z`，并且 X、Y、Z 均为正整数并且不断递增。X 表示大版本（major）、Y 表示小版本（minor）、Z 表示补丁版本（patch）。
  - 一个版本发布后，此版本内容不能再变更，变更必须再发布一个新版本。也就是不能覆盖发布。
  - `0.Y.Z` 表示初始版本，这种版本下的 API 不能保证稳定，随时可能变更。
  - 当进行了向后兼容的 bug 修复时，补丁版本 Z 必须增加。
  - 当引入了向后兼容的新功能时，小版本 Y 必须增加，同时 Z 必须重置为 0（小版本里面可能会包含 bug 修复）。
  - 当引入了不兼容的变更时，大版本 X 必须增加，同时 Y、Z 必须重置为 0（大版本里面可能会包含小版本或者补丁版本的改动）。
  - `X.Y.Z` 后面还可以加预发布版本号、构建信息，格式为：`X.Y.Z-pre_lease+build_meta`，比如：`1.0.0-alpha+20151226`、`1.0.0-beta.2+20151230`。
  - 进行版本号比较时，遵循下面的规则：1）依次按数值比较 X、Y、Z 的值，直到第一个不同的位置；2）如果两个版本的 X、Y、Z 都相等，含有 pre-release 版本号的较小；3）如果两个版本的 X、Y、Z 都相等并且都含有 pre-release 版本号，要单独比较 pre-release 版本。比如：`1.0.0 < 2.0.0 < 2.1.0 < 2.1.1`，`1.0.0-alpha < 1.0.0`，`1.0.0-alpha < 1.0.0-alpha.1 < 1.0.0-alpha.beta < 1.0.0-beta < 1.0.0-beta.2`。

## 模块的依赖管理

在进行项目开发时，对于依赖的模块进行管理是非常必要的。这种管理主要体现在：
  1. 如何方便的获取项目需要使用的模块
  2. 当前项目依赖了哪些模块，或许还会指定模块依赖的环境（哪些模块是生产阶段依赖的、哪些是开发阶段依赖的）
  3. 模块的哪些版本是和当前项目兼容的，可以直接使用
  4. 其他成员（或者系统）如何方便的获取所有模块，从而能让项目正常运行

下面我们看看 npm 是如何帮助我们解决上面这些问题的。

### 获取模块

我们在进行项目开发时，如果需要某个功能，都会先去 npm registry 上搜一搜，看看有没有类似的模块可以直接复用，从而提高开发效率。获取模块非常简单，我们在项目的根目录直接执行 `npm install name` 就可以将模块安装在 `node_modules` 目录下，然后直接 `require` 就可使用。

### 依赖了哪些模块

上面获取模块的方法是一种“临时”起意的方式，并不能记录我们依赖模块的具体信息（除非我们自己去查找）。我们安装模块时，可以执行 `npm install name --save`（生产阶段的依赖） 或是 `npm install name --save-dev`（开发阶段的依赖），来将模块信息保存到项目的 `package.json` 文件中。

### 模块依赖表示法

执行 `npm install name --save` 来安装模块时，npm 会首先安装模块的最新版本，然后将模块名及模块版本号以最“保险”的表示方式写入到 `package.json` 文件中。

比如，我们执行 `npm install koa --save`，安装模块后，会更新`package.json` 文件的 `dependencies` 字段：

```json
{
  "dependencies": {
    "koa": "~1.1.0"
  }
}
```

项目对模块的依赖可以使用下面的 3 种方法来表示（假设当前版本号是 1.1.0 ）：
  1. 兼容模块新发布的补丁版本：`~1.1.0`、`1.1.x`、`1.1`
  2. 兼容模块新发布的小版本、补丁版本：`^1.1.0`、`1.x`、`1`
  3. 兼容模块新发布的大版本、小版本、补丁版本：`*`、`x`

可以看到，npm 默认会将依赖表示为最“保险”的方式（即：获取模块新发布的补丁版本），我们也可以使用 `npm install koa --save-exact` 来精确的指定模块版本（这种情况依赖就直接表示为：`"koa": "1.1.0"`）。

除了获取模块的最新版本，我们还可以精确获取模块的某个具体版本，如：`npm install koa@1.0.0 --save`，这种情况依赖就会表示为：`"koa": "~1.0.0"`。

### 安装模块

模块的依赖都被写入了 `package.json` 文件，其他合作者（比如：其他小伙伴、或者是部署服务）只要进入项目的根目录，执行 `npm install` 就可以将依赖的模块全部安装到 `node_modules` 目录下。

比如下面的 `package.json`：

```json
{
  "dependencies": {
    "bluebird": "~3.1.1",
    "request-promise": "^1.0.2",
    "lodash": "*"
  },
  "devDependencies": {
    "mocha": "~2.3.4"
  }
}
```

当我们执行 `npm install` 时，就会安装 `dependencies` 及 `devDependencies` 字段里列出的所有模块。如果只想安装 `dependencies` 里面列出的模块，可以使用 `npm install --only=production`。

这里 npm 实际安装的模块是根据依赖的表示来决定的：
  - `"bluebird": "~3.1.1"` 表示会安装最新的补丁版本。比如：安装 3.1.2、3.1.3 等，但是不会安装 3.2.0 这种小版本或者4.0.0 这种大版本（就算这些版本是最新的）。这种表示法是比较保险的，因为补丁版本只是 bug 修复，不会新增功能。
  - `"request-promise": "^1.0.2"` 表示会安装最新的补丁版本或者小版本。比如：安装  1.1.0、1.1.1 等小版本，或者 1.0.3、1.0.4 等补丁版本，但是不会安装 2.0.0 这种大版本。而且，总是安装版本号最大的（也就是优先安装小版本）。这种表示法通常是保险的，小版本会增加向后兼容的功能。
  - `"lodash": "*"` 表示安装最新的版本（不管这个版本是大版本、小版本、还是补丁版本）。这种表示法非常危险，如果有大版本，直接就安装了大版本，而大版本通常是不会向后兼容的，可能导致项目功能运行异常。

## 模块的发布

对于模块的提供者来说，每一次发布都应该尽量遵循 semver 的规范来变更版本号，不要让用户困惑甚至是想骂人......

在“模块的版本号”部分我们介绍了 semver 规范，npm 也提供了相关的命令来让我们方便的进行版本号变更：
  - 升级补丁版本号：`npm version patch`
  - 升级小版本号：`npm version minor`
  - 升级大版本号：`npm version major`

一个比较合适的发布流程可以是：
  - 根据此次变更执行 `npm version [new version]` 命令（npm 会根据 `new version` 指定的类型更新 `package.json` 中的` version` 字段，然后进行一次 commit，最后打上一个该版本号的 tag）。
  - 执行 `git push origin master --tags`，将改动同步到远程代码仓库。
  - 执行 `npm publish`，将模块发布到 npm 仓库。

### 模块的 tag

模块安装命令的最简形式 `npm install name` 的完整版其实应该是：`npm install name@latest`。这里的 `latest` 是模块版本的一个 tag，会对应到模块的一个具体版本。

我们来看一个例子：模块 koa 在 npm registry 上的信息如下（完整版见：<http://registry.npmjs.com/koa>）：

```json
{
  "name": "koa",
  "dist-tags": {
    "latest": "1.1.2",
    "next": "2.0.0-alpha.3"
  },
  "versions": {
    "0.0.1": {...},
    "1.1.2": {...},
    "2.0.0-alpha.3": {...}
  }
}
```

当执行 `npm install koa` 时，其实是执行 `npm install koa@latest`，而这个 `latest` 等于 `dist-tags.latest`（版本 1.1.2），最后版本 1.1.2 被安装，同时依赖会标记为 `"koa": "~1.1.2"`。

当执行 `npm install koa@next` 时， `next` 等于 `dist-tags.next`（版本 2.0.0-alpha.3），最后版本 2.0.0-alpha.3 被安装，同时依赖会标记为 `"koa": "~2.0.0-alpha.3"`。

模块的维护者在进行模块发布时，可以指定将当前版本发布为哪个 tag（默认是 latest）。

当执行 `npm publish` 时，会首先将当前版本发布到 npm registry，然后更新 `dist-tags.latest` 的值为新版本。

当执行 `npm publish --tag=next` 时，会首先将当前版本发布到 npm registry，并且更新 `dist-tags.next` 的值为新版本。这里的 next 可以是任意有意义的命名（比如：v1.x、v2.x 等等）。

可以看出，tag 就是对模块某一个版本的标记（类比 git 中 tag 是对某一次提交的标记），并且这个 tag 还可以随时更新为新的版本号。

能对版本打 tag，使得我们在维护多个版本时非常方便。比如，可以像 koa 的做法一样，新开一个 next 的 tag 来提供新版本给社区试用，而不影响现在的稳定版本。等到新版本逐渐稳定后，再将其发布为 latest 即可。

## 其他问题

这样看起来，发布 Node 模块是不是很简单？其实，我们没有说的东西还有很多。比如下面这些问题：

每次发布前怎么保证模块提供的功能没有损坏？可以考虑接入持续集成系统，每次 push 运行单元测试，未通过不进行发布。

版本号其实对维护者不友好，很容易改错，有没有什么好方法来解决？可以考虑规范提交信息，维护者只是描述变更，从提交信息中抽取变更类型然后生成正确的版本号和变更日志（比如：angular 的提交信息规范：<https://docs.google.com/document/d/1QrDFcIiPjSLDn3EL15IJygNPiHORgU1_OOAqWjiDU5Y/edit>）

这些问题要是可以自动化多好！我尝试搜索了下，果然有开发者在搞了。推荐大家看看这个工具：<https://github.com/semantic-release/semantic-release>，很好的解决了上面的问题。

但是其实最最核心的问题是维护者得养成好习惯（写单测、规范提交信息），不然再好的工具都白搭。我也在慢慢培养中，真是痛苦......

## 参考资料
  - <http://semver.org/>
  - <https://docs.npmjs.com/misc/semver>
  - <https://docs.npmjs.com/cli/dist-tag>
  - <https://docs.npmjs.com/cli/publish>
  - <https://docs.npmjs.com/cli/install>
  - <https://github.com/semantic-release/semantic-release>
  - <https://medium.com/greenkeeper-blog/one-simple-trick-for-javascript-package-maintainers-to-avoid-breaking-their-user-s-software-and-to-6edf06dc5617>