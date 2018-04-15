---
title: JS的静态作用域、子程序引用环境与参数传递类型
date: 2017-01-11 1:33:26
tags:
    - javascript
---
#### 静态作用域

我们先来看下面这个小程序：

```
 //JS版本：
 function sub1() {
        var x;
        function sub2() { alert(x); }
        function sub3() { var x; x=3; sub4(sub2); }
        function sub4(subx) { var x; x=4; subx(); }
        x = 1;
        sub3();
    }

    sub1();
    
 #Python版本
def sub1():
    def sub2():
        print x
    def sub3():
        x=3
        sub4(sub2)
    def sub4(subx):
        x=4
        subx()
    x = 1
    sub3()

sub1()   
```

不用亲自运行，实际上输出结果都是1，这可能不难猜到，但是需要解释一番，鉴于Python和JS在这一点上表现的类似，我就以JS来分析。

我们知道，JS是静态作用域的，所谓静态作用域就是作用域在编译时确定，所以sub2中引用的x，实际上和x=3以及x=4的x没有任何关系，指向第二行的var x;

#### 子程序的引用环境

实际上这里面还有一个子程序(注：子程序和函数不是很一样，但我们可以认为子程序包括函数，也约等于函数)的概念，sub2、sub3、sub4都是子程序，对于允许嵌套子程序的语言，应该如何使用执行传递的子程序的引用环境？

* 浅绑定：如果这样的话，应该输出4，这对动态作用域的语言来说比较自然。
* 深绑定：也就是输出1的情况，这对静态作用域的语言来说比较自然。
* Ad hoc binding: 这是第三种，将子程序作为实际参数传递到调用语句的环境。

#### 参数传递类型

参数传递类型我们普遍认为有按值传递和按引用传递两种，实际上不止。

下面是一张图：

![](https://www.10000h.top/images/call.png)

这张图对应的第一种传递方式，叫做Pass-by-Value(In mode)，第二种是Pass-by-Result(Out mode)，第三种是Pass-by-Value-Result(Inout mode),图上说的比较明白，实际上如果有result就是说明最后把结果再赋值给参数。

第二种和第三种编程语言用的少，原因如下：
>Potential problem: sub(p1, p1)   
With the two corresponding formal parameters having different names, whichever formal parameter is copied back last will represent current value of p1

