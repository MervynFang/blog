---
title: '【译】Node.js, TC-39, 和模块'
date: 2017-08-17 11:40:40
tags: [FrontEnd, Translation]
valine: true
---

本文链接: http://mervynfang.com/2017/08/17/201708/Node-js-TC-39-%E5%92%8C%E6%A8%A1%E5%9D%97/

原文标题: Node.js, TC-39, and Modules

作者: James M Snell

原文链接: https://medium.com/hacker-daily/node-js-tc-39-and-modules-a1118aecf95e

这周我第一次参加 TC-39 会议。如果你没听说过 TC-39，这里可以简单解释下，TC-39 是 ECMA（欧洲计算机制造联合会）中 ECMAScript 语言（或者叫大家更为熟知的 Javascript）的标准制定委员会。在 TC-39 上，Javascript 语言标准的很多微小的细节实现和差异被定案，委员会致力于保持 Javascript 的持续发展以及给满足更多开发者的需求。

<!-- more -->

我这周参加 TC-39 会议的理由非常简单：一个 TC-39 定义的最新的 Javascript 语言标准，也就是 Modules，给 Node.js 核心团队带来一些麻烦。我们（这里我特指布拉德利·法里亚斯 推特@bradleymeck）已经试图去找出怎么去实现支持 Node.js 的 ECMAScript 模块（下面用 ESM 代指），才不会带来更多的疑惑和麻烦，使它更有价值。

真正的问题并不在于我们不能按照现在定义的 ES 标准去实现 Node.js 的 ESM，问题在于我们按照标准去做会不符合 Nodejs 开发者的预期，并且给他们带来不好的开发体验。我们非常想要确定 ESM 的实现能够是最佳优化和可用性强的。因为这个问题比较复杂，和 TC-39 的成员坐下来一起面对面讨论相信会是最有效的方式。幸运的是，我觉得我们取得了显著的进展。

为了使读者能够清楚问题所在，并且将要怎么解决，这里我花点时间解释下那个我们最关心的问题的基本所在。

首先，一个警告：接下来的内容对于透过代码里面详细的解释会比较简单，这里是因为这篇文章主要是提供一个综述，而不是一篇关于模块系统的深度剖析的论文。

其次，另一个警告：所有东西都是我对于 TC-39 这次会议的理解，很有可能我说的某些细节是错的，因为随着会议的进行，事情的进展可能会变得跟我这篇文章描述的不一样，这真的很有可能。我写这篇文章只是为了提供会议上被讨论的的东西的记录。

### ECMSScript 模块 vs CommonJS：或者说...什么是模块？

实际上，Node.js 和 TC-39 在什么是模块，怎么定义模块，模块怎么写进内存，模块怎么使用上面有非常不一样的意见。

在 Node.js 刚开始的时候，有一个从非常宽松定义的标准——叫 CommonJS，Node.js 的模块系统就是遵循它的标准的。

<div align=center>![](/assets/static/201708/1.png)</div>

简要地说就是一个 JS 文件 export 的标识，比如函数或者变量，将可以被另外一个 JS 文件使用。在 Node.js 里面，这可以通过使用 require() 函数完成。当 Node.js 调用一个类似 require('foo') 的时候，有一个非常特殊的队列步骤在进行。

<div align=center>![](/assets/static/201708/2.png)</div>

第一步是把标识符 'foo' 转换成 Node.js 能够读懂的绝对文件路径。这里的处理过程包括多步内部步骤，本质上就是去扫描本地文件系统去匹配对应的原生模块，JS 文件，JSON 文档。这个解析处理步骤的结果是产生一个 'foo' 指定文件的能够被 Node.js 加载和使用的绝对文件路径。

接下来，加载完全取决于解析步骤产生的绝对文件路径指向什么。例如，如果解析地址指向的是 Node.js 原生模块，那就开始加载代码，这中间包括动态引入应用的 Node.js 共享库进入当前的 Node.js 进程。如果指向的是一个 JSON 文件或者 JS 文件，确认文件存在之后，文件的内容被读进内存里。这里应该重点注意加载 JS 代码和执行 JS 代码并不一样。前者只是严格的拉取文件的字符串内容进入内存，后者是将字符串传递给 JS 引擎解析和执行。

