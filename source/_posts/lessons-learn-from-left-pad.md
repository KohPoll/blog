title: 从 left-pad 事件我们可以学到什么
date: 2016-03-28
tags:
---

最近 NPM 圈发生了“一个 17 行的模块引发的血案”。[left-pad](https://github.com/azer/left-pad/blob/master/index.js) 工具模块被作者从 NPM 上撤下，所有直接或者间接依赖这个模块的 NPM 包就忧伤的挂掉了，包括 babel 这样的热门项目。

<!-- more -->

而其中的原因大概是这样：作者 Azer 写了一个叫 [kik](https://github.com/starters/kik) 的工具和某个公司同名了，这天公司的律师要求其删掉这个模块，把 kik 这个名字“让”给他们，作者不答应，律师就直接找 NPM 了，而 NPM 未经作者同意就把包的权限转移给了这家公司。于是，Azer 一怒冲冠，将他所有的 NPM 包全部删掉了。

我们不打算讨论这件事中的价值观、自由之精神、法律细节等等，我只想站在一个程序员的角度来凑个热闹，聊聊看法。

## 细分模块是必要的

有人会觉得就这几行代码有必要单独抽取成一个模块吗？我觉得有必要，原因如下：
  - 你应该不想把这个工具函数在各个项目里面重复的去复制粘贴，毕竟 DRY（Don't Repeat Yourself） 嘛。
  - 人们管理软件复杂度的通常方法都是拆分，写好一个模块，做好测试，然后直接使用这个模块。“small module” 的想法本来就是很好的。
  - 另外一种想法是我所有的工具函数都统一放在一个模块，这很容易造成工具模块越来越大，什么都往里放。而别人使用这个庞大的工具模块时，很可能其中很多东西根本就用不上。
  - 还有人说这个功能根本不用这么多代码，但是到处写这一行代码也挺烦不是？况且你也不知道一个几行代码能解决的问题有可能随着需求的变化需要更复杂的实现（比如：更多的定制性、性能要求）。

所以，NPM 一直提倡和推动的 small module 的理念并没有什么错，我们不应该因为这个事件就把各种“小模块”的依赖从自己项目中去掉，甚至自己也不写“小模块”了。

## 不要太依赖他人

如果我们想构建稳定的应用，非常重要的一点就是“不要将自己的全部身家都拴在别人的裤腰带”上，你永远不知道那个别人什么时候会“扑街”。其实在代码实现中，我们一直都被教导要小心强依赖，依赖过强会导致我们不够灵动。“牵一发而动全身”是很可怕的一件事。

具体到 Node 模块的依赖这件事上，太依赖他人会有些什么“可怕”的事情？
  1. 模块全部要从 NPM 的 registry 拉取然后安装，每天的持续集成越来越慢、越来越慢。
  2. 像 left-pad 这个模块一样，你依赖的模块被作者怒删了，应用挂掉。
  3. 你在 package.json 里面指定依赖时使用了 `~a.b.c` 这种表示法（意思是小版本我都要），这表示每次 npm install 时其实获取到的模块依赖很可能是和你测试后发布的版本不一致的（模块发布了新的小版本），心里慌慌的。
  4. 你依赖模块的作者是个傻逼，不小心将不兼容的改动当作小版本发布了一个新版。npm install 或者 npm update 以后你依赖了这个新版，应用挂掉了。

*注：对 3、4 有疑问的可以查看：<https://segmentfault.com/a/1190000004221514>*

是不是有点“细思极恐”？那这里抛出一个解决方案。如果有更好的方案，欢迎讨论。

## 破解强依赖

先来列出我们需要些什么：
  - 在发布前“冻结”依赖模块的版本号。这让我们对安装的依赖有信心，依赖模块的版本都是我们验证、测试过的。
  - 在发布前“打包”依赖模块到自己项目。这让我们可以坦然面对我们依赖的某个模块“没有了”这样的囧境。

### 冻结依赖模块

冻结依赖模块的版本号最简单的办法就是直接在 package.json 里面写死版本号，但是这解决不了深度依赖的问题。我们来看个例子。

假设有下面这样的依赖：

```powershell
A@0.1.0
└─┬ B@0.0.1
  └── C@0.0.1
```

A 模块依赖了 B 模块，B 模块又依赖了 C 模块。我们可以将 B 模块的依赖写死成 `0.0.1` 版本，但是如果 B 模块对 C 模块的依赖写的是 `C: ~0.0.1`，会怎样？

这时候 C 模块更新到了 `0.0.2` 版本，虽然我们安装的 B 模块是 `B@0.0.1`，但是安装的 C 模块却是 `C@0.0.2`。如果不巧这个 `C@0.0.2` 刚好有 bug，那我们的模块很有可能就不能正常工作了。

实际上，NPM 提供了一个叫做 `npm shrinkwrap` 的命令来我们解决这个问题：

```powershell
NAME
  npm-shrinkwrap -- Lock down dependency versions

SYNOPSIS
  npm shrinkwrap

DESCRIPTION
  This  command  locks down the versions of a package's dependencies so that you can control exactly which versions of each  dependency  will be used when your package is installed.
```

这条命令会根据目前我们 `node_modules` 目录下的模块来生成一份“冻结”住的模块依赖（npm-shrinkwrap.json）。

还是上面的例子，我们在模块 A 的根目录执行 `npm shrinkwrap` 后，生成的 `npm-shrinkwrap.json` 文件内容大概是下面这样：

```json
{
    "name": "A",
    "dependencies": {
        "B": {
            "version": "0.0.1",
            "resolved": "http://registry.npmjs.com/B-0.0.1.tgz",
            "dependencies": {
                "C": {  
                    "version": "0.0.1",
                    "resolved": "http://registry.npmjs.com/C-0.0.1.tgz"
                }
            }
        }
    }
}
```

然后，当我们执行 `npm install` 时，依赖查找的“来源”不再是 `package.json`，而是我们生成的 `npm-shrinkwrap.json`，再也不会突然装上什么 `C@0.0.2` 了，依赖里面的模块版本都是我们验证、测试后的版本，让人安心。

注：`npm shrinkwrap` 默认只会生成 `dependencies` 的依赖，不会生成 `devDependencies` 的依赖，如果你真的需要，可以加 `--dev` 参数。

### 打包依赖模块

我们解决了依赖模块版本号的问题，但是每次安装时其实还是会去 NPM 的 registry 获取模块的 tgz 包然后进行安装。我们需要将这些依赖都打包进我们的项目。这可能会带来一些问题（比如：项目体积的增大），但是好处也是显而易见的。

上面生成的 `npm-shrinkwrap.json` 里面有个 `resolved` 字段，表示模块所在的位置，实际上这个字段完全可以写一个文件路径。所以，我们可以递归的遍历 `npm-shrinkwrap.json` 文件，将所有的 tgz 包先下载到我们项目的某个目录，然后改写 `resolved` 字段为对应的文件路径。这样的功能有开发者已经实现了，我们可以直接享用：<https://github.com/JamieMason/shrinkpack>

还是上面的例子：

```powershell
A@0.1.0
└─┬ B@0.0.1
  └── C@0.0.1
```

执行 `shrinkpack` 后，会生成下面的打包目录：

```powershell
node_shrinkpack
 - B-0.0.1.tgz
 - C-0.0.1.tgz
```

和 `node-shrinkwrap.json` 文件：

```json
{
    "name": "A",
    "dependencies": {
        "B": {
            "version": "0.0.1",
            "resolved": "./node_shrinkpack/B-0.0.1.tgz",
            "dependencies": {
                "C": {  
                    "version": "0.0.1",
                    "resolved": "./node_shrinkpack/C-0.0.1.tgz"
                }
            }
        }
    }
}
```

于是，我们以后再进行 `npm install --loglevel=http` 时会发现依赖模块的获取根本没有网络请求了（因为依赖都在我们自己的仓库里了嘛）。

可能有人会说，为啥不直接把 `node_modules` 目录提交进仓库算了？原因主要是这样：
  - 有些模块需要编译，编译是和环境有关的，你当前的环境编译可用，其他环境直接使用该模块不一定能用。
  - `node_modules` 目录里面啥东西都有，太凌乱，很容易把提交给搅乱。diff 时突然 diff 出 `node_modules` 下的源代码、README，你应该不想这样吧？

只存储模块的 tgz 包，安装编译的过程交给 NPM 命令更明智。

### 新方式

于是，现在我们使用 NPM 模块的正确姿势应该是这样了：
  1. 本地安装、更新需要的模块，测试、验证
  2. 执行 `npm shrinkwrap` 将依赖模块的版本冻结
  3. 执行 `shrinkpack .` 将依赖模块打包进仓库
  4. 提交代码（注意要将 `npm-shrinkwrap.json` 和 `node_shrinkpack` 一起提交哦）
  5. 发布模块或者部署应用

如果你觉得这样很繁琐，可以定义一个 NPM 命令：

```json
"scripts": {
  "pack": "npm shrinkwrap & shrinkpack ."
}
```

## 总结

  - 拆分模块是必要的，我们应该坚持模块“小而美”
  - 不要太依赖他人，一定要有依赖方挂掉的应急方案
  - 推荐使用 `npm shrinkwrap`（冻结依赖模块的版本） 加 `shrinkpack`（打包依赖模块到自己项目） 来解决依赖模块的不确定性

## 参考资料

  - https://docs.npmjs.com/cli/shrinkwrap
  - https://github.com/JamieMason/shrinkpack
  - https://nodejs.org/en/blog/npm/managing-node-js-dependencies-with-shrinkwrap/
  - http://blog.keithcirkel.co.uk/how-to-use-npm-as-a-build-tool/

本文采用 [知识共享署名 3.0 中国大陆许可协议](http://creativecommons.org/licenses/by/3.0/cn)，可自由转载、引用，但需署名作者且注明文章出处 。
