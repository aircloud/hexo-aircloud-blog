---
title: 一些性能相关的 JavaScript 代码编写建议规范
date: 2021-08-29 13:59:00
tags:
	- javascript
---

本文对一些日常编写 JavaScript 的过程中，一些有助于提高代码性能的规范进行罗列。

> 本文比较零碎，不作为规范提议，仅作为交流参考。

### 1. 使用解构赋值，减少中间变量。

对于一些比如变量替换的场景，我们使用解构赋值，可以省略中间变量，整体代码也会更加清晰。

```
let a = 3;
let b = 4;
[b, a] = [a, b];
```

### 2. 通过条件判断提前返回

这里主要是提醒大家如何写好 if 语句。

实际上， 在编写复杂的 if 语句之前，我们应该考虑是否可以**逻辑外化**：

即尽可能的将代码的复杂逻辑向外推，例如抽离成多个函数，而不是在程序里面进行过多判断。有一种比较典型的不合理的重用是把大量的逻辑都堆叠到一个函数里面，然后提供一个很复杂的功能。我认为更好的做法应当是分离成更多的模块。

经过以上思考之后，我们可能还有一些 if 语句，一般的原则是：

* if 语句先简单，后复杂。
* if 语句，可以提前返回即提前返回，减少复杂的嵌套。

```
// nice:
if (condition1) {
  // do something
  return;
}

if (condition2) {
  // do something
  return;
}

other_function();

// bad:
if (condition1) {
  // do something
} else {
  if (condition2) {
    // do something
  } else {
    other_function();
  }
}
```

### 3. 尽量避免在循环体内包裹函数表达式

函数表达式会生成对应的函数对象，如果我们在循环体内去做这个事情，很可能会造成额外的浪费。

```
// nice:
function callback() {
}

const len = nodelist.length;
for(let i = 0; i < len; i += 1) {
  addListener(nodelist[i], callback);
}

// bad:
for(let i = 0; i < nodelist.length; i += 1) {
  addListener(nodelist[i], function() {});
}
```

### 4. 对循环体内的不变值，在循环体外使用缓存

这一条其实是对上一条的补充，实际上是同样的原理，即希望我们在循环体内尽量保持逻辑的简单，减少重复的 cpu 时间和内存的消耗。

### 5. 清空数组使用 .length = 0

这样写可以方便我们清空一个 const 数组。

```
const a = [1,2,3,4];
// 如果使用 a = [] 会报错
a.length = 0;
```

### 6. 不得为了编写方便，将并行的 io 串行化

虽然现在 JavaScript 有了 async/await，但是我发现很多同学会对此滥用，一个很常见的清空就是将可以并行的操作串行化了:

```
let res1 = await process1();
let res2 = await process2();
next(res1, res2);
```

这个时候，虽然写代码方便，但是这样写是不可取的，Promise 提供了若干的方便我们处理并行任务的[方法](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Promise#)，我认为这些都是必须要了解的。


### 7. 禁止直接使用 eval

eval 的安全性非常差，事实上有很多已知的 xss 等漏洞都和 eval 有关，所以我们在实际场景中避免使用 eval。

如下为一个例子，使用了 eval 函数，由于其执行代码的作用域为本地作用域，所以对我们的本地变量进行了修改并且可以生效：

```
let tip = "请重新登录"
let otherCode = `tip = "请前往 xxx.com 重新登录"`
eval(otherCode);
```

一些取代方式：

我们可以使用 `new Function` 的方式来代替 eval，这样至少可以进行作用域的隔离，相对会安全一些（但是请注意其仍然会可能影响到全局变量）。

### 8. 浏览器环境中，尽量避免使用 document.all、document.querySelectorAll

类似的 all 相关操作都要避免使用，由于我们很难控制随着项目发展内容会有多少，所以我们最好一开始就不要留下随着项目内容增加性能越来越差的隐患。

### 9. 获取元素的样式，尽量使用 getComputedStyle 或 currentStyle

通过 style 只能获得内联定义或通过 JavaScript 直接定义的样式，通过 CSS class 设置的样式无法直接获取。

### 10. 尽可能通过为元素添加预定义的 ClassName 来改变元素样式，避免直接操作 style 进行设置。

直接操作 style，会比较混乱，而且有的时候还会忘记写单位，导致实际上不管用。