如果加载的是一个 JS 文件，Node.js 现在会假定文件是一个 CommonJS 模块。接下去，Node.js 做的是饱受争议并且许多 Node.js 应用的开发者也经常误解的。在把加载完的 JS 字符串传递给 JS 引擎解析之前，JS 代码被包裹在一个函数里面。

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

Node.js 使用 JS 运行环境来执行 JS 代码。很多全局的变量，比如 'exports', 'module', '\_\_filename', '\_\_dirname'，在 Node.js 常常被使用的，其实在 JS 的运行环境中并不是全局变量。当函数被调用时，Node.js 将这些函数作为参数传入包裹的函数。

包裹的函数本质上是一个工厂函数，exports 对象是一个普通的 JS 对象，包裹函数把对象和属性与 exports 函数连接起来。一旦包裹的函数产生返回值，exports 对象做了缓存，然后把值返回给 `require()` 函数。

理解这个讨论的关键概念在于，没有办法提前检测 CommonJS 模块 export 什么东西出来，直到包裹函数被执行后，才能直到确切是什么内容。

这是 CommonJS 模块和 ECMAScript 模块之间的显著区别。因为 CommonJS 模块的 `exports` 是在包裹函数被执行的时候定义的，而 ESM 的 exports 是语法上的定义。这意味着，当 JS 代码解析的时候，ESM 的 `exports` 标识已经被解析，就已经识别为 ESM，不用等到代码被执行的时候才能被判断。

例如，下面的这个简单的 ECMAScript 模块：

```javascript
export const m = 1;
```

当这段代码被解析的时候，在执行之前，就创建了一个内部的结构叫模块记录。在这个模块记录里面，记录着一些关键的数据，包括一个记录着模块 export 的标识的静态的列表。JS 解析器会通过寻找 `export` 关键字去识别。由于缺乏更好的表述，这里就说标识吧。模块记录的标识实质上指向的东西是还不存在的，因为代码还没有执行，只有模块记录建立好之后，模块的代码才真正的开始执行。这里面其实有很多细节，但我还是比较模糊不清，这里关键点就是，ESM 会在执行前去检测 export 出来的标识。

当使用 ECMAScript 模块时，它使用了一个 import 的语句：

```javascript
import {m} from “foo”;
```

这段代码主要讲，“我将要使用foo模块输出的m标识”。

这个语句是一个词法定义语句，当代码解析的时候，它用来建立 import 所在的脚本和 ‘foo’ 模块之间的连接。现在的ECMAScript 模块标准写着，这个连接必须在任何代码执行前生效，这意味着，模块的实现必须确保 ‘foo’ 模块确实有输出标识 ‘m’，这样两个 JS 文件才能被执行。

对于熟悉基于强对象的面向对象编程语言比如 Java 或者 C++ 的人来说，对模块的这么处理应该会觉得容易理解，因为它很想通过一个接口去访问对象。模块输出的标识会在执行前被确认和连接，如果有标识实现不完整，有错误，就会在执行阶段被抛出。

对于 Node.js 来说，这里有一个挑战，因为 ‘foo’ 模块并不是一个 ECMAScript 模块，是通过词法去定义 export 的，它是一个 CommonJS 模块，是动态去定义 export 的。具体来说，这里当我说，`import {m} from “foo”`，ECMAScript 模块现在需要在执行前检测 ‘foo’ 模块是否有输出标识 ‘m’。当时，就像我们知道的，因为 ‘foo’ 是一个 CommonJS 模块，这是不可能提前去检测的，因为执行后 m 才被输出。最终的结果就是，在现在定义的 ESM 标准下，ECMAScript 模块的很重要的一项功能，`import` 和 `export`，并不能 import CommonJS 模块。

这对于 Node.js 的开发者来说并不是很理想。所以我们回去 TC-39 问下是否能够在标准上面做些改变。起初，我们其实是有点害怕去问这个问题。但实际上，TC-39 很关心，并且也在标准上面正在做一些改表，以确保 Node.js 能够很好地实现 ECMAScript 模块，使得东西在 Node.js 环境下面运行得很好。

### 操作顺序

一个被提议的确切的改变是对动态定义的模块做出说明。本质上，当我做出 `import {m} from “foo”` 这样的操作时，结果应该是返回 ‘foo’ 模块不是一个拥有 export 词法定义的 ECMAScript 模块，而不是像现在标准里面做的，直接抛出错误结束。进程应该把 ‘foo’ 模块和导入脚本进入一个待定状态，延迟验证动态模块导入的标识直到动态模块的代码能够被执行。一旦执行了代码，模块记录中关于 CommonJS 模块的就能完成，并且导入的链接也生效。这个 ECMAScript 模块标准的改进允许 `export` 和 `import` 的 CommonJS 模块能够工作。（虽然，这里关于循环依赖的边际案例还是会有一些坑存在）

