---
title: 'Vue UI Framework 对比'
date: 2017-04-17 09:15:35
tags: [FrontEnd]
valine: true
---

本文链接: http://mervynfang.com/2017/04/17/201704/Vue-UI-Framework-%E5%AF%B9%E6%AF%94/

Vue 2.0 正式版之后，基于 1.x 的许多 Vue UI 框架都做了更新，以适配 Vue 2.x。当然其中也有一些并没有做升级，仍然只兼容 Vue 1.x。其中包括 vue UI 组件开始得比较早的 [VueStrap](https://github.com/yuche/vue-strap "VueStrap") （基于 bootsrap 的 UI 组件 ），还有 [vue-antd](https://github.com/okoala/vue-antd "vue-antd")（基于 ant-design），[vue-desktop](https://github.com/ElemeFE/vue-desktop "vue-desktop")等。

这篇文章主要选择了 Github 开源社区并且兼容了 Vue 2.x 的 Star 较为多的进行比较，分别为

<!-- more -->

桌面端
- [element](https://github.com/ElemeFE/element "element")（star 8546）
- [Keen-UI](https://github.com/JosephusPaye/Keen-UI "Keen-UI")（star 2209）

移动端
- [vux](https://github.com/airyland/vux "vux")（star 6512）
- [mint-ui](https://github.com/ElemeFE/mint-ui "mint-ui")（star 4464）

其他的也有相对比较好的 UI 框架 比如 muse-ui 等，可以在 [awesome-vue](https://github.com/vuejs/awesome-vue#component-collections "awesome-vue") 查看。

#### UI Framework or Toolkit

所谓 UI 框架，也可以说是 UI 的工具箱，利用已经现成的组件（比如输入框，提示，弹窗等）进行开发，不用去顾及设计规范，交互，api设计，页面展示样式，数据绑定，将大大提高项目的开发效率。虽然现有的 UI 框架所覆盖的组件类型已经非常广了，但是还是会存在高度定制化的组件存在，这时候也可以通过增加组件去扩展。而基于 vue 的 UI 框架更多的是解决了 vue 数据绑定的问题，使得我们可以只通过数据去控制 UI 的展示，而不需要去关注实现细节。

现行的 Vue UI Framework，基本上都做到了 api 简洁清晰，组件丰富，设计精美。在选择框架上，有的时候会考虑众多的因素，比如外观设计，社区的维护力度，或者个人偏好等。接下来将从框架的组件的功能以及技术设计方面浅析对比下 element 和 Keen-UI 以及 vux 和 mint-ui。

#### 组件功能特点完善性

桌面端上，element 是饿了么开发的。在 vue 2.0 还是 beta 版本时候就已经开源了，已经经过社区的一段时间的验证，并且 start 的数量也验证了 element 是一个优秀的 Vue UI 框架。Keen-UI 是社区出品，并没有中文文档，基于 Material Design 设计，较为轻量。

功能上，element 比 keen-UI 较为大，可以自定义外观主题，并且包含了布局的选择，内置了一套图标，总体上比较完善。element 实现了较为多的细节上的动画，交互看起来较为舒服，设计上保持了一致性。keen-ui 则遵循 Material Design 设计，主要实现组件，并不支持主题 icon 等的设置。
二者都有较为丰富的组件，下列是 datepicker 组件的外观对比

element:
<div align=center>![](/assets/static/201704/20170216163733.png)</div>

keen-ui:
<div align=center>![](/assets/static/201704/20170216163748.png)</div>

移动端上，vux 是 vue 与 weui 的结合，因为现有一套视觉规范，所以 vux 的组件的丰富度比较高，功能实现较为完善。并且也与微信原生视觉体验保持一致。而 mint-ui 也是饿了么开发，与 element 的视觉接近，功能点相较于并不是很完善。

vux 有 60 个功能组件，而 mint-ui 只有 30 个左右，所以总体上 vux 要更加强大一点。下面是二者的 swiper 外观对比：

vux:
<div align=center>![](/assets/static/201704/20170216171103.png)</div>

mint-ui
<div align=center>![](/assets/static/201704/20170216171131.png)</div>

可见 vux 实现的功能比 mint-ui 更加多，更加完善。

#### 框架设计及开发

api 设计上，因为这四个框架都已经挺完善的，只是在语义化上以及某些细节上有细微的差别，总体上还是设计得比较合理，通过文档 api 阅读便可快速进行开发。

在文档建设上面，element 做得更加好，每个组件的 example，api，source code 都是在同一个页面下，非常便于查阅，并且支持 jsfiddle 在线运行。很容易进行尝试。
keen-ui 则做得相对比较差，source code 需要跳转到 github 文件查看。

移动端上，vux 文档做得比较全面，从配套的设施到组件文档都有涉及。mint-ui 主要是组件 example 介绍，可以在页面内体验移动端组件，做到了与 element 一致。vux 还有一个包括所有组件的[总站](https://vux.li/demos/v2/?x-page=v2-doc-home#/ "总站")，可以在手机端进行尝试，组件文档页面包括源码和 api，预览还是需要跳转。

引用方式上，四个框架都可以通过 CDN 或者 NPM 引入，并且都可以支持引入部分组件，并不需要都引入整个框架。

i18n 上，除了 keen-ui 外都有中英文文档，keen-ui 只有英文文档。当然，英语好没问题。国内的框架也都支持 i18n。

#### 选择结论

好吧结果，桌面端上，推荐使用 element，这是一个挺完善的项目，支持的功能多，社区繁荣，bug 解决得也快。

移动端上，推荐使用 vux。因为遵循着一套 weui 的规范，会使其看起来更加美观，设计也一致，mint-ui 在一些 logo 上还是会出现冲突。当然，如果产品的 PC 和 移动端要保持一致的话，又要使用社区的 UI 框架的话，那么 element 加 mint-ui 也未尝不是一个好的选择。


### 分享
#### Vue UI Framework 文章

[Vue框架Element的事件传递broadcast和dispatch方法分析](http://www.cnblogs.com/xxcanghai/p/6382607.html?utm_source=tuicool&utm_medium=referral "Vue框架Element的事件传递broadcast和dispatch方法分析")
[iView：一套基于 Vue 的高质量 UI 组件库 ](http://div.io/topic/1833?utm_source=tuicool&utm_medium=referral "iView：一套基于 Vue 的高质量 UI 组件库 ")
[Element 一套优雅的 Vue 2 组件库是如何开发的](https://cinwell.com/post/element/ "Element 一套优雅的 Vue 2 组件库是如何开发的")

#### 本周 Front End 好文 2017-02-17

[下一代 Web 应用模型 —— Progressive Web App](https://huangxuan.me/2017/02/09/nextgen-web-pwa/ "下一代 Web 应用模型 —— Progressive Web App")
[2016年JavaScript领域中最受欢迎的“明星”们](http://www.infoq.com/cn/news/2017/02/JavaScript-2016-star "2016年JavaScript领域中最受欢迎的“明星”们")
[网易和淘宝移动WEB适配方案再分析](https://zhuanlan.zhihu.com/p/25216275?refer=jscss "网易和淘宝移动WEB适配方案再分析")