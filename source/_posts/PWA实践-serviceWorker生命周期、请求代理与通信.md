---
title: '[PWA实践]serviceWorker生命周期、请求代理与通信'
date: 2018-02-11 15:05:43
tags:
    - PWA
---

本文主要讲 serviceWorker 生命周期和挂载、卸载等问题，适合对 serviceWorker 的作用有所了解但是具体细节不是特别清楚的读者

**以下所有分析基于 Chrome V63**

### serviceWorker的挂载

先来一段代码感受serviceWorker注册:

```
if ('serviceWorker' in navigator) {
      window.addEventListener('load', function () {
          navigator.serviceWorker.register('/sw.js', {scope: '/'})
              .then(function (registration) {
                  // 注册成功
                  console.log('ServiceWorker registration successful with scope: ', registration.scope);
              })
              .catch(function (err) {
                  // 注册失败:(
                  console.log('ServiceWorker registration failed: ', err);
              });
      });
}
```
通过上述代码，我们定义在`/sw.js`里的内容就会生效(对于当前页面之前没有 serviceWorker 的情况而言，我们注册的 serviceWorker 肯定会生效，如果当前页面已经有了我们之前注册的 serviceWorker，这个时候涉及到 serviceWorker的更新机制，下文详述)

如果我们在`sw.js`没有变化的情况下刷新这个页面，每次还是会有注册成功的回调以及相应的log输出，但是这个时候浏览器发现我们的 serviceWorker 并没有发生变化，并不会重置一遍 serviceWorker

### serviceWorker更新

我们如果想更新一个 serviceWorker，根据我们的一般web开发策略，可能会想到以下几种策略：

* 仅变更文件名(比如把`sw.js`变成`sw-v2.js`或者加一个hash)
* 仅变更文件内容(仅仅更新`sw.js`的内容，文件名不变)
* 同时变更：同时执行以上两条

在这里，我可以很负责的告诉你，**变更serviceWorker文件名绝对不是一个好的实践**，浏览器判断 serviceWorker 是否相同基本和文件名没有关系，甚至有可能还会造成浏览器抛出404异常(因为找不到原来的文件名对应的文件了)。

所以我们只需要变更内容即可，实际上，我们每次打开或者刷新该页面，浏览器都会重新请求一遍 serviceWorker 的定义文件，如果发现文件内容和之前的不同了，这个时候:

(*下文中，我们使用“有关 tab”来表示受 serviceWorker 控制的页面*，刷新均指普通刷新(F5/CommandR)并不指Hard Reload)

* 这个新的 serviceWorker 就会进入到一个 “waiting to activate” 的状态，并且只要我们不关闭这个网站的所有tab(更准确地说，是这个 serviceWorker 控制的所有页面)，新的 serviceWorker 始终不会进入替换原有的进入到 running 状态(就算我们只打开了一个有关 tab，直接刷新也不会让新的替换旧的)。

* 如果我们多次更新了 serviceWorker 并且没有关闭当前的 tab 页面，那么新的 serviceWorker 就会挤掉原先处于第二顺位(waiting to activate)的serviceWorker，变成`waiting to activate`状态

也就是说，我们只有关闭当前旧的 serviceWorker 控制的所有页面 的所有tab，之后浏览器才会把旧的 serviveWorker 移除掉，换成新的，再打开相应的页面就会使用新的了。

当然，也有一个特殊情况：如果我们在新的 serviceWorker 使用了`self.skipWaiting();`，像这样：

```
self.addEventListener('install', function(event) {
    self.skipWaiting();
});
```

这个时候，要分为以下两种情况：

* 如果当前我们只打开了一个有关 tab，这个时候，我们直接刷新，发现新的已经替换掉旧的了。
* 如果我们当前打开了若干有关 tab，这个时候，无论我们刷新多少次，新的也不会替换掉旧的，只有我们一个一个关掉tab(或者跳转走)只剩下最后一个了，这个时候刷新，会让新的替换旧的(也就是上一种情况)

Chrome 的这种机制，防止了同一个页面先后被新旧两个不同的 serviceWorker 接管的情况出现。

#### 手动更新

虽然说，在页面每次进入的时候浏览器都会检查一遍 serviceWorker 是否更新，但如果我们想要手动更新 serviceWorker 也没有问题：

```
navigator.serviceWorker.register("/sw.js").then(reg => {
  reg.update();
  // 或者 一段时间之后更新
});
```

这个时候如果 serviceWorker 变化了，那么会重新触发 install 执行一遍 install 的回调函数，如果没有变，就不会触发这个生命周期。

#### install 生命周期钩子

我们一般会在 sw.js 中，添加`install`的回调，一般在回调中，我们会进行缓存处理操作，像这样：

