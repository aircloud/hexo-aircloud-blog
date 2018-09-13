---
title: 一篇关于react历史的流水账
date: 2018-06-10 15:54:14
tags:
	- react
---

react 目前已经更新到 V16.3，其一路走来，日臻完善，笔者接触 react 两年有余，在这里做一个阶段性的整理，也对 react 的发展和我对 react 的学习做一个整体记录。

笔者是在 16 年初开始关注 react，而实际上那个时候 react 已经发布快三年了， 16 年初的我写页面还是主要使用 backbone.js、Jquery，并且认为，相比于纯粹使用 Jquery 的“刀耕火种”的时代，使用 backbone.js 已经足够方便并且不需要替代品了。

这篇文章会从 react 开源之初进行讲起，直到 2018 年六月。

### 为什么是 react

我们知道，react 并不是一个 MVC 框架，也并没有使用传统的前端模版，而是采用了纯 JS 编写（实际上用到了 JSX ），使用了虚拟 DOM，使用 diff 来保证 DOM 的更新效率，并且可以结合 facebook 的 Flux 架构，解决传统 MVC 模式的一些痛点。

在 react 开源之初，相关生态体系并不完善，甚至官方还在用`Backbone.Router`加 react 来开发单页面应用。

但是那个时候的 react，和现在的 react，解决的核心问题都没有变化，那就是**复杂的UI渲染问题（ complex UI rendering ）**，所有的它的组件化，虚拟 DOM 和 diff 算法，甚至目前提出的 Fiber、async rendering等等，都是围绕这个中心。

### FLUX

在 2014 年五月左右，也就是距离 react 开源接近一年时间，react 公开了 FLUX 架构。当然，我们现在在学习的过程中，甚至都很难听到 FLUX 这个词汇了，更多的则是 redux 甚至 dva 等更上层的框架，但是目前绝大多数 react 相关的数据管理框架都受到了 FLUX 很大启发。

