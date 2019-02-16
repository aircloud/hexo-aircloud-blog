---
title: 入门WebAssembly以及使用其进行图像卷积处理
date: 2019-02-16 19:15:51
tags:
    - WebAssembly
---

> WebAssembly 出现有很长时间了，但是由于日常工作并无直接接触，因此一直疏于尝试，最近终于利用一些业余时间简单入门了一下，因此在此总结。

### 简介

首先我们需要知道 WebAssembly 是一个什么东西，其实际是一个字节码编码方式，比较接近机器码（但是又无法直接执行），这样可以方便地做到跨平台同时省去像 JavaScript 等语言的解释时间，所以是有一定优势的，使用起来其实也比较灵活，凡是可以转化成字节码的，都是可以使用 WebAssembly。

以下仅列举部分可以支持 WebAssembly 转化的语言：

* [AssemblyScript](https://github.com/AssemblyScript/assemblyscript): 语法和 TypeScript 一致(事实上，其是 Typescript 的一个子集)，对前端来说学习成本低，为前端编写 WebAssembly 最佳选择；
* c\c++: 官方推荐的方式，详细使用见[文档](http://webassembly.org.cn/getting-started/developers-guide/);
* [Rust](https://www.rust-lang.org/): 语法复杂、学习成本高，对前端来说可能会不适应。详细使用见[文档](https://github.com/rust-lang-nursery/rust-wasm);
* [Kotlin](http://kotlinlang.org/): 语法和 Java、JS 相似，语言学习成本低，详细使用见[文档](https://kotlinlang.org/docs/reference/native-overview.html);
* [Golang](https://golang.org/): 语法简单学习成本低。但对 WebAssembly 的支持还处于未正式发布阶段，详细使用见[文档](https://blog.gopheracademy.com/advent-2017/go-wasm/)。

尝试使用 WebAssembly 官方推荐的方式，我们首先可以在[这里](http://webassembly.org.cn/getting-started/developers-guide/)来下载。

如果用腾讯内网有的文件是下载不下来的，这个时候我们可以给命令行增加一个代理（如果我们用的 Fiddler 或 Charles，开启的时候默认命令行也可以走代理，如果是 Whistle，我们需要手动设置代理），有些文件我们还可以下载好之后使用文件代理。

```
export https_proxy="http://127.0.0.1:8899"
export http_proxy="http://127.0.0.1:8899"
// 文件代理：
https://s3.amazonaws.com/mozilla-games/emscripten/packages/node-v8.9.1-darwin-x64.tar.gz file:///Users/niexiaotao/node-v8.9.1-darwin-x64.tar.gz
```

## 初体验

这里考虑到前端同学的上手难度，我们先使用 AssemblyScript 写一个极小的例子，一个斐波那契函数：

```
export function f(x: i32): i32 {
    if (x === 1 || x === 2) {
        return 1;
    }
    return f(x - 1) + f(x - 2)
}
```

通过类似 `asc f.ts -o f.wasm` 这样的命令编译成 f.wasm 之后，我们可以分别在 Node 环境和浏览器环境来执行：

Node：

```
const fs = require("fs");
const wasm = new WebAssembly.Module(
    fs.readFileSync(__dirname + "/f.wasm"), {}
);
const myModule = new WebAssembly.Instance(wasm).exports;
console.log(myModule.f(12));
```

浏览器：

```
fetch('f.wasm') // 网络加载 f.wasm 文件
        .then(res => res.arrayBuffer()) // 转成 ArrayBuffer
        .then( buffer =>
            WebAssembly.compile(buffer)
        )
        .then(module => { // 调用模块实例上的 f 函数计算
            const instance = new WebAssembly.Instance(module);
            const { f } = instance.exports;
            console.log('instance:', instance.exports);
            console.log('f(20):', f(20));
        });
```

于是，我们完成了 WebAssembly 的初体验。

当然，这个例子太简单了。

## 使用 WebAssembly 进行图像卷积处理

实际上，WebAssembly 的目的在于解决一些复杂的计算问题，优化 JavaScript 的执行效率。所以我们可以使用 WebAssembly 来处理一些图像或者矩阵的计算问题。

接下来，我们通过 WebAssembly 来处理一些图像的卷积问题，用于图像的风格变换，我们最终的例子可以在[这里](http://assembly.niexiaotao.com/)体验。

每次进行卷积处理，我们的整个流程是这样的：

* 将原图像使用 canvas 绘制到屏幕上。
* 使用 `getImageData` 获取图像像素内容，并转化成类型数组。
* 将上述类型数组通过共享内存的方式传递给 WebAssembly 部分。
* WebAssembly 部分接收到数据，进行计算，并且通过共享内存的方式返回。
* 将最终结果通过 canvas 画布更新。

上述各个步骤中，绘制的部分集中在 JavaScript 端，而计算的部分集中在 WebAssembly，这两部分相互比较独立，可以分开编写，而双端数据通信是一个比较值得注意的地方，事实上，我们可以通过 ArrayBuffer 来实现双端通信，简单的说，JavaScript 端和 WebAssembly 可以共享一部分内存，并且都拥有读写能力，当一端写入新数据之后，另一段也可以读到，这样我们就可以进行通信了。

关于数据通信的问题，这里还有一个比较直白的[科普文章](https://segmentfault.com/a/1190000010434237)，可以参考。

在这里没有必要对整个项目代码进行展示，因此可以参考（[代码地址](https://github.com/aircloud/assemConvolution)），我们这里仅仅对部分关键代码进行说明。

### 共享内存

首先，我们需要声明一块共享内存，这其实可以使用 WebAssembly 的 API 来完成：

```
let memory = new WebAssembly.Memory({ initial: ((memSize + 0xffff) & ~0xffff) >>> 16 });
```

这里经过这样的比较复杂的计算是因为 initial 传入的是以 page 为单位，详细可以参考[这里](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/WebAssembly/Memory)，实际上 memSize 即我们共享内存的字节数。

然后这里涉及到 memSize 的计算，我们主要需要存储三块数据：卷积前的数据、卷积后的数据（由于卷积算法的特殊性以及为了避免更多麻烦，这里我们不进行数据共用），还有卷积核作为参数需要传递。

这里我们共享内存所传递的数据按照如下的规则进行设计：

![](http://niexiaotao.cn/img/ker1.jpg)

传递给 WebAssembly 端的方式并不复杂，直接在 `WebAssembly.instantiate` 中声明即可。 

```
fetch(wasmPath)
     .then(response => response.arrayBuffer())
     .then(buffer => WebAssembly.instantiate(buffer, {
         env: {
             memory,
             abort: function() {}
         },
         Math
     })).then(module => {})
                
```

然后我们在 AssemblyScript 中就可以进行读写了：

```
//写：
store<u32>(position, v) // position 为位置

//读：
load<u32>(position) // position 为位置
```

而在 JavaScript 端，我们也可以通过 `memory.buffer` 拿到数据，并且转化成类型数组：

```
let mem = new Uint32Array(memory.buffer)
//通过 mem.set(data) 可以在 JavaScript 端进行写入操作
```

这样，我们在 JavaScript 端和 AssemblyScript 端的读写都明晰了。

这里需要注意的是，**JS端采用的是大端数据格式，而 AssemblyScript 中采用的是小端，因此其颜色数据格式为 AGBR**

### 卷积计算

我们所采用的卷积计算本身算法比较简单，并且不是本次的重点，但是这里需要注意的是：

* 我们无法直接在 AssemblyScript 中声明数组并使用，因此除了 Kernel 通过共享内存的方式传递过来以外，我们应当尽量避免声明数组使用（虽然也有使用非共享内存数组的相关操作，但是使用起来比较繁琐）
* 卷积应当对 R、G、B 三层单独进行，我这里 A 层不参与卷积。

以上都在代码中有所体现，参考相关代码便可明了。

卷积完成后，我们通过共享内存的方法写入类型数组，然后在 JavaScript 端合成数据，调用 `putImageData` 上屏即可。

### 其他

当然，本次图像卷积程序仅仅是对 Webassembly 和 AssemblyScript 的初步尝试，笔者也在学习阶段，如果上述说法有问题或者你想和我交流，也欢迎留言或者提相关 issue。