```
self.addEventListener('install', function(event) {
    console.log('[sw2] serviceWorker Installed successfully', event)

    event.waitUntil(
        caches.open('mysite-static-v1').then(function(cache) {
            return cache.addAll([
                '/stylesheets/style.css',
                '/javascripts/common.39c462651d449a73b5bb.js',
            ]);
        })
    )
}    
```

如果我们新打开一个页面，如果之前有 serviceWorker，那么会触发`install`，如果之前没有， 那么在 serviceWorker 装载后会触发 `install`。

如果我们刷新页面，serviceWorker 和之前没有变化或者 serviceWorker 已经处在 `waiting to activate`，不会触发`install`，如果有变化，会触发`install`，但不会接管页面(上文中提到)。

#### activate 生命周期钩子

activate 在什么时候被触发呢？

如果当前页面没有 serviceworker ，那么会在 install 之后触发。

如果当前页面有 serviceWorker，并且有 serviceWorker更新，新的 serviceWorker 只会触发 install ，不会触发 activate

换句话说，当前变成 active 的 serviceWorker 才会被触发这个生命周期钩子


### serviceWorker 代理请求

serviceWorker 代理请求相对来说比较好理解，以下是一个很简单的例子：

```
self.addEventListener('install', function(event) {
    console.log('[sw2] serviceWorker Installed successfully', event)

    event.waitUntil(
        caches.open('mysite-static-v1').then(function(cache) {
            return cache.addAll([
                '/stylesheets/style.css',
                '/javascripts/common.39c462651d449a73b5bb.js',
            ]);
        })
    );
});

self.addEventListener('fetch', function(event) {
    console.log('Handling fetch event for', event.request.url);
    // console.log('[sw2]fetch but do nothing')

    event.respondWith(
        // caches.match() will look for a cache entry in all of the caches available to the service worker.
        // It's an alternative to first opening a specific named cache and then matching on that.
        caches.match(event.request).then(function(response) {
            if (response) {
                console.log('Found response in cache:', response);

                return response;
            }

            console.log('No response found in cache. About to fetch from network...');

            // event.request will always have the proper mode set ('cors, 'no-cors', etc.) so we don't
            // have to hardcode 'no-cors' like we do when fetch()ing in the install handler.
            return fetch(event.request).then(function(response) {
                console.log('Response from network is:', response);

                return response;
            }).catch(function(error) {
                // This catch() will handle exceptions thrown from the fetch() operation.
                // Note that a HTTP error response (e.g. 404) will NOT trigger an exception.
                // It will return a normal response object that has the appropriate error code set.
                console.error('Fetching failed:', error);

                throw error;
            });
        })
    );
});
```

有两点要注意的：

我们如果这样代理了，哪怕没有 cache 命中，实际上也会在控制台写from serviceWorker，而那些真正由serviceWorker发出的请求也会显示，有一个齿轮图标，如下图：

![](https://www.10000h.top/images/sw_1.png)

第二点就是我们如果在 fetch 的 listener 里面 do nothing， 也不会导致这个请求直接假死掉的。

另外，通过上面的代码我们发现，实际上由于现在我们习惯给我们的文件资源加上 hash，所以我们基本上不可能手动输入需要缓存的文件列表，现在大多数情况下，我们都是借助 webpack 插件，完成这部分工作。

### serviceWorker 和 页面之间的通信

serviceWorker向页面发消息：

```
sw.js:

self.clients.matchAll().then(clients => {
    clients.forEach(client => {
        console.log('%c [sw message]', 'color:#00aa00', client)
        client.postMessage("This message is from serviceWorker")
    })
})

主页面:

navigator.serviceWorker.addEventListener('message', function (event) {
    console.log('[Main] receive from serviceWorker:', event.data, event)
});
```

当然，这里面是有坑的：

* 主界面的事件监听需要等serviceWorker注册完毕后，所以一般`navigator.serviceWorker.register`的回调到来之后再进行注册(或者延迟足够的时间)。
* 如果在主界面事件监听还没有注册成功的时候 serviceWorker 发送消息，自然是收不到的。如果我们把 serviceWorker 直接写在 install 的回调中，也是不能被正常收到的。

从页面向 serviceWorker 发送消息：

```
主页面:

navigator.serviceWorker.controller && navigator.serviceWorker.controller.postMessage('hello serviceWorker');

sw.js:
self.addEventListener('message', function (event) {
    console.log("[sw from main]",event.data); // 输出：'sw.updatedone'
});
```

同样的，这也要求主界面的代码需要等到serviceWorker注册完毕后触发，另外还有一点值得注意， serviceWorker 的事件绑定代码要求主界面的serviceWorker已经注册完毕后才可以。

也就是说，如果当前页面没有该serviceWorker 第一次注册是不会收到主界面接收到的消息的。

记住，只有当前已经在 active 的 serviceWorker， 才能和主页面收发消息等。

**以上就是和 serviceWorker 有关的一些内容，在下一篇文章中，我会对PWA 添加至主屏幕等功能进行总结**

