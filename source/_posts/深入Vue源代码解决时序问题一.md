---
title: 深入Vue源代码解决时序问题一
date: 2019-07-06 15:21:51
tags:
	- Vue
	- viola
---

>viola 是一个支持 Vue 的动态化框架，其 Vue 版本在 Vue 官方版本 2.5.7 上进行了少量改写，本文针对其进行具体分析。

最初，有使用者报告一个错误：在 iOS 系统，退出页面的时候，框架报错：

```
TypeError: undefined is not an object(evaluating 'e.isDestroyed"
```

接到这个错误之后，我首先进入 Vue 的 debug 版本，尝试获取更详细的信息：

```
TypeError: undefined is not an object(evaluating 'componentInstance.isDestroyed"
```

我们顺利地拿到了报错的变量名称，去 Vue 源代码中搜索，我们可以发现报错之处：

```javascript
destroy: function destroy (vnode) {
    var componentInstance = vnode.componentInstance;
    if (!componentInstance._isDestroyed) { // 这里报错
      if (!vnode.data.keepAlive) {
        componentInstance.$destroy();
      } else {
        deactivateChildComponent(componentInstance, true /* direct */);
      }
    }
  }
```

这里是 `componentInstance` 为 undefined，这个实际上是 vnode 的实例，其为 undefined，说明该 vue 组件在之前的阶段就已经出错不正常了，这里并不是错误的根源所在，我们需要再次进行寻找报错原因。

于是我们查看业务代码的所有日志，又发现了这样一条报错：

```
[Vue warn]: Error in nextTick: "TypeError: undefined is not an object (evaluating 'vm.$options')" 
```

初始化阶段出现这样一个错误，我们怀疑 `vm` 就是上文的 `componentInstance`，于是，我们打印报错堆栈：

```javascript
 调用栈:
function updateChildComponent(
    vm,
    propsData,
    listeners,
    parentVnode,
    renderChildren
  ) {
        //...
        var hasChildren = !!(
              renderChildren ||
              vm.$options._renderChildren || // 这里报错
              parentVnode.data.scopedSlots ||
              vm.$scopedSlots !== emptyObject
            );
    }

function prepatch(oldVnode, vnode) {
      var options = vnode.componentOptions;
      var child = vnode.componentInstance = oldVnode.componentInstance;
      updateChildComponent(
        child,
        options.propsData,
        options.listeners,
        vnode,
        options.children
      );
    }

function patchVnode(oldVnode, vnode, insertedVnodeQueue, removeOnly) {}
function patch(oldVnode, vnode, hydrating, removeOnly) {}
function (vnode, hydrating) {}
function () {
        vm._update(vm._render(), hydrating);
      }
function get() {}
function getAndInvoke(cb) {}
function run() {}
function flushSchedulerQueue() {}
function flushCallbacks() {}
```

调用栈实际上有点冗长，不过我们还是能发现两个有用的信息：

* 初始化阶段为 `undefined` 的 `vm`，就是 `componentInstance`，也就是和 destroy 阶段的报错属于同一个原因。
* 根据调用栈发现，这是一个更新阶段的报错。

这引发了我们的思考：更新阶段找不到 `componentInstance` 报错。

这里实际上有点阻塞了，因为一般来说，Vue 的源代码经过测试，应该不会出现这种问题的，那是不是我们的问题呢，我们回归到业务代码：

```
created() {
    this.getFeedsListFromCache();
},
methods: {
    getFeedsListFromCache() {
        viola.requireAPI("cache").getItem(this.cacheKey_feeds, data => {
            this.processData(data.list);
        });
    },
    processData(list = [], opt = {}) {
        if (this.list.length < cacehFeedsLength) {
        }
        this.list = [];
    },
}
```

我们对业务代码进行了抽象简化，上面是我们的最小问题 Demo，实际上我们就做了这样一件事情：

* 在 created 执行方法，调用端的接口，再回调函数里面更新某个 data 中声明的数据。

首先，我们可以梳理下对一般 vue 组件的初始化更新，vue 是如何做的：

* created 时实际上 vnode 已经建立完成，这个时候还没有 mount，但是数据监听已经建立了，这个时候如果改动数据，会把相关 update 函数放在一个名为 flushCallbacks 的函数队列中。
* 该函数队列会通过默认为 `Promise.then` 的 microtask 方式来调度，当前阶段的 mount 流程会继续，mount 结束后，会执行 flushCallbacks 队列中的更新操作。

