---
title: web 页面内存分析与生产环境禁用 console
date: 2021-09-04 19:15:01
tags:
    - 性能优化
    - javascript
---

## 背景

我们在开发前端页面中，建议在生产环境中将所有的 console 禁用，并通过自定义的日志函数进行日志输出，即使无法禁用，也需要自定义文本过滤函数，严格控制 console 的输出。

但实际上，笔者经历的项目中很多都没有办法做到这一点，虽然我们知道，禁用 console 的主要原因除了信息泄漏的风险外，还有就是 console 打印的内容无法被内存回收。但仍然总是会有一些同学对禁用 console 的必要性表示质疑，在本篇文章中，本文通过两个实际遇到的比较严重的例子，来解释禁用 console 的必要性。

> 出于保密性考虑，例子本身已经脱敏，本文使用示例代码模拟原始场景。

## 页面内存

在具体例子讲解之前，我们需要先对页面内存有一个认知，在前端开发中，我们虽然开发的只是在 Chrome 等浏览器中浏览的页面，但是对页面的 cpu 和 内存占用也需要时刻保持关注。

cpu 和 内存一般是针对进程级别，chrome 的进程模型比较复杂，一般情况下，我们可以认为同域的页面有比较大的概率进行进程复用。

Chrome 提供了一些手段，让我们可以监控页面的 cpu 和内存，例如：

Performance Monitor 可以让我们直观地监测页面的 cpu、js heap 的分配情况等：

![chrome_monitor](/img/chrome_monitor.png)

Chrome 自身提供了一个任务管理器（More Tools -> Task Manager），可以让我们关注各个页面的性能情况：

![chrome_task_manager](/img/chrome_task_manager.png)

除了实时监控以外，Chrome DevTools 的 Memory 等 tab 也可以让我们对内存占用进行取样分析，以及内存泄漏分析：

* 一般来说，我们可以通过对两次 heap snapshot，然后搜索关键变量的数目与引用关系是否符合预期，来证明是否存在内存泄漏。
* 除此之外，我们使用 WeakMap 来跟踪我们的实例，也可以辅助进行一定的内存泄漏分析。


## 使用 console.log 打印 dom 元素造成死循环 OOM 

之前笔者负责的一个页面，在某个版本出现了一个问题：打开页面后不久，在什么操作也没有做的情况下直接卡死无响应。

一般来说，js 导致网页无响应的可能性并不多，我们首先怀疑是因为死循环导致的。

不过我们通过对比上次和这一次的代码，发现变动极小（实际上，我们一开始都忽略了 console.log），我们通过在 Chrome 的 devTools 里面打断点，最终定位发现是卡死在第三方库 sentry 的 console.log 中。

最终我们定位出真正的原因：其中一处 try catch 在 catch 到错误之后，会 console.log 打印包括 dom 在内的一些内容，而我们使用的 console.log 被 sentry 进行了覆盖，它的覆盖方法大致如下（这个确实有点坑，以至于我们直接查看 console.log 仍然是 [native code]， 不过最新版本的 Chrome 这个代码已经不能完全 work）：

```
let __native_console = console.log;
console.log = function() {
  // 递归遍历各个属性
  __native_console(...arguments);
}
console.log.prototype.__native_console = __native_console;
console.log.prototype.toString = function() { 
  if (this.__native_console) return this.__native_console.toString();
  return this.toString();
}
// TODO: 2021.09 @niexiaotao 补充一下最新的实现
```

**这里之所以死循环，是因为 React 中 FiberNode 是 Dom 的其中一个属性，console.log 递归遍历到了 FiberNode，其本质是一个双向链表，最终造成无限递归死循环**。

我们可以比较方便的随便找个 React 项目验证这一点：

![React Fiber](/img/chrome_fiber.png)

## detached dom 过多导致页面内存持续上涨

另外笔者接触到的一个比较严重的问题，是之前某项目的一个页面，随着使用时间增加，页面的内存使用量快速持续增加，最终导致卡顿和崩溃。

这个问题的定位过程也比较艰辛，最终发现其中的一个主要原因是 **console.log 打印了 dom 节点，导致 detached dom 持续增多并且无法被回收，最终导致严重问题**。

关于 detached dom 的问题我们可以使用[通过压缩合成层优化性能](http://niexiaotao.cn/2021/09/04/%E9%80%9A%E8%BF%87%E5%8E%8B%E7%BC%A9%E5%90%88%E6%88%90%E5%B1%82%E4%BC%98%E5%8C%96%E6%80%A7%E8%83%BD/) 这里的 demo，简单修改：

将原本需要挂载到 dom 的节点直接进行打印：

```
for(let i = 0; i < totalListCount; i += 1) {
  let fragment = document.createElement("div");
  fragment.classList.add("li");
  fragment.innerHTML = `<p>this is the ${i} element</p>`;
  console.log(fragment);
  // list.appendChild(fragment);
}
```

我们很容易看到这样就产生了 500 个 detach 节点，并且在页面的生命周期内，无法进行释放：

![detach console](/img/chrome_detach_console.png)

## 总结

实际上，在生产环境使用 console.log 造成的问题远不止上面的两例，而且这类问题通常排查起来都会比较艰难，因此，建议大家落实在生产环境禁用 console。


