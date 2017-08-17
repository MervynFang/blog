---
title: '【译】Node.js, TC-39, 和模块'
date: 2017-08-17 11:40:40
tags: [FrontEnd, Translation]
---
本文链接: http://mervynfang.github.io/blog/2017/04/17/Node-js-TC-39-%E5%92%8C%E6%A8%A1%E5%9D%97/

原文标题: Node.js, TC-39, and Modules

作者: James M Snell

原文链接: https://medium.com/hacker-daily/node-js-tc-39-and-modules-a1118aecf95e

这周我第一次参加 TC-39 会议。如果你没听说过 TC-39，这里可以简单解释下，TC-39 是 ECMA（欧洲计算机制造联合会）中 ECMAScript 语言（或者叫大家更为熟知的 Javascript）的标准制定委员会。在 TC-39 上，Javascript 语言标准的很多微小的细节实现和差异被定案，委员会致力于保持 Javascript 的持续发展以及给满足更多开发者的需求。

<!-- more -->

我这周参加 TC-39 会议的理由非常简单：一个 TC-39 定义的最新的 Javascript 语言标准，也就是 Modules，给 Node.js 核心团队带来一些麻烦。我们（这里我特指布拉德利·法里亚斯 推特@bradleymeck）已经试图去找出怎么去实现支持 Node.js 的 ECMAScript 模块（下面用 ESM 代指），才不会带来更多的疑惑和麻烦，使它更有价值。

真正的问题并不在于我们不能按照现在定义的 ES 标准去实现 Node.js 的 ESM，问题在于我们按照标准去做会不符合 Nodejs 开发者的预期，并且给他们带来不好的开发体验。我们非常想要确定 ESM 的实现能够是最佳优化和可用性强的。因为这个问题比较复杂，和 TC-39 的成员坐下来一起面对面讨论相信会是最有效的方式。幸运的是，我觉得我们取得了显著的进展。

为了使读者能够清楚问题所在，并且将要怎么解决，这里我花点时间解释下那个我们最关心的问题的基本所在。

首先，一个警告：接下来的内容对于投过代码里面详细的解释会比较简单，这里是因为这篇文章主要是提供一个综述，而不是一篇关于模块系统的深度剖析的论文。

其次，另一个警告：所有东西都是我对于 TC-39 这次会议的理解，很有可能我说的某些细节是错的，因为随着会议的进行，事情的进展可能会变得跟我这篇文章描述的不一样，这真的很有可能。我写这篇文章只是为了提供会议上被讨论的的东西的记录。

### ECMSScript 模块 vs Commonjs：或者说...什么是模块？

实际上，Node.js 和 TC-39 在什么是模块，怎么定义模块，模块怎么写进内存，模块怎么使用上面有非常不一样的意见。

在 Node.js 刚开始的时候，有一个从非常宽松定义的标准——叫 Commonjs，Node.js 的模块系统就是遵循它的标准的。

<div align=center>![](/blog/assets/static/201708/1.png)</div>

简要地说就是一个 JS 文件 export 的标识，比如函数或者变量，将可以被另外一个 JS 文件使用。在 Node.js 里面，这可以通过使用 require() 函数完成。当 Node.js 调用一个类似 require('foo') 的时候，有一个非常特殊的队列步骤在进行。

<div align=center>![](/blog/assets/static/201708/2.png)</div>

第一步是把标识符 'foo' 转换成 Node.js 能够读懂的绝对文件路径。这里的处理过程包括多步内部步骤，本质上就是去扫描本地文件系统去匹配对应的原生模块，JS 文件，JSON 文档。这个解析处理步骤的结果是产生一个 'foo' 指定文件的能够被 Node.js 加载和使用的绝对文件路径。

接下来，加载完全取决于解析步骤产生的绝对文件路径指向什么。例如，如果解析地址指向的是 Node.js 原生模块，那就开始加载代码，这中间包括动态引入应用的 Node.js 共享库进入当前的 Node.js 进程。如果指向的是一个 JSON 文件或者 JS 文件，确认文件存在之后，文件的内容被读进内存里。这里应该重点注意加载 JS 代码和执行 JS 代码并不一样。前者只是严格的拉取文件的字符串内容进入内存，后者是将字符串传递给 JS 引擎解析和执行。

如果加载的是一个 JS 文件，Node.js 现在会假定文件是一个 Commonjs 模块。接下去，Node.js 做的是饱受争议并且许多 Node.js 应用的开发者也经常误解的。在把加载完的 JS 字符串传递给 JS 引擎解析之前，JS 代码被包裹在一个函数里面。

例如，一个 JS 文件 'foo.js'：

```javascript
const m = 1;
module.exports.m = m;
```

事实上会被 Node.js 解析成为如下的函数：

```javascript
function (exports, require, module, __filename, __dirname) {
  const m = 1;
  module.exports.m = m;
}
```


