---
title: 十条编写优化的 JavaScript 代码的建议
date: 2018-05-29 16:59:40
tags:
    - javascript
---

本文总结了十条编写优秀的 JavaScript 代码的习惯，主要针对 V8 引擎：

1.始终以相同的顺序实例化对象属性，以便可以共享隐藏类和随后优化的代码。V8 在对 js 代码解析的时候会有构建隐藏类的过程，以相同的顺序实例化（属性赋值）的对象会共享相同的隐藏类。下面给出一个不好的实践：

```javascript
function Point(x, y) {
    this.x = x;
    this.y = y;
}
var p1 = new Point(1, 2);
p1.a = 5;
p1.b = 6;
var p2 = new Point(3, 4);
p2.b = 7;
p2.a = 8;
// 由于 a 和 b 的赋值顺序不同，p1 和 p2 无法共享隐藏类
```

2.避免分配动态属性。在实例化之后向对象添加属性将强制隐藏类更改，并减慢为先前隐藏类优化的所有方法。相反，在其构造函数中分配所有对象的属性。  

3.重复执行相同方法的代码将比仅执行一次（由于内联缓存）执行许多不同方法的代码运行得更快。  

4.避免创建稀疏数组。稀疏数组由于不是所有的元素都存在，因此是一个哈希表，因此访问稀疏数组中的元素代价更高。另外，尽量不要采用预分配数量的大数组，更好的办法是随着你的需要把它的容量增大。最后，尽量不要删除数组中的元素，它会让数组变得稀疏。  

5.标记值：V8采用32位来表示对象和数字，其中用一位来区别对象（flag = 0）或数字（flag = 1），因此这被称之为 SMI (Small Integer)因为它只有31位。因此，如果一个数字大于31位，V8需要对其进行包装，将其变成双精度并且用一个对象来封装它，因此应该尽量使用31位有符号数字从而避免昂贵的封装操作。  

6.检查你的依赖，去掉不需要 import 的内容。  

7.将你的代码分割成一些小的 chunks ，而不是整个引入。 
 
8.尽可能使用 defer 来推迟加载 JavaScript，另外只加载当前路由需要的代码段。
  
9.使用 dev tools 和 DeviceTiming 来寻找代码瓶颈。  

10.使用诸如Optimize.js这样的工具来帮助解析器决定何时需要提前解析以及何时需要延后解析。  
  
以上内容来源：
* [How JavaScript works: Parsing, Abstract Syntax Trees (ASTs) + 5 tips on how to minimize parse time](https://blog.sessionstack.com/how-javascript-works-parsing-abstract-syntax-trees-asts-5-tips-on-how-to-minimize-parse-time-abfcf7e8a0c8)
* [How JavaScript works: inside the V8 engine + 5 tips on how to write optimized code](https://blog.sessionstack.com/how-javascript-works-inside-the-v8-engine-5-tips-on-how-to-write-optimized-code-ac089e62b12e)

