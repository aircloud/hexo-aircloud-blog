---
title: 从源码分析sentry的错误信息收集
date: 2018-08-18 19:23:55
tags:
    - javascript
    - 前端监控
---

raven.js 是 sentry 为 JavaScript 错误上报提供的 JS-SDK，本篇我们基于其源代码对其原理进行分析，本篇文章只分析前端部分，对应的文件目录是`https://github.com/getsentry/sentry-javascript/tree/master/packages/raven-js`。

首先抛出几个问题：

* **raven.js 是如何收集浏览器错误信息的？**
* **raven.js 上报的错误信息格式是什么样的？又是如何把这些信息传给后端？支不支持合并上报？**
* **面包屑（breadcrumbs）是什么？raven.js 如何来收集面包屑信息？**
* **raven.js 如何和框架配合使用（比如 vue、react）？**

在回答以上这几个问题之前，我们首先来对 raven.js 做一个宏观的分析，主要涉及其文件目录、所引用的第三方框架等。

raven.js 的核心文件内容并不多，其中使用了三个第三方库，放在了 vendor 文件夹下：

* [json-stringify-safe](https://github.com/moll/json-stringify-safe) ：一个对 `JSON.stringify` 的封装，安全的 json 序列化操作函数，不会抛出循环引用的错误。
	* 这里面有一个注意点要单独说一下，我们熟知的 `JSON.stringify` , 可以接受三个参数：第一个参数是我们要序列化的对象；第二个参数是对其中键值对的处理函数；第三个参数是控制缩进空格。reven.js 的 `json-stringify-safe` 就是充分利用了这三个参数。
* [md5](https://github.com/blueimp/JavaScript-MD5)：js 的 md5 函数。
* [TraceKit](https://github.com/csnover/TraceKit)：TraceKit 是一个已经比较完善的错误收集、堆栈格式化的库，reven.js 的功能在很大程度上对它有所依赖。

除此之外，raven.js 支持插件，官方提供的一些知名库的 sentry 插件主要放在了 plugin 文件夹下面，raven.js 的一些核心文件，则放在了 src 文件夹下面。

### raven.js 是如何收集错误信息的？

我们知道，在前端收集错误，肯定离不开 `window.onerror` 这个函数，那么我们就从这个函数说起。

实际上，这部分工作是 raven.js 引用的第三方库 TraceKit 完成的：

```
function installGlobalHandler() {
  if (_onErrorHandlerInstalled) { // 一个起到标志作用的全局变量
    return;
  }
  _oldOnerrorHandler = _window.onerror; 
  // _oldOnerrorHandler 是防止对用户其他地方定义的回调函数进行覆盖
  // 该 _window 经过兼容，实际上就是 window
  _window.onerror = traceKitWindowOnError;
  _onErrorHandlerInstalled = true;
}
```

相关错误回调函数交给 traceKitWindowOnError 处理，下面我们来看一下 traceKitWindowOnError 函数，为了避免太多冗余代码，我们仅分析一种主要情况：

```
function traceKitWindowOnError(msg, url, lineNo, colNo, ex) {
	
	var exception = utils.isErrorEvent(ex) ? ex.error : ex;
	//...
    stack = TraceKit.computeStackTrace(exception);
    notifyHandlers(stack, true);
    //...
   
    //...
    if (_oldOnerrorHandler) {
       return _oldOnerrorHandler.apply(this, arguments);
    }
    return false;
}
```

其中调用的最重要的一个函数，就是 computeStackTrace，而这个函数也是 TraceKit 的核心函数，简单来讲，它做的事情就是统一格式化报错信息调用栈，因为对于各个浏览器来说，返回的 Error 调用栈信息格式不尽相同，另外甚至还有的浏览器并不返回调用栈，computeStackTrace 函数对这些情况都做了兼容性处理，并且对于一些不返回调用栈的情况，还使用了 caller 来向上回溯函数的调用栈，最终把报错信息转化成一个键相同的对象数组，做到了报错信息格式的统一。

notifyHandlers 函数则是通知相关的回调函数。 实际上，raven.js 在 install 函数中会调用 TraceKit.report.subscribe 函数，并把对错误的处理逻辑写入回调：

```
function subscribe(handler) {
    installGlobalHandler();
    handlers.push(handler);
}
```

以上过程完成了错误处理过程中的负责角色转换，并且借助 TraceKit，可以使 raven.js 得到一个结构比较清晰的带有格式化好的调用栈信息的错误内容对象，之后，raven.js 对错误内容进一步处理并最终上报。

下面我们对错误处理 raven.js 控制的部分做了一些梳理：

```
 _handleOnErrorStackInfo: function(stackInfo, options) {
    options.mechanism = options.mechanism || {
      type: 'onerror',
      handled: false
    };
    // mechanism 和错误统计来源有关

    if (!this._ignoreOnError) {
      this._handleStackInfo(stackInfo, options);
    }
},

_handleStackInfo: function(stackInfo, options) {
    var frames = this._prepareFrames(stackInfo, options);

    this._triggerEvent('handle', {
      stackInfo: stackInfo,
      options: options
    });

    this._processException(
      stackInfo.name,
      stackInfo.message,
      stackInfo.url,
      stackInfo.lineno,
      frames,
      options
    );
},

_processException: function(type, message, fileurl, lineno, frames, options) {
    // 首先根据 message 信息判断是否是需要忽略的错误类型
    // 然后判断出错的文件是否在黑名单中或者白名单中
    // 接下来对错误内容进行必要的整合与转换，构造出 data 对象
    // 最后调用上报函数
    this._send(data);
}

_send: function(data) {
	
	// 对 data 进一步处理，增加必要的信息，包括后续会提到的面包屑信息

	// 交由 _sendProcessedPayload 进行进一步处理
	this._sendProcessedPayload(data);
}

_sendProcessedPayload: function(data, callback) {

	// 对 data 增加一些必要的元信息
	// 可以通过自定义 globalOptions.transport 的方式来自定义上报函数 
	(globalOptions.transport || this._makeRequest).call(this, {
	     url: url,
	     auth: auth,
	     data: data,
	     options: globalOptions,
	     onSuccess: function success() {
	       
	     },
	     onError: function failure(error) {
	       
	     }
	});
}    

// 真正发起请求的函数
_makeRequest: function(opts) {
	// 对于支持 fetch 的浏览器，直接使用 fetch 的方式发送 POST 请求
	// 如果浏览器不支持 fetch，则使用 XHR 的传统方式发送 POST 请求
}
``` 

实际上我们可以发现，从拿到已经初步格式化的报错信息，到最终真正执行数据上报，raven.js 的过程非常漫长，这其中我分析有如下几个原因：

* 每个函数只处理一件或者一些事情，保持函数的短小整洁。
* 部分函数可以做到复用（因为除了自动捕获错误的方式， raven.js 还提供通过 captureException，即 `try {
    doSomething(a[0])
} catch(e) {
    Raven.captureException(e)
}` 的方式来上报错误，两个过程中有一些函数的调用是有重叠的）。

但是笔者认为，raven.js 的代码设计还有很多值得优化的地方，比如：

* 对最终上报数据（data）的属性处理和增加分散在多个函数，并且有较多可选项目，很难梳理出一个完整的 data 格式，并且不便于维护。
* 部分函数的拆分必要性不足，并且会增加链路的复杂性，比如 `_processException `、`_sendProcessedPayload `、`_makeRequest `等都只在一个链路中被调用一次。
* 部分属性重命名会造成资源浪费，由于 TraceKit 部分最终返回的数据格式并不完全满足 raven.js 的需要，所以 raven.js 之后又在较后阶段进行了重命名等处理，实际上这些内容完全可以通过一些其他的方式避免。

最后，非常遗憾，sentry 目前完全不支持合并上报，就算是在同一个事件循环（甚至事件循环的同一个阶段，关于事件循环，可以参考我之前绘制的[一张图](https://www.processon.com/view/link/5b6ec8cbe4b053a09c2fb977)）的两个错误，sentry 都是分开来上报的，这里有一个简单例子：

```javascript
Raven.config('http://8ec3f1a9f652463bb58191bd0b35f20c@localhost:9000/2').install()
let s = window.ss;

try{
    let b = s.b
} catch (e) {
    Raven.captureException(e)
    // sentry should report error now
}

s.nomethod();
// sentry should report error now
```

以上例子中，sentry 会发送两个 POST 请求。

### raven.js 最终上报数据的格式


这一部分，我们并不会详细地分析 raven.js 上报的数据的每一项内容，仅会给读者展示一个比较典型的情况。

我们看一下对于一个一般的 js 错误，raven.js 上报的 json 中包含哪些内容，下面是一个已经删掉一些冗余内容的典型上报信息：

```
{
  "project": "2",
  "logger": "javascript",
  "platform": "javascript",
  "request": {
    "headers": {
      "User-Agent": "Mozilla/5.0 (iPhone; CPU iPhone OS 11_0 like Mac OS X) AppleWebKit/604.1.38 (KHTML, like Gecko) Version/11.0 Mobile/15A372 Safari/604.1"
    },
    "url": "http://localhost:63342/sentry-test1/test1.html?_ijt=j54dmgn136gom08n8v8v9fdddu"
  },
  "exception": {
    "values": [
      {
        "type": "TypeError",
        "value": "Cannot read property 'b' of undefined",
        "stacktrace": {
          "frames": [
            {
              "filename": "http://localhost:63342/sentry-test1/test1.html?_ijt=j54dmgn136gom08n8v8v9fdddu",
              "lineno": 19,
              "colno": 19,
              "function": "?",
              "in_app": true
            }
          ]
        }
      }
    ],
    "mechanism": {
      "type": "generic",
      "handled": true
    }
  },
  "transaction": "http://localhost:63342/sentry-test1/test1.html?_ijt=j54dmgn136gom08n8v8v9fdddu",
  "extra": {
    "session:duration": 6
  },
  "breadcrumbs": {
    "values": [
      {
        "timestamp": 1534257309.996,
        "message": "_prepareFrames stackInfo: [object Object]",
        "level": "log",
        "category": "console"
      },
      // ...
   ]
  },
  "event_id": "ea0334adaf9d43b78e72da2b10e084a9",
  "trimHeadFrames": 0
}
```

其中支持的信息类型重点分为以下几种：

* sentry 基本配置信息，包括库本身的配置和使用者的配置信息，以及用户的一些自定义信息
* 错误信息，主要包括错误调用栈信息
* request 信息，主要包括浏览器的 User-Agent、当前请求地址等
* 面包屑信息，关于面包屑具体指的是什么，我们会在下一环节进行介绍

### raven.js 面包屑收集

面包屑信息，也就是错误在发生之前，一些用户、浏览器的行为信息，raven.js 实现了一个简单的队列（有一个最大条目长度，默认为 100），这个队列在时刻记录着这些信息，一旦错误发生并且需要上报，raven.js 就把这个队列的信息内容，作为面包屑 breadcrumbs，发回客户端。

面包屑信息主要包括这几类：

* 用户对某个元素的点击或者用户对某个可输入元素的输入
* 发送的 http 请求
* console 打印的信息（支持配置 'debug', 'info', 'warn', 'error', 'log' 等不同级别）
* window.location 变化信息

接下来，我们对这几类面包屑信息 sentry 的记录实现进行简单的分析。

实际上，sentry 对这些信息记录的方式比较一致，都是通过对原声的函数进行包装，并且在包装好的函数中增加自己的钩子函数，来实现触发时候的事件记录，实际上，sentry 总共包装的函数有：

* window.setTimeout
* window.setInterval
* window.requestAnimationFrame
* EventTarget.addEventListener
* EventTarget.removeEventListener
* XMLHTTPRequest.open
* XMLHTTPRequest.send
* window.fetch
* History.pushState
* History.replaceState

>备注：这里包装的所有函数，其中有一部分只是使 raven.js 具有捕获回调函数中错误的能力（对回调函数进行包装）

接下来我们看一段典型的代码，来分析 raven.js 是如何记录用户的点击和输入信息的（通过对 EventTarget.addEventListener 进行封装）：

```javascript
function wrapEventTarget(global) {
      var proto = _window[global] && _window[global].prototype;
      if (proto && proto.hasOwnProperty && proto.hasOwnProperty('addEventListener')) {
        fill(
          proto,
          'addEventListener',
          function(orig) {
            return function(evtName, fn, capture, secure) {
              try {
                if (fn && fn.handleEvent) { //兼容通过 handleEvent 的方式进行绑定事件
                  fn.handleEvent = self.wrap(
                    {
                      mechanism: {
                        type: 'instrument',
                        data: {
                          target: global,
                          function: 'handleEvent',
                          handler: (fn && fn.name) || '<anonymous>'
                        }
                      }
                    },
                    fn.handleEvent
                  );
                }
              } catch (err) {
              }

              var before, clickHandler, keypressHandler;

              if (
                autoBreadcrumbs &&
                autoBreadcrumbs.dom &&
                (global === 'EventTarget' || global === 'Node')
              ) {
                // NOTE: generating multiple handlers per addEventListener invocation, should
                //       revisit and verify we can just use one (almost certainly)
                clickHandler = self._breadcrumbEventHandler('click');
                keypressHandler = self._keypressEventHandler();
                before = function(evt) { // 钩子函数，用于在回调函数调用的时候记录信息
                  if (!evt) return;

                  var eventType;
                  try {
                    eventType = evt.type;
                  } catch (e) {
                    // just accessing event properties can throw an exception in some rare circumstances
                    // see: https://github.com/getsentry/raven-js/issues/838
                    return;
                  }
                  if (eventType === 'click') return clickHandler(evt);
                  else if (eventType === 'keypress') return keypressHandler(evt);
                };
              }
              return orig.call(
                this,
                evtName,
                self.wrap(
                  {
                    mechanism: {
                      type: 'instrument',
                      data: {
                        target: global,
                        function: 'addEventListener',
                        handler: (fn && fn.name) || '<anonymous>'
                      }
                    }
                  },
                  fn,
                  before
                ),
                capture,
                secure
              );
            };
          },
          wrappedBuiltIns
        );
        fill(
          proto,
          'removeEventListener',
          function(orig) {
            return function(evt, fn, capture, secure) {
              try {
                fn = fn && (fn.__raven_wrapper__ ? fn.__raven_wrapper__ : fn);
              } catch (e) {
                // ignore, accessing __raven_wrapper__ will throw in some Selenium environments
              }
              return orig.call(this, evt, fn, capture, secure);
            };
          },
          wrappedBuiltIns
        );
      }
    }
```

以上代码兼容了通过 handleEvent 的方式进行绑定事件（如果没有听说过这种方式，可以在[这里](http://www.ayqy.net/blog/handleevent%E4%B8%8Eaddeventlistener/)补充一些相关的知识）。

默认情况下，raven.js 只记录通过 `EventTarget.addEventListener` 绑定的点击和输入信息，实际上这是比较科学的，并且这些信息较为有效。另外，raven.js 也提供了记录所有点击和输入信息的可选项，其实现方式更为简单，直接在 document 上添加相关的监听即可。

### raven.js 如何和框架配合使用

raven.js 和框架配合使用的方式非常简单，但是我们要知道，很多框架内置了错误边界处理，或者对错误进行转义。以至于我们通过 window.onerror 的方式得不到完整的错误信息。同时，有些框架提供了错误处理的接口（比如 vue），利用错误处理的接口，我们能够获取到和错误有关的更多更重要的信息。

raven.js 利用各个框架的官方接口，提供了 vue、require.js、angular、ember、react-native 等各个框架的官方插件。

插件内容本身非常简单，我们可以看一下 vue 插件的代码：

```
function formatComponentName(vm) {
  if (vm.$root === vm) {
    return 'root instance';
  }
  var name = vm._isVue ? vm.$options.name || vm.$options._componentTag : vm.name;
  return (
    (name ? 'component <' + name + '>' : 'anonymous component') +
    (vm._isVue && vm.$options.__file ? ' at ' + vm.$options.__file : '')
  );
}

function vuePlugin(Raven, Vue) {
  Vue = Vue || window.Vue;

  // quit if Vue isn't on the page
  if (!Vue || !Vue.config) return;

  var _oldOnError = Vue.config.errorHandler;
  Vue.config.errorHandler = function VueErrorHandler(error, vm, info) {
    var metaData = {};

    // vm and lifecycleHook are not always available
    if (Object.prototype.toString.call(vm) === '[object Object]') {
      metaData.componentName = formatComponentName(vm);
      metaData.propsData = vm.$options.propsData;
    }

    if (typeof info !== 'undefined') {
      metaData.lifecycleHook = info;
    }

    Raven.captureException(error, {
      extra: metaData
    });

    if (typeof _oldOnError === 'function') {
      _oldOnError.call(this, error, vm, info);
    }
  };
}

module.exports = vuePlugin;
```

应该不用进行过多解释。

你也许想知道为什么没有提供 react 插件，事实上，react 16 以后才引入了[Error Boundaries](https://reactjs.org/blog/2017/07/26/error-handling-in-react-16.html)，这种方式由于灵活性太强，并不太适合使用插件，另外，就算不使用插件，也非常方便地使用 raven.js 进行错误上报，可以参考[这里](https://docs.sentry.io/clients/javascript/integrations/react/)

>但笔者认为，目前 react 的引入方式会对源代码进行侵入，并且比较难通过构建的方式进行 sentry 的配置，也许我们可以寻找更好的方式。

完。