我们通过一些例子来详细说明。

我有一个应用依赖于 ECMAScript 模块 A，A 模块依赖于 CommonJS 模块 B。

<div align=center>![](/assets/static/201708/3.png)</div>

我的应用（myapp.js）的代码是

```javascript
const foo = require('A').default
foo()
```

A 模块的代码是

```javascript
import {log} from "B"
export default function() {
  log('hello world')
}
```

B 模块的代码是

```javascript
module.exports.log = function(msg) {
  console.log(msg);
}
```

当我运行 `node myapp.js` 的时候，调用 `require(A)` 会检测到加载的是一个 ECMAScript 模块（后面会解释这个是怎么完成的）。与 CommonJS 模块现在使用的包裹函数去加载模块不同，Node.js 会使用 ECMAScript 模块的标准去解析，初始化，和执行 A 模块。当 A 的代码解析后，就产生了模块记录，它会检测 B 不是一个 ECMAScript 模块，所以校验的步骤在校验 B 导出的日志之后，就会进入待定状态。ECMAScript 模块加载器就会开始它的执行阶段。这个阶段将首先使用现有的 CommonJS 包裹函数来执行 B 模块，其执行结果将传递回 ECMAScript 模块加载器，以完成模块记录的构造。其次，它将会根据完整的模块记录来执行 A 模块的代码。

关于交换依赖的顺序。就是 A 模块是一个 CommonJS 模块而 B 模块是一个 ECMAScript 模块。这里的话一切都工作正常，就像上面有插图的例子一样，ECMAScript 模块能够去 `require()` CommonJS 模块。

在绝大多数的使用场景下，这个加载模型是能够很好地运行的。但模块之间又循环依赖的时候，这里就开始有点麻烦了。那些之前使用过 CommonJS 模块循环依赖的人知道，当那些模块加载的顺序不同时有出现一些诡异的边际场景。同样的问题会在 CommonJS 模块和 ECMAScript 模块之间循环依赖的时候存在。

<div align=center>![](/assets/static/201708/4.png)</div>

myapp.js 的代码保留和之前一样。但是，A 模块依赖于 B 模块，B 模块也依赖于 A 模块。

A 模块的代码是

```javascript
const b = require('B')
exports.b = b.foo()
exports.a = 1
```

B 模块的代码是

```javascript
module.exports.log = function(msg) {
  console.log(msg);
}
```

当然这是一个相当特殊的人为举例的场景，主要是为了说明问题所在。这种循环变得非常不可能实现。当 ECMAScript 模块 B 被连接和执行后，`"a"` 标识仍然没有被定义和被 CommonJS 模块 A 导出。这种场景下应该被处理成一个引用错误。

但是，如果我们把 B 模块换成下面的代码：

```javascript
import A from “A”
export foo () => A.a
```

这个循环依赖就能够正常工作，因为当一个 CommonJS 模块被使用一个 `import` 语句导入是，`module.exports` 变成 `default` 导出。这个场景下，ECMAScript 模块连接的是导出的 `default` 而非一个标识。

更简洁地说，指定命名标识的从 CommonJS 模块导入只有在 ECMAScript 模块和 CommonJS 模块之间没有循环依赖的时候才能正常工作。

另外一个由于 CommonJS 模块和 ECMAScript 模块之间不同导致的限制是，任何 CommonJS 模块在初始执行后的变化对于一个指定命名标识的导入来说并不可用。举个例子，加入 ECMAScript 模块 A 依赖于 CommonJS 模块 B。

<div align=center>![](/assets/static/201708/5.png)</div>

假设 B 的代码如下

```javascript
module.exports.foo = function(name, key) {
  module.exports[name] = key
}
```

当模块 B 被 模块 A 导入时，唯一导出的能够用来做命名引用的有效标识就是 `default` 和 `foo`。没有其他的标识会被加到 `module.exports` 上，当调用函数 foo 后，命名引用就开始生效了。它们通过 `default` 导出的话就会一直生效了。就像下面的代码一样，就会正常地工作了。

