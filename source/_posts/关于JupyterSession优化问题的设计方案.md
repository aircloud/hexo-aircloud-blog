---
title: 关于 JupyterLab cell 输出保持问题的设计方案
date: 2022-09.16 19:15:51
tags:
    - 前端综合
---

## 摘要

本文试图尝试阐述目前 JupyterLab 当前存在的 cell 输出保持问题的原因，并且给出一个基于当前 Jupyter 架构的解决方案。

## 问题概述

这个问题，简单描述为就是当我们在 JupyterLab Notebook 中执行一个耗时代码，如果此时我们因为某些原因刷新或者关闭重新打开了页面，我们就永远无法继续跟踪输出了。

我们通过一个例子来更直观地感受：

JupyterLab 在 NodeBook 中运行如下的代码：

```python
import time

i = 0
while True:
    i += 1
    if (i > 1000):
        break;
    print(f'current: {i}')
    time.sleep(1)
```

![跟踪日志](/img/jupyter_cell1.png)

如果此时你强制刷新页面，就会发现标准输出丢失，并且这个时候没有办法再找到。

## 该问题的社区讨论和成因概述

> 如果只关注解决方案，可以直接滚动至下文解决方案部分

### 社区现状

* 在 2015 年这个问题就被提出来了，不过因为它和当前 jupyter 的架构有所冲突，需要对 server 端进行巨大的重构，所以就一直搁置，**官方至今未修复此问题**。
* 社区一些曲折方案，但是所有方案**都需要使用者代码改动或者安装额外插件**，笔者对其中一些方案进行列举：
  * 有人提出使用 ipycache 这个插件，可以保存输出，不至于让输出丢失，但仍然不能监控进度，而且要写一些额外代码，相对来说还是会比较麻烦。
  * 还有人提出使用 `%%capture var` 来捕获一些 stderr，通过 `var.show()` 来显示，同样的，这个也是只能等到执行完 cell 之后。
  * 第三个类似的方案：`sys.stdout = open('my_log.log', 'w')`
