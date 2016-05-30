title: 纠结的链接——ln、ln -s、fs.symlink、require
date: 2016-05-30
tags:
---

最近在使用 `fs.symlink` 实现软链时，发现[文档](https://nodejs.org/api/fs.html#fs_fs_symlink_target_path_type_callback)里面写的是：`fs.symlink(target, path)`；然而 `man ln` 的时候显示的是：`ln source_file target_file`；而且，`require` 模块的时候其实还会处理软链但是处理的又不是想象中那样。于是，我彻底被相关东西绕晕。这篇文章算是我的学习笔记，希望对你有帮助。

<!-- more -->

## inode

我们首先来看看 linux 系统里面的一个重要概念：inode。

我们知道，文件存储在硬盘上，硬盘存储的最小单位是扇区（sector，每个扇区 512 B）。而操作系统读取文件时，按块读取（连续的多个扇区），也就是说文件存取的最小单位是块（block，块通常是 4 KB）。

除了文件数据，我们还必须存储文件的元信息（如：文件大小、文件创建者、文件数据的块位置、文件读/写/执行权限、文件时间戳等等），这种存储文件元信息的结构就称为 inode。我们可以使用 `stat` 命令查看文件的 inode 信息。在 Node 中，调用 `fs.stat` 后返回的结果中也有相关信息

![stat](http://img.alicdn.com/tps/TB1IUOkKXXXXXX2aXXXXXXXXXXX-1126-980.png)

每个 inode 都有一个唯一的号码标志，linux 系统内部使用 inode 的号码来识别文件，并不使用文件名。我们打开一个文件时，系统首先找到文件名对应的 inode 号码，然后通过 inode 号码获取 inode 信息，最后根据 inode 信息中的文件数据所在的 block 读出数据。

实际上，在 linux 系统中，目录也是一种文件。目录文件包含一系列目录项，每个目录项由两部分组成：所包含文件的文件名，以及该文件名对应的 inode 号码。我们可以使用 `ls -i` 来列出目录中的文件以及它们的 inode 号码。这其实也解释了仅更改目录的读权限，并不能实现读取目录下所有文件内容的原因，通常需要 `chmod -R` 来进行递归更改。

总结下：
  - 硬盘存取的最小单位是扇区，文件存取的最小单位是块（连续的扇区）
  - 存储文件元信息（文件大小、创建者、块位置、时间戳、权限等非数据信息）的结构称为 inode
  - 每个 inode 拥有一个唯一号码，系统内部通过它来识别文件
  - 目录也是一种文件，其内容包含一系列目录项（每个目录项由文件的文件名和文件对应的 inode 号码组成）

## 硬链接和软链接

### 硬链接

一般情况，一个文件名“唯一”对应一个 inode。但是，linux 允许多个文件名都指向同一个 inode。这表示我们可以使用不同的文件名访问同样的内容；对文件内容进行修改将“反映”到所有文件；删除一个文件不影响另一个文件的访问  。这种机制就被称为“硬链接”。

我们可以使用 `ln source target` 来建立硬链接（注意：`source` 是本身已存在的文件，`target` 是将要建立的链接）。

![hard link](http://img.alicdn.com/tps/TB1Q31gKXXXXXbLaXXXXXXXXXXX-912-438.png)

形象化的表示为下图：

![hard link graph](http://img.alicdn.com/tps/TB1mYuqKXXXXXbGXVXXXXXXXXXX-300-361.png)

需要注意的是，只能给文件建立硬链接，而不能给目录建立硬链接。另外，`source` 文件必须存在，否则将会报错。

删除一个文件为什么不影响另一个文件的访问呢？实际上，文件 inode 中还有一个链接数的信息，每多一个文件指向这个 inode，该数字就会加 1，每少一个文件指向这个 inode，该数字就会减 1，当值减到 0，系统就自动回收 inode 及其对应的 block 区域。很像是一种引用计数的垃圾回收机制。

当我们对某个文件建立了硬链接后，对应的 inode 的链接数会是 2（原文件本身已经有一个指向），当删除一个文件时，链接数变成 1，并没达到回收的条件，所以我们还是可以访问文件。

### 软链接

软链接类似于 windows 中的”快捷方式“。两个文件虽然 inode 号码不一样，但是文件 A 内部会指向文件 B 的 inode。当我们读取文件 A 时，系统就自动导向文件 B，文件 A 就是文件 B 的软链接（或者叫符号链接）。这表示我们同样可以使用不同的文件名访问同样的内容；对文件内容修改将”反映“到所有文件。但是当我们删除掉源文件 B 时，再访问文件 A 时会报错 “No such file or directory”。

我们可以使用 `ln -s source target` 来建立软链接（注意：表示让 `target` “指向”`source`）。

![soft link](http://img.alicdn.com/tps/TB1xOeTKXXXXXabXXXXXXXXXXXX-994-488.png)

形象化的表示为下图：

![soft link graph](http://img.alicdn.com/tps/TB1PhakKXXXXXaSaXXXXXXXXXXX-300-366.png)

和硬链接不同，我们可以给目录建立软链接，这带来许多便利。比如我们有一个模块有很多个版本，分别存放在 1.0.0、2.0.0 这样的目录下面，当更新模块时，只需要建立一个软链接指向最新版本号的目录就能很方便的切换版本。

另外，建立软链接时，`source` 是可以不存在的。这很像一种”运行时“机制，而不是“编译时”机制，建立的时候不报错，等执行的时候发现找不到就报错了。

![danggling soft link](http://img.alicdn.com/tps/TB1UrifKXXXXXXjapXXXXXXXXXX-994-618.png)

### 总结

  - 使用 `ln source target` 建立硬链接；使用 `ln -s source target` 建立软链接
  - 硬链接不会创建额外 inode，和源文件共用同一个 inode；软链接会创建额外一个文件（额外 inode），指向源文件的 inode
  - 建立硬链接时，`source` 必须存在且只能是文件；建立软链接时，`source` 可以不存在而且可以是目录
  - 删除源文件不会影响硬链接文件的访问（因为 inode 还在）；删除源文件会影响软链接文件的访问（因为指向的 inode 已经不存在了）
  - 对于已经建立的同名链接，不能再次建立，除非删掉或者使用 `-f` 参数

### 关于软链接的补充

上面的例子 `ln -s file file-soft` 给我们的感觉像是 `file-soft` 是“凭空”出现的。当我们跨目录来创建软链接时，可能会“幻想”这样的命令也是可以生效的：`ln -s ~/development/mod ~/production/dir-not-exits/mod`。

这里并没有 `~/production/dir-not-exits/` 这个目录，而软链接本质上是一个新的“文件”，所以，我们不可能正确建立软链接（会报错说 “no such file or directory”）。

如果我们先通过 `mkdir` 建立好目录 `~/production/dir-not-exits/`，再进行软链接，即可达到预期效果。

## fs.symlink

在 node 中，我们可以使用方法 `fs.symink(target, path)` 建立软链接（符号链接），没有直接的方法建立硬链接（就算通过子进程的方式直接指向 shell 命令也不能跨平台）。

如果是对目录建立链接，请总是传递第三个参数 `dir`（虽然第三个参数只在 windows 下生效，这可以保证代码跨平台）：`fs.symlink(target, path, 'dir')`。

为啥这个接口的参数会是 `target` 和 `path`。实际上这是一个 linux 的 API，[symlink(target, linkpath)](http://man7.org/linux/man-pages/man2/symlink.2.html)。它是这样描述的：建立一个名为 `linkpath` 的符号链接并且含有内容 `target`。其实就是让 `linkpath` 指向 `target`，和 `ln -s source target` 功能一样，让 `target` 指向 `source`。

是不是有点晕？其实我们只需要明白 `ln -s` 和 `fs.symlink` 后面传递的两个参数顺序是一致的，只是叫法不一样，使用起来也就没那么纠结了：

```bash
ln -s file file-soft # file-soft -> file
ln -s dir dir-soft # dir-soft -> dir
```

```javascript
fs.symlinkSync('file', 'file-soft'); // file-soft -> file
fs.symlinkSync('dir', 'dir-soft', 'dir'); // dir-soft -> dir
```

## require

在 Node 中，我们经常通过 `require` 来引用模块。非常有趣的是，`require` 引用模块时，会“考虑”符号链接，但是却使用模块的真实路径作为 `__filename`、`__dirname`，而不是符号链接的路径。

考虑下面的目录结构：

```bash
- app
  - index.js // require('dep1')
  - node_modules
    - dep1 -> ../../mods/dep1 //符号链接
- mods
  - dep1
    - index.js
```

以及下面的文件内容：

```javascript
// index.js
console.log('index.js', __dirname, __filename);
require('dep1');

// dep1/index.js
console.log('dep1', __dirname, __filename);
console.log(module.paths);
```

执行 `node index.js` 后输出是下面这样：

```bash
index.js /Users/kohpoll/Workspace/test/app /Users/kohpoll/Workspace/test/app/index.js

dep1 /Users/kohpoll/Workspace/test/mods/dep1 /Users/kohpoll/Workspace/test/mods/dep1/index.js
[ '/Users/kohpoll/Workspace/test/mods/dep1/node_modules',
  '/Users/kohpoll/Workspace/test/mods/node_modules',
  '/Users/kohpoll/Workspace/test/node_modules',
  '/Users/kohpoll/Workspace/node_modules',
  '/Users/kohpoll/node_modules',
  '/Users/node_modules',
  '/node_modules' ]
```

我们发现，`index.js` 可以成功的 `require('dep1')`。这很好啊，这让我们调试本地开发中的 npm 模块很方便。我们只需要去 `require` 模块的文件所在的 `node_modules` 下面建立一个符号链接就行了。

但是在模块 `dep1` 中，`__dirname`、`__filename` 都变成了模块实际的路径，更要命的是模块查找路径 `module.paths` 也变成了从实际路径开始查找。

这会带来什么问题？

再考虑下面的目录结构：

```bash
- app
  - index.js // require('dep1')
  - node_modules
    - dep1 -> ../../mods/dep1 // require('dep2')
    - dep2 -> ../../mods/dep2 // 符号连接
- mods
  - dep1
    - index.js
  - dep2
    - index.js
```

以及下面的文件内容：

```javascript
// index.js
console.log('index.js', __dirname, __filename);
require('dep1');

// dep1/index.js
console.log('dep1', __dirname, __filename);
console.log(module.paths);
require('dep2');

// dep2/index.js
console.log('dep2', __dirname, __filename);
console.log(module.paths);
```

当我们再执行 `node index.js` 时，输出是下面这样：

```bash
index.js /Users/kohpoll/Workspace/test/app /Users/kohpoll/Workspace/test/app/index.js

dep1 /Users/kohpoll/Workspace/test/mods/dep1 /Users/kohpoll/Workspace/test/mods/dep1/index.js
[ '/Users/kohpoll/Workspace/test/mods/dep1/node_modules',
  '/Users/kohpoll/Workspace/test/mods/node_modules',
  '/Users/kohpoll/Workspace/test/node_modules',
  '/Users/kohpoll/Workspace/node_modules',
  '/Users/kohpoll/node_modules',
  '/Users/node_modules',
  '/node_modules' ]
  
module.js:339
    throw err;
    ^
Error: Cannot find module 'dep2'
    at Function.Module._resolveFilename (module.js:337:15)
    at Function.Module._load (module.js:287:25)
    at Module.require (module.js:366:17)
    at require (module.js:385:17)
    at Object.<anonymous> (/Users/kohpoll/Workspace/test/mods/dep1/index.js:6:1)
    at Module._compile (module.js:435:26)
    at Object.Module._extensions..js (module.js:442:10)
    at Module.load (module.js:356:32)
    at Function.Module._load (module.js:311:12)
    at Module.require (module.js:366:17)
```

发现了吗？`dep1` 根本就 `require` 不到 `dep2`，因为 `dep2` 不在它的查找路径里面！

关于这个问题，github 上有一个冗长的 issue 在讨论。问题解决起来确实很麻烦，而且会 break 掉一大堆已有功能，所以，最终的结论是在找到更好的方法前给 node v6 增加了一个 `--preserve-symlinks` 选项来禁止这种 `require` 的行为，而是使用全新的 `require` 逻辑。有兴趣和闲情的可以去围观：<https://github.com/nodejs/node/issues/3402>（真的好长......）。

至于全新的 `require` 逻辑会不会有新的坑，在没有具体实践前，我也不知道。

那我们上面的情况有办法解决吗？其实也有，那就是将目录结构调整成下面这样，从而让 `dep2` 能在 `dep1` 的查找路径里面：

```bash
- app
  - index.js // require('dep1')
  - node_modules
    - dep1 -> ../../mods/node_modules/dep1 // 软链接
    - dep2 -> ../../mods/node_modules/dep2 // 软连接
- mods
  - node_modules
    - dep1
      - index.js
    - dep2
      - index.js
```

## 参考链接
  - <http://www.ruanyifeng.com/blog/2011/12/inode.html>
  - <http://www.geekride.com/hard-link-vs-soft-link/>
  - <http://man7.org/linux/man-pages/man2/symlink.2.html>
  - <https://nodejs.org/api/fs.html>
  - <https://github.com/nodejs/node/issues/3402>

本文采用 [知识共享署名 3.0 中国大陆许可协议](http://creativecommons.org/licenses/by/3.0/cn)，可自由转载、引用，但需署名作者且注明文章出处 。