```javascript
import {foo} from “B”
import B from "B"
foo("abc", 123)
if (B.abc === 123) { /** ... **/ }
```

### require() vs import

`require()` 和 `import` 之间有一个非常清晰的区别：就是我们可以用 `require()` 来加载一个 ECMAScript 模块，也可以用 `import` 来导入一个 CommonJS 模块，但是却不可以在 CommonJS 模块里面用 `import`，同样的，`require()` 也默认不能在 ECMAScript 模块里面使用。

换句话说，如果我有一个 CommonJS 模块 A，那么下面的代码就是不能执行的，因为 `import` 语句不能再 CommonJS 模块里面使用：

```javascript
const b = require(‘B’)
import c from "C"
```

如果你在 CommonJS 模块里面操作，正确的方式就是使用 `require` 去加载和使用一个 ECMAScript 模块：

```javascript
const b = require(‘B’)
const c = require('C')
```

在一个 ECMAScript 模块里面，只有在特别地从原生导入之后，`require()` 才能生效和被使用。导入 `require()` 必须检测到特定的标识符，但是本质上它一般是这样的东西：

```javascript
import {require} from “nodejs”
require(“foo”)
```

但是，因为能够直接通过 `import` 去导入一个 CommonJS 模块，所以只有非常少的情况下会去这么做。

另外一点：Node.js 成员有其他的一些想法，例如加载一个 ECMAScript 模块是否必须是异步的，因为这样会要求在整个依赖图表里面使用 Promise。TC-39 向我们保证（包括上面描述的改变也允许）加载不必须是异步的。这是一件好事。

**import() 怎么样**

在 TC-39 会议之前有一个提议提出来了，就是引进一个新的 `import()` 函数。这个就跟上面的例子里面的 `import` 语句展示的明显不一样了。看下下面的例子：

```javascript
import {foo} from “bar”
import(“baz”).then((module)=>{/*…*/}).catch((err)=>{/**…*/})
```

第一个 `import` 语句是词法上的定义。就像之前说的，它在代码解析的时候就已经被处理和确认。而 `import()` 函数不同，则是在运行代码的时候才被处理。它也可以 `import` ECMAScript 模块（或者 CommonJS 模块），但是，像现在 Node.js 里面的 `require()` 方法一样，完全在代码运行的时候执行。但不像 `require()` 函数，`import()` 函数返回一个 `Promise`，允许（但不要求）加载相关的模块完全异步地进行。

由于 `import()` 函数返回一个 `Promise`，像 `await import('foo')` 这样的代码就可能出现了。但是，很重要的是 `import()` 在 TC-39 内还远远没有完整，现在也还不成熟。同时，Node.js 能否完全实现使用 `import()` 函数异步加载模块也还不是很清楚。

### 检测CommonJS vs. ESM

在代码是否使用 `require()`, `import`, `import()` 来加载模块前，很重要的是，要能够先检测导入的是什么类型的模块，这样 Node.js 才能够知道恰当的加载和处理它的方式。

按照惯例，Node.js `require()` 函数的实现是依赖于文件扩展名去区分怎么加载不同类型的文件。例如，\*.node 文件被加载成原生模块，\*.json 文件简单地通过 `JSON.parse` 传递，\*.js 文件则被处理成 CommonJS 模块。

根据 ECMAScript 模块的介绍，需要一种机制来区分 CommonJS 模块和 ECMAScript 模块。这里有几种建议的方法。

一种方式是确保一个 JS 文件能够被清楚地解析成或者一个 ECMAScript 模块或者其他的东西，换句话说，当我解析一段 JS，它是不是一个 ESM 应该是非常显而易见的，这种方法叫做“无歧义文法”。不幸的是，它要实现会有点棘手。

另一种被考虑的方式是在 `package.json` 文件里面添加元数据。如果 `package.json` 文件里有一些特定的值，那么它将被加载成 ECMAScipt 模块，而不是 CommonJS 模块。

第三种方式是使用一种新的文件扩展名（\*.mjs）来认定是 ECMAScript 模块。这种方式是最贴近于 Node.js 现在已经做的事情的。

例如，假设我有一个应用的代码 `myapp.js` 和一个定义在分离的文件的 ECMAScript 模块。