* 目前 JupyterLab V4 版本正在进行 [Real Time Collaboration](https://github.com/jupyterlab/jupyterlab/issues/5382) 功能的开发，涉及到很大的改动，**在原本的计划中，会顺带把这个问题修复掉**。
  * *V4 版本在 22 年 5月左右开始发布 alpha 版本，目前仍然处于 alpha 阶段，乐观来讲明年年初也许可以 Release 正式版本（P.S. 我们现在一般用的是 V3， 的 21 年 11月左右发布的一个较为稳定的版本）*
* 在 2022 年的 5 月 19 日，这个问题被移动到了 Jupyter 的[任务看板](https://github.com/jupyterlab/jupyterlab/projects/12#card-65141043)之中，不过目前**没有负责人**。

总之，解决这个问题的主要困境为：

* 在现有架构上无法很好地解决，需要新增模块，在架构上做变更，而且需要改动 server 部分（和 notebook 共用），同时向前兼容。
* 需要对 Jupyter 以及其周边实现理解非常深刻，并且对 Jupyter 正在和将要进行的工作非常熟悉，特别是 RTC 部分，才有可能解决并将代码合入主干。
    * 而 RTC 又是一个非常复杂、暂时很难完全 release 的功能，所以如果基于 RTC 去设计解决方案，很可能你在一年内都无法上线 Jupyter 正式版
* 解决该问题需要花费的时间较多，沉没成本高，所以很多有一些想法的研究者，也只是眼巴巴希望官方能解决。

## 解决方案概述

我们这里经过权衡，设计出一套不依赖 RTC 功能解决该问题的方案。

我们主要需要改动一下几点：

1. **修改 JupyterLab Workspace 机制，改成唯一 ID**
2. **修改 JupyterLab Notebook 连接 kernel 的 UUID 逻辑，改成非随机的 UUID**
3. **修改发消息的 msg ID，改为非随机的 UUID**
4. **新增恢复逻辑，根据 message 的 UUID 将输出写入到对应 cell**
5. [可选] Server 端保留更多输出内容

改动的范围主要是 JupyterLab 前端 packages 下的各个模块，后端部分需配合做少量修改。

接下来我们依次详细说明以下改动

### 1. Workspace 修改

JupyterLab 的 workspace 实际上就是你打开多个 JupyterLab 之后，URL 中 `/lab/workspaces/***` 后面的那一串，JupyterLab 会针对不同的 Workspace 存储不同的布局等信息，不过笔者认为，Jupyter 的 Workspace 还是有一定缺陷的，接下来会进行详细分析。

**在同一个浏览器里面：**

每当你打开一个 jupyter 页面，它会做这样几件事情：

1. 获取下当前自己的 workspace 名字，如果 url 里面有 workspace 参数就用 url 里面的，如果没有就用默认名字（第一个打开的页面一般是没有的），比如 `default`
2. 在 localstorage 里面写一个 `ping`，并 `window.addEventListener('storage')` 来接受其他页面的 `pong`
    * 其他已经打开的同源 jupyter 页面，也会通过 `window.addEventListener('storage')` 监听 `ping` 并返回 `pong`，携带自己的 workspace 名字
    * 这样，这个页面就知道当前有哪些页面打开了
3. 如果其他已经打开的页面里面有重名的 workspace，这个时候：
    * 它从 `abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789` 中随机取一个随机数，组成 `auto-随机数`, 放到 workspace 参数里面，**重启整个页面**
    * 重启页面之后，重新执行步骤 1

因此：

* 除了第一个 jupyter 页面以外，其他的 Tab 打开都是需要刷新两遍的
* 并且随着冲撞的概率变高，当你打开的 Tab 越多，新开的 Tab 的刷新次数期望值越高，也就是说，你等待的时间越长。

因为一共就 62 个随机数，假设所有的随机数用完了，会发生什么？

**是的，JupyterLab 会直接无限循环 Reload, 无法使用，经过实测，确实如此。**

**在不同的浏览器里面：**

他们完全互不知情，因此：

不同浏览器直接打开的第一个 workspace 都叫做 default，后续的更多 tab 是否冲突看运气。
当你在不同的浏览器 JupyterLab，并且在 JupyterLab 里面打开了一些东西，之后原来浏览器 Tab 刷新，就会被影响到了，**这个问题在多用户集群场景下会更加显著。**

同时，JupyterLab 后端会把 workspace 配置持久化到文件中，**不过自身没有配置文件清理逻辑**

所以，针对 Workspace 我们整体的改动为：

* 去掉默认的 Workspace 逻辑。
* Workspace 的 id 变成 uuid，保证大家即使不同浏览器访问也不一致。
* 添加 Workspace 配置数据的清理逻辑，定时清理很久没有在用的 Workspace 信息。

### 2. Notebook kernel uuid

> 这部分可以结合下文“扩展阅读”，来更加深入地了解。

目前，当我们打开一个 Notebook 的时候，默认 Jupyter 就会新开一个 Kernel，然后前端 Notebook 通过建立 websocket(有一个叫做 `KernelConnection` 的类来管理 websocket) 来和 Kernel 通信。

同一个 Notebook，会随机生成一个 uuid，这个 uuid 是在 KernelConnection 管理类创建的时候生成的：

```typescript
export class KernelConnection implements Kernel.IKernelConnection {
  /**
   * Construct a kernel object.
   */
  constructor(options: Kernel.IKernelConnection.IOptions) {
    // 一般而言这里的 clientId 都是空
    this._clientId = options.clientId ?? UUID.uuid4();
    // 其他逻辑...
  }
```

然后对于非广播的消息（这里比如 kernel 的状态，它就是一个广播消息，而 python 代码块的具体执行，它是一个非广播消息，这一点更多也可以参考下文扩展阅读），是只会发送到这个 uuid 对应的 KernelConnection 客户端。

也因此，当目前刷新页面之后，**这个 uuid 就变了**，这个时候就无法对接到之前的非广播消息。

那么 JupyterLab 为什么不保持 uuid 不变呢，主要说法是：

1. 如果你打开了多个 Tab，它们使用相同的 uuid，这个必然会造成消息发送紊乱，实际上这样在建立连接的这步就会失败。
2. 有的时候，客户端会短暂断网，这个时候对应 uuid 的 websocket 会断连，假设后面再连接上，不知道是新的页面还是当前的网络恢复了，后端不知道这个信息，因此会对设计 uuid 复用逻辑有所影响。

所以，如果我们改变 uuid 的策略，需要在保证 Workspace 的唯一的前提下，设计连接 websocket 的 uuid 为 **`workspaceId - notebookId`**

### 3. Notebook cell uuid

在上一点我们更改了一个 Notebook 连接 kernel 的 uuid，但是对于其中一个 cell 的 uuid 还是随机的，这会导致即使 kernel 收到了消息，也无法得知这个消息是属于哪个 cell。

我们梳理下目前 cell 执行 python 代码的过程，接下来我们简化这个过程：

1. 当我们点击按钮，执行一个 cell 的代码的时候，Notebook 会构造这样的一条消息：

```typescript
export function createMessage<T extends Message>(options: IOptions<T>): T {
  return {
    buffers: options.buffers ?? [],
    channel: options.channel,
    content: options.content,
    header: {
      date: new Date().toISOString(),
      // msgId 一般也是没有的
      msg_id: options.msgId ?? UUID.uuid4(),
      msg_type: options.msgType,
      session: options.session,
      username: options.username ?? '',
      version: '5.2'
    },
    metadata: options.metadata ?? {},
    parent_header: options.parentHeader ?? {}
  } as T;
}
```

2. 这条消息会调用上文提到的 `KernelConnection` 的实例方法 `sendShellMessage`，`sendShellMessage` 会把消息发送给后端，同时维护一个基于 `msgId` 的绑定关系。

3. 后端 Kernel 执行完之后返回消息。返回的消息和发送的消息比较类似，比较重要的字段是 `parent_header`，`parent_header` 即发送的消息体中的 `header` 字段，其中存储 `msgId`。

4. Notebook 客户端收到消息后，通过 `parent_header` 中的 `msgId` 找到对应的 cell（这里实际上是执行一个 callback，callback 通过闭包捕获之前 cell 的相关引用），然后更新状态。

也就是说，当我们更改 cell 的 msgId 计算方式的时候，实际上大多数时候上面的逻辑都是没有变化的，只是在刷新页面之后，因为这个时候之前的回调等逻辑是不存在的，我们需要手动找到这个 cell，然后把结果输出。

### 4. 新增恢复逻辑

基于上文我们 Workspace 和 uuid 的更新设计，我们已经可以把一个页面的一条消息对应到一个 cell 中，并且这里的绑定信息是可推断、可持久化、非随机的，也就是说，我们已经有了完成恢复功能的能力。

不过实际上要完成这个逻辑，还要加的内容非常多，Notebook cell 执行相关的逻辑本身状态判断较多且调用复杂，我们新加直接输出的逻辑不仅需要融入到现有的逻辑中，而且需要考虑各种边界情况，**无异于做一次重构**。

也因为第四点过于复杂，笔者并没有完全验证，只是依靠控制台输出做了一些分析验证可行性的工作。

## 本文解决方案的不足之处

本文提供的解决方案，能够解决我们遇到的输出丢失问题，不过同时，它也是有一些副作用的，比如：

* **Notebook 修改名称逻辑会更复杂：**因为设计上是将一个 Notebook 和它的 uuid 对应起来，实际上对于 Notebook 来说，它自身的 uuid 也就是它的路径名等信息，它并不存在一个类似 `文件 uuid` 的概念，所以当文件名称变了，这个 uuid 也会跟着变化，这里的方案可能是：
  * 当 kernel 繁忙的时候不允许改名，因为这个时候改名会导致 uuid 变化，这样输出就丢失了，并把这个信息提示给用户。（事实上，这类交互在软件设计中不难见到，比如在 macOS 中，一个文件在拷贝的过程中是不允许改名的，所以笔者觉得这个也是可以被接受的）。
  * 或者在 jupyter server 记录文件的更名信息，这样当改名的时候，我们可以通过一些临时记录保证长链接不变（但这样会设计的比较复杂，不是很建议）。

## 扩展阅读

为了方便更加深入了解 JupyterLab Client->Kernel 的架构，我们补充一些扩展信息。

### 当前 Jupyter Notebook-Kernel 的架构模型

当前的 Jupyter 的 Notebook-Kernel 是一个多对一的架构，也就是说，我们可以开多个页面连接到相同的一个 Kernel，同时每个页面都执行不同的 cell，它们是都可以正确的和输出对应起来的。

![](/img/jupyter_frontend_kernel.webp)

同时，对于一些比如 kernel 状态的信息，是会广播给所有客户端的。

目前 Jupyter 通过 [jupyter_client](https://github.com/jupyter/jupyter_client) 这个包来和 kernel 进行管理和通信，这个包虽然叫做 client，但是是在 server 端，使用的，主要是和 jupyter 的 kernel 通信。

默认使用的是 [ipykernel](https://github.com/ipython/ipykernel)

启动代码：

```shell
['/path/to/bin/python', '-m', 'ipykernel_launcher', '-f', '/path/to/Jupyter/runtime/kernel-85259599-797f-4b21-b701-2a63c96fbe10.json']
```

kernel 文件中会存储通信端口等一些信息：

```
{
  "shell_port": 57748,
  "iopub_port": 57749,
  "stdin_port": 57750,
  "control_port": 57752,
  "hb_port": 57751,
  "ip": "127.0.0.1",
  "key": "ba04817a-be19696087459a6a772e6268",
  "transport": "tcp",
  "signature_scheme": "hmac-sha256",
  "kernel_name": ""
}
```

jupyter_client 使用 zmq 来做和 kernel 之间的通信。

### 其他链接

* [Reconnect to running session: keeping output](https://github.com/jupyterlab/jupyterlab/issues/2833)
* [jupyter 邮件组](https://groups.google.com/g/jupyter)
* https://github.com/jupyterlab/rtc