从代码层面上来讲，这几个流程应该是这样的：

```
 ├── callHook(vm, 'created'); // 执行created 钩子
 ├── proxySetter(val); // 改变数据，调用 proxy
 ├── Watcher.prototype.update; // 调用 Watcher，将 update 操作入栈
 ├── vm.$mount(vm.$options.el); // 执行 mount 流程
 ├── callHook(vm, 'beforeMount');
 ├──  callHook(vm, 'mounted'); // 依次调用 beforeMount 和 mounted
 └── flushCallbacks // 执行 更新
```

然后我们分析我们这里的流程，首先值得强调的是这个函数 `viola.requireAPI("cache").getItem`，这个函数是端注入的函数，但我们不能将其当作异步函数来对待，实际上，**这是一个同步函数**，（至于这个同步函数和 js 中的普通函数，是否有区别，还有待商榷，不过应该是有区别的，因为如果我们不用此函数的话，就不会出现该问题。）

接下来，我们打出详细的调用栈，根据顺序来分析实际的执行流程：

```
 ├── callHook(vm, 'created'); // 执行created 钩子
 ├── proxySetter(val); // 改变数据，调用 proxy
 ├── Watcher.prototype.update; // 调用 Watcher，将 update 操作入栈
 ├── flushCallbacks // 执行 更新
 ├── vm.$mount(vm.$options.el); // 执行 mount 流程 
 ├── callHook(vm, 'beforeMount');
 └── callHook(vm, 'mounted'); // 依次调用 beforeMount 和 mounted
```

我们发现，我们的执行流程出现了很大问题：**在 mount 阶段未完成的时候就执行了 flushCallbacks，先执行更新操作，这里的顺序错乱导致了后续问题**。

我们可看下调用 `flushCallbacks` 的代码：

```javascript
if (typeof Promise !== 'undefined' && isNative(Promise)) {
  var p = Promise.resolve();
  microTimerFunc = function () {
    p.then(flushCallbacks);
    // in problematic UIWebViews, Promise.then doesn't completely break, but
    // it can get stuck in a weird state where callbacks are pushed into the
    // microtask queue but the queue isn't being flushed, until the browser
    // needs to do some other work, e.g. handle a timer. Therefore we can
    // "force" the microtask queue to be flushed by adding an empty timer.
    if (isIOS) { setTimeout(noop); }
  };
} 
```

这里 `microTimerFunc` 的 `p.then`，被同步执行了，也就是说，这里的微任务优先于当前事件循环的函数执行了（此时由于 mount 流程是同步的，mount 流程的相关函数**理应**在该事件循环中，优先于微任务执行）。

我们找到了根源，接下来就是分析解决方案和根本原因。

由于我们的问题在于 update 流程执行太快了，所以采用一种方式放慢一点即可：

* 将 vue 的微任务模式（默认）改成宏任务模式：`var useMacroTask = false; => true`。
* 在 created 阶段的加一个 `setTimeout(0)`。

不过对于根本原因，实际上本次仍然没有完全分析透彻，还留有如下疑问：

* `viola.requireAPI("cache").getItem` 这个函数到底做了什么？其对事件循环有什么影响？
* 在执行 `microTimerFunc` 的时候，为什么 `p.then` 优先于 `vm.$mount` 执行了？
* 该错误仅在 iOS 系统出现，iOS 系统是否会在某些情况将微任务的优先级变高？

对于这些疑问，Vue 源代码中也做了一些评论：

```
// Here we have async deferring wrappers using both microtasks and (macro) tasks.
// In < 2.4 we used microtasks everywhere, but there are some scenarios where
// microtasks have too high a priority and fire in between supposedly
// sequential events (e.g. #4521, #6690) or even between bubbling of the same
// event (#6566). However, using (macro) tasks everywhere also has subtle problems
// when state is changed right before repaint (e.g. #6813, out-in transitions).
// Here we use microtask by default, but expose a way to force (macro) task when
// needed (e.g. in event handlers attached by v-on).
```

不过，这里始终都没有找到最本质的原因，也许这和 iOS JSCore 的微任务/宏任务的处理机制有关，具体原因，待下次探究。



