---
title: 使用 Node.js 打造多用户实时监控系统
date: 2018-10-21 17:15:38
tags:
    - Node.js
    - javascript
    - Rx.js
---

### 背景概述

首先描述一下笔者遇到的问题，我们可以设定这样一个场景：现在有一个实时监控系统的开发需求，要求同时支持多个用户（这里我们为了简化，暂时不涉及登陆态，假定一个设备即为一个用户），对于不同的用户来讲，他们需要监控的一部分内容是完全相同的，比如设备的 CPU 信息、内存信息等，而另外一部分内容是部分用户重叠的，比如对某一区域的用户来说某些监控信息是相同的，而还有一些信息，则是用户之间完全不同的。

对于每个用户来讲，当其进入页面之后即表明其开始监控，需要持续地进行数据更新，而当其退出界面或者手动点击停止监控，则停止监控。

### 问题描述

实际上，对于以上情况，我们很容易想到通过 WebSocket，对不同的用户进行隔离处理，当一个用户开始监控的时候，通过函数来逐个启动其所有的监控项目，当其停止监控的时候，取消相关监控，并且清除无关变量等。我们可以将所有内容写到 WebSocket 的连接回调中，由于作用域隔离，不同用户之间的监控（读操作）不会产生互相影响。

这种方式可以说是最为快捷方便的方式了，并且几乎无需进行设计，但是这样有一个非常明显的效率问题：

由于不同用户的部分监控项目是有重叠的，对于这些重叠的项目，我们如果对于每一个用户都单独监控，那么就会产生非常多的浪费，如果这些监控中还涉及到数据库交互或者较为复杂的计算，那么成倍之后的性能损失是非常难以承受的。

所以，我们需要将不同用户重叠的那些监控项目，进行合并，合并成一个之后，如果有新的消息，我们就推到所有相关用户的回调函数中去处理。

也就是说，我们需要管理一个一对多的订阅发布模式。

到这里，我们发现我们想要实现这样一个监控系统，并不是非常简单，主要有下列问题：

* [1]对于可能有用户重叠的监控项目，我们需要抽离到用户作用域之外，并且通过统计计数等方式来"记住"当前所有的监控用户，当有新内容时推到各个用户的处理函数中，并且当最后一个用户取消监控的时候要及时清理相关对象。
* [2]不同用户的重叠监控项目的监控方式也各不相同，有的是通过 `setInterval` 等方式的定时任务，有的是事件监听器等等。
* [3]判断不同用户的项目是否重叠也有一定的争议，比如假设不同用户端监控的是同一个项目，调用的也是相同的函数，但是由于用户 ID 不同，这个时候我们如何判断是否算"同一个监控"？

以上的这些问题，如果我们不借助现有的库和工具，自己顺着思路一点点去写，则很容易陷入修修补补的循环，无法专注监控本身，并且最后甚至在效率上适得其反。

### 解决方案

以下解决方案基于 Rx.js，需要对 [Observable](https://cn.rx.js.org/class/es6/Observable.js~Observable.html) 有一定了解。

#### 多个用户的监控以及取消

[Monitor-RX](https://github.com/aircloud/monitor-rx) 是对以上场景问题的一个解决方案封装，其利用了 Rx.js 对订阅发布的管理能力，可以让整个流程变的清晰。

在 Rx.js 中，我们可以通过以下方式建立一个多播对象 `multicasted`：

```
var source = Rx.from([1, 2, 3]);
var subject = new Rx.Subject();
var multicasted = source.pipe(multicast(subject)).refCount();
// 其属于 monitor-rx 的实现细节，无需理解亦可使用 monitor-rx

subscription1 = refCounted.subscribe({
    next: (v) => console.log('observerA: ' + JSON.stringify(v))
});

setTimeout(() => {
    subscription2 = refCounted.subscribe({
        next: (v) => console.log('observerB: ' + JSON.stringify(v))
    });
}, 1200);

subscription1.unsubscribe();
setTimeout(() => {
    subscription2.unsubscribe();
    // 这里 refCounted 的 unsubscribe 相关清理逻辑会自动被调用
}, 3200);
```

在这里采用多播，有如下几个好处：

* 可以随时增加新的订阅者，并且新的订阅者只会收到其加入订阅之后的数据。
* 可以随时对任意一个订阅者取消订阅。
* 当所有订阅者取消订阅之后，Observable 会自动触发 Observable 函数，从而可以对其事件循环等进行清理。

以上能力其实可以帮助我们解决上文提到的问题 [1]。

#### 监控格式的统一

实际上，在我们的监控系统中，从数据依赖的角度，我们的监控函数会有这样几类：

* [a]纯粹的定时任务，无数据依赖，这方面比如当前内存快照数据等。
* [b]带有记忆依赖的定时任务：定时任务依赖前一次的数据（甚至更多次），需要两次数据做差等，这方面的数据比如一段时间的消耗数据，cpu 使用率的计算。
* [c]带有用户依赖的定时任务：依赖用户 id 等信息，不同用户无法共用。

而从任务触发的角度，我们仍待可以对其分类：

* [i]简单的 `setInterval` 定时任务。
* [ii]基于事件机制的不定时任务。
* [iii]基于其他触发机制的任务。

实际上，我们如果采用 Rx.js 的模式进行编写，无需考虑任务的数据依赖和触发的方式，只需写成一个一个 Observable 实例即可。另外，对于比较简单的 [a]&[i] 或 [c]&[i]  类型，我们还可以通过 monitor-rx 提供的 `convertToRx` 或 `convertToSimpleRx` 转换成 Observable 实例生成函数，例如：

```
var os = require('os');
var process = require('process');
const monitorRx = require('monitor-rx');

function getMemoryInfo() {
    return process.memoryUsage();
}

const memory = monitorRx.Utils.convertToSimpleRx(getMemoryInfo)

// 或者
//const memory = monitorRx.Utils.convertToRx({
//    getMemoryInfo
//});

module.exports = memory;
```

convertToRx 相比于 convertToSimpleRx，可以支持函数配置注入（即下文中 opts 的 func 属性和 args 属性）,可以在具体生成 Observable 实例的时候具体指定使用哪些函数以及其参数。

如果是比较复杂的 Observable 类型，那么我们就无法直接通过普通函数进行转化了，这个时候我们遵循 Observable 的标准返回 Observable 生成函数即可（不是直接返回 Observable 实例） 

这实际上也对问题 [2] 进行了解决。

#### 监控唯一性：

我们知道，如果两个用户都监控同一个信息，我们可以共用一个 Observable，这里的问题，就是如何定义两个用户的监控是"相同"的。

这里我们采用一个可选项 opts 的概念，其一共有如下属性：

```
{
    module: 'ModuleName',
    func: ['FuncName'],
    args: [['arg1','arg2']],
    opts: {interval:1000}, 
}
```

module 即用户是对哪一个模块进行监控（实际上是 Observable），func 和 args 则是监控过程中需要调用的函数，我们也可以通过 agrs 传入用户个人信息。于没有内部子函数调用的监控，二者为空即可，opts 是一些其他可选项，比如定义请求间隔等。

之后，我们通过 `JSON.stringify(opts)` 来序列化这个可选项配置，如果两个用户序列化后的可选项配置相同，那么我们就认为这两个用户可以共用一个监控，即共用一个 Observable。

### 更多内容

实际上，借助 Monitor-RX，我们可以很方便的解决上述提出的问题，Monitor-RX 也在积极的更新中，大家可以在[这里](https://github.com/aircloud/monitor-rx)了解到更多的信息。