FLUX 和双向数据绑定的关系，我认为这里有必要援引当初官方写的一点解释（更详细的一些信息，可以看[这篇文章](https://www.10000h.top/react_flux.pdf)）：

```
To summarize, Flux works well for us because the single directional data flow makes it easy to understand and modify an application as it becomes more complicated. We found that two-way data bindings lead to cascading updates, where changing one data model led to another data model updating, making it very difficult to predict what would change as the result of a single user interaction.

总而言之，Flux对我们来说效果很好，因为单向数据流可以让应用程序变得更加复杂，从而轻松理解和修改应用程序。我们发现双向数据绑定导致级联更新，其中更改一个数据模型导致另一个数据模型更新，使得很难预测单个用户交互的结果会发生什么变化。
```

从此之后，下面这张图便多次出现在官方博客和各个网站中，相信我们也肯定见过下图：

![](https://www.10000h.top/images/flux.png)

### react-router

2014年8月，react-router 的雏形发布，在其发布之前，不少示例应用还在使用 backbone
.js 的 router，而 react-router 的发布，标志着 react 生态的进一步成熟。

### react ES6 Class

实际上，在 2015.01.27 之前，我们都是在使用 `React.createClass`来书写组件。

而在 2015.01.27 这一天，也就是第一届 `reactjs conf` 的前一天，react 官方发布了 React V0.13.0 beta 版本。这一个版本的最大更新就是支持 ES6 的 Class 写法来书写组件，同时也公布了比如 propTypes 类型检查、defaultProps、AutoBind、ref 等一系列相关工作在 ES6 Class 模式下的写法。

这次发布是 react 开源至此最为重大的一次更新，也因此直接将 react 的写法进行了革新，在我看来，这标志着 react 从刀耕火种的原始时代进入了石器时代。

*实际上，直到一个半月后的 03.10 ，V0.13 的正式版本才发布。*

而在之后的 V15.5 版本（2017年4月发布），react 才将`React.createClass`的使用设置为 Deprecation，并且宣布会在将来移除该 API，与此同时，react 团队仍然提供了一个单独的库`create-react-class` 来支持原来的 `React.createClass` 功能。

### Relay & GraphQL

在 2015 年的 2月，Facebook 公布了 GraphQL，GraphQL 是一种新的数据查询解决方案，事实证明，它是非常优秀的一个解决方案，到现在已经基本在行业内人尽皆知。

而 Relay 则是链接 react 和 GraphQL 的一个解决方案，有点类似 redux（但是 stat 数只有 redux 的四分之一左右），但是对 GraphQL 更为友好，并且在缓存机制的设计（按照 Graph 来 cache）、声明式的数据获取等方面，有一些自己的独到之处。

当然，我们使用 redux 配合相关插件，也可以不使用 Relay。


### React Native

在第一届 React.js Conf 中，react 团队首次公开了 React Native，并且在3月份真正开源了 React Native（实际上这个时候安卓版本还并不可用），之后在2015年上半年，相关团队陆陆续续披露了关于 React Native 发展情况的更多信息。

并且也是在这个时候（2015年3月），react 团队开始使用 **learn once, write anywhere** 这个如今我们耳熟能详的口号。

### react & react-dom & babel

在2015年七月，官方发布了React v0.14 Beta 1，这也是一个变动比较大的版本，在这个版本中，主要有如下比较大的变化:

* 官方宣布废弃 react-tools 和 JSTransform，这是和 JSX 解析相关的库，而从此 react 开始使用 babel，我认为这对 react 以及其使用者来说无疑是一个利好。
* 分离 react 和 react-dom，由于 React Native 已经迭代了一段时间，这个分离同时也意味着 react 之后的发展方向，react 本身将会关注抽象层和组件本身，而 react-dom 可以将其在浏览器中落地，React Native 可以将其在客户端中落地，之后也许还会有 react-xxx ...

将 react 和 react-dom 分离之后，react 团队又对 react-dom 在 dom 方面做了较为大量的更新。

### Discontinuing IE 8 Support

在 react V15 的版本中，放弃了对 IE 8 的支持。


### Fiber

react 团队使用 Fiber 架构完成了 react V16 的开发，得益于 Fiber 架构，react 的性能又得到了显著提升（尤其是在某些要求交互连续的场景下），并且包大小缩小了 32%。

到目前来说，关于 Fiber 架构的中英文资料都已经相当丰富，笔者在这里就不进行过多的赘述了。

### 接下来的展望

react 团队目前的主要工作集中在 async rendering 方面，这方面的改进可以极大提升用户交互体验（特别是在弱网络环境下），会在 2018 年发布。

如果你对这方面的内容很感兴趣，不妨看看 react 之前的[演讲视频](https://reactjs.org/blog/2018/03/01/sneak-peek-beyond-react-16.html)

### 附录1 一些你可能不知道的变化

* react并非直接将 JSX 渲染成 DOM，而是对某些事件和属性做了封装（优化）。 react 对表单类型的 DOM 进行了优化，比如封装了较为通用的 onChange 回调函数，这其中需要处理不少问题，react 在 V0.4 即拥有了这一特性，可以参考[这里](https://reactjs.org/blog/2013/07/23/community-roundup-5.html#cross-browser-onchange)
* 事实上，react 在V0.8之前，一直在以“react-tools”这个名字发布，而 npm 上面叫做 react 的实际上是另外一个包，而到 V0.8 的时候，react 团队和原来的 “react” 包开发者协商，之后 react 便接管了原来的这个包，也因此，react并没有 V0.6 和 V0.7，而是从 V0.5 直接到了 V0.8
* react 从 V0.14 之后，就直接跳跃到了 V15，官方团队给出的理由是，react 很早就已经足够稳定并且可以使用在生产版本中，更改版本的表达方式更有助于表示 react 项目本身的稳定性。

### 附录2 一些比较优秀的博客

* 关于React Components, Elements, 和 Instances，如果你还有一些疑问，可以看一看React官方团队的文章：[React Components, Elements, and Instances](https://reactjs.org/blog/2015/12/18/react-components-elements-and-instances.html)
* 如果你倾向于使用 mixins，不妨看看 react 关于取消 mixin的说法：[Mixins Considered Harmful](https://reactjs.org/blog/2016/07/13/mixins-considered-harmful.html)
* react props 相关的开发模式的建议，我认为目前在使用 react 的程序员都应该了解一下[You Probably Don't Need Derived State](https://reactjs.org/blog/2018/06/07/you-probably-dont-need-derived-state.html)