使用无歧义文法的方式，则 Node.js 需要能够去解析第二个文件的 JS 代码，并且自动判定它是使用 ECMAScript 模块。在这种方式中，ECMAScript 模块文件能够使用 \*.js 文件扩展名，然后也能正常地运行。就像我说的，无歧义文法想要做的正确有点棘手，还有很多边际场景使得它很难实现。

使用 `package.json` 的方式，则 ECMAScript 模块必须被打包在它自己的目录里面（本质上是它自己的包）或者包的根目录必须有一个包含着一些元数据的 `package.json` 文件，元数据需要能够指明哪些 JS 包含着 ECMAScript 模块实际上是一个 ECMAScript 模块。这种方式不是很理想，因为需要对 `package.json` 文件做附加的操作。

使用 \*.mjs 文件扩展名的方式，ECMAScript 模块的代码会被放进一个特定的文件例如 `foo.mjs`。在 Node.js 解析标识符成为文件的绝对路径后，
它会开始查找文件的扩展名，就像现在它对原生扩展和 JSON 文件处理的一样。如果它判断是 \*.mjs 文件扩展名，它就知道当做一个 ECMAScript 模块加载和处理。如果它判断是 \*.js，就会降级处理加载成一个 CommonJS 模块。

### 幂等性的考量

一般来说，重复调用 `require('foo')` 多次会返回完全相同的模块的实例。但是，返回的对象不是不可变的，它可以被其他模块更新，被一些方法或者标识补丁替换，甚至被替换整个函数。这种事情现在在 Node.js 生态圈中非常普遍存在。

例如，假设 myapp.js 有两个依赖 A 和 B。它们都是 CommonJS 模块。A 为了扩展它也依赖于 B。

myapp.js 的代码是：

```javascript
const A = require('A')
const B = require('B')
B.foo()
```

A 的代码是：

```javascript
const B = require(‘B’)
const foo = B.foo
B.foo = function() {
  console.log('intercepted!')
  foo()
}
```

B 的代码是：

```javascript
module.exports.foo = function() {
  console.log('foo bar baz')
}
```

在这个案例里面，在 A 里面调用 `require('B')` 返回与 myapp.js 里面调用 `require('B')` 不一样的结果。

如果使用 ECMAScript 模块，这种模块间的互相干扰也不是很容易解决。理由是双重的：首先，引用是在运行前完成连接的；其次，引用需要是幂等性的——即在一个给定的上下文中，每次调用都会返回相同的不可变的标识。在实际情况下，这个意味着，当使用命名引用是，ES 模块 A 不能轻易地去改变 ES 模块 B。

这个规则对下面的 myapp.js 的代码有同样的影响：

```javascript
const B = require('B')
const foo = B.foo
const A = require('A')
foo()
```

这里，A 模块依然更新 B 模块的函数 foo，但因为 foo 的引用已经在那个更新之前捕获了，调用函数 `foo()` 就是调用原来的函数而不失更新后的那个。在一个 ES 模块里面，没有办法能够引用能够被 A 更新过的模块 B。

这个幂等性的规则导致的问题还有很有其他各种各样的场景。mock，APM 和 为了测试的监控是最主要的例子。幸运的是，有很多方式能够去处理这种局限性。一种方式是增加加载阶段的钩子函数，允许一个 ES 模块的导出能够被包裹。另一个是让 TC-39 允许加载后的 ES 模块能够被替换。有几种机制在这里被考虑。好消息是拦截 ES 模块跟拦截 CommonJS 模块不一样，ES 模块能够被拦截。

### 还有很多要做

还有一大堆工作要去做，上面讨论的任何东西在任何情况下都还不是最终定下来的。很多细节都要去解决，事情到最后看起来很可能会非常一样。重要的是 Node.js 和 TC-39 现在一起工作去解决所有的这些问题，这是在正确的方向上迈出的优秀的受欢迎的一步。

---

**译者按**

原文写于 2016 年 9 月份，今年 4 月份原文作者又写了一篇文章关于 Node.js 里的 ES 模块的一次更新——[An Update on ES6 Modules in Node.js](https://medium.com/the-node-js-collection/an-update-on-es6-modules-in-node-js-42c958b890c)。当时译者准备对此文进行翻译但是凹凸实验室已经做了很好的翻译了，特此附上链接——[【译】关于 Node.js 里 ES6 Modules 的一次更新说明](https://aotu.io/notes/2017/04/22/an-update-on-es6-modules-in-node-js/)。
