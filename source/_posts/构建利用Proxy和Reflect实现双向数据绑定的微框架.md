---
title: 构建利用Proxy和Reflect实现双向数据绑定的微框架
date: 2018-04-09 14:40:44
tags:
    - MVVM
---
>写在前面：这篇文章讲述了如何利用Proxy和Reflect实现双向数据绑定，个人系Vue早期玩家，写这个小框架的时候也没有参考Vue等源代码，之前了解过其他实现，但没有直接参考其他代码，如有雷同，纯属巧合。

代码下载地址：[这里下载](https://github.com/aircloud/Polar.js)

### 综述

*关于Proxy和Reflect的资料推荐阮老师的教程:http://es6.ruanyifeng.com/ 这里不做过多介绍。*

实现双向数据绑定的方法有很多，也可以参考本专栏之前的其他实现，我之所以选择用Proxy和Reflect，一方面是因为可以大量节约代码，并且简化逻辑，可以让我把更多的经历放在其他内容的构建上面，另外一方面本项目直接基于ES6，用这些内容也符合面向未来的JS编程规范，第三点最后说。

由于这个小框架是自己在PolarBear这个咖啡馆在一个安静的午后开始写成，暂且起名Polar，日后希望我能继续完善这个小框架，给添加上更多有趣的功能。

首先我们可以看整体功能演示：  
[一个gif动图，如果不能看，请点击[这里的链接](https://www.10000h.top/images/data_img/gif1.gif)]

![](https://www.10000h.top/images/data_img/gif1.gif)

### 代码分析

我们要做这样一个小框架，核心是要监听数据的改变，并且在数据的改变的时候进行一些操作，从而维持数据的一致。

我的思路是这样的：

* 将所有的数据信息放在一个属性对象中(this._data),之后给这个属性对象用Proxy包装set,在代理函数中我们更新属性对象的具体内容，同时通知所有监听者，之后返回新的代理对象(this.data)，我们之后操作的都是新的代理对象。
* 对于input等表单，我们需要监听input事件，在回调函数中直接设置我们代理好的数据对象，从而触发我们的代理函数。
* 我们同时也应该支持事件机制，这里我们以最常用的click方法作为例子实现。

下面开始第一部分，我们希望我们之后使用这个库的时候可以这样调用:

```
<div id="app">
    <form>
        <label>name:</label>
        <input p-model = "name" />
    </form>
    <div>name:{{name}} age:{{age}}</div>
    <i>note:{{note}}</i><br/>
    <button p-click="test(2)">button1</button>
</div>
<script>
 var myPolar = new Polar({
        el:"#app",
        data: {
            name: "niexiaotao",
            age:16,
            note:"Student of Zhejiang University"
        },
        methods:{
            test:function(e,addNumber){
                console.log("e:",e);
                this.data.age+=Number(addNumber);
            }
        }
});
</script>
```

没错，和Vue神似吧，所以这种调用方式应当为我们所熟悉。

我们需要建立一个Polar类，这个类的构造函数应该进行一些初始化操作:

```
 constructor(configs){
        this.root = this.el = document.querySelector(configs.el);
        this._data = configs.data;
        this._data.__bindings = {};
        //创建代理对象
        this.data = new Proxy(this._data, {set});
        this.methods = configs.methods;

        this._compile(this.root);
}
```

这里面的一部份内容是直接将我们传入的configs按照属性分别赋值，另外就是我们创建代理对象的过程，最后的`_compile`方法可以理解为一个私有的初始化方法。

实际上我把剩下的内容几乎都放在`_compile`方法里面了，这样理解起来方便，但是之后可能要改动。

我们还是先不能看我们代理的set该怎么写，因为这个时候我们还要先继续梳理思路：

假设我们这样`<div>name:{{name}}</div>`将数据绑定到dom节点，这个时候我们需要做什么呢，或者说，我们通过什么方式让dom节点和数据对应起来，随着数据改变而改变。

看上文的`__bindings`。这个对象用来存储所有绑定的dom节点信息，`__bindings`本身是一个对象，每一个有对应dom节点绑定的数据名称都是它的属性，对应一个数组，数组中的每一个内容都是一个绑定信息，这样，我们在自己写的set代理函数中，我们一个个调用过去，就可以更新内容了：

```
dataSet.__bindings[key].forEach(function(item){
       //do something to update...
});
```

我这里创建了一个用于构造调用的函数，这个函数用于创建存储绑定信息的对象：

```
function Directive(el,polar,attr,elementValue){
    this.el=el;//元素本身dom节点
    this.polar = polar;//对应的polar实例
    this.attr = attr;//元素的被绑定的属性值，比如如果是文本节点就可以是nodeValue
    this.el[this.attr] = this.elementValue = elementValue;//初始化
}
```

这样，我们的set可以这样写:

```
function set(target, key, value, receiver) {
    const result = Reflect.set(target, key, value, receiver);
    var dataSet = receiver || target;
    dataSet.__bindings[key].forEach(function(item){
        item.el[item.attr] = item.elementValue = value;
    });
    return result;
}
```

接下来可能还有一个问题：我们的`{{name}}`实际上只是节点的一部分，这并不是节点啊，另外我们是不是还可以这么写：`<div>name:{{name}} age:{{age}}</div>`？

关于这两个问题，前者的答案是我们将`{{name}}`替换成一个文本节点，而为了应对后者的情况，我们需要将两个被绑定数据中间和前后的内容，都变成新的文本节点，然后这些文本节点组成文本节点串。(这里多说一句，html5的normalize方法可以将多个文本节点合并成一个，如果不小心调用了它，那我们的程序就要GG了)

所以我们在`_compile`函数首先：

```
var _this = this;

        var nodes = root.children;

        var bindDataTester = new RegExp("{{(.*?)}}","ig");

        for(let i=0;i<nodes.length;i++){
            var node=nodes[i];

            //如果还有html字节点，则递归
            if(node.children.length){
                this._compile(node);
            }

            var matches = node.innerHTML.match(bindDataTester);
            if(matches){
                var newMatches = matches.map(function (item) {
                    return  item.replace(/{{(.*?)}}/,"$1")
                });
                var splitTextNodes  = node.innerHTML.split(/{{.*?}}/);
                node.innerHTML=null;
                //更新DOM，处理同一个textnode里面多次绑定情况
                if(splitTextNodes[0]){
                    node.append(document.createTextNode(splitTextNodes[0]));
                }
                for(let ii=0;ii<newMatches.length;ii++){
                    var el = document.createTextNode('');
                    node.appendChild(el);
                    if(splitTextNodes[ii+1]){
                        node.append(document.createTextNode(splitTextNodes[ii+1]));
                    }
                //对数据和dom进行绑定
                let returnCode = !this._data.__bindings[newMatches[ii]]?
                    this._data.__bindings[newMatches[ii]] = [new Directive(el,this,"nodeValue",this.data[newMatches[ii]])]
                    :this._data.__bindings[newMatches[ii]].push(new Directive(el,this,"nodeValue",this.data[newMatches[ii]]))
                }
            }

```

这样，我们的数据绑定阶段就写好了，接下来，我们处理`<input p-model = "name" />`这样的情况。

这实际上是一个指令，我们只需要当识别到这一个指令的时候，做一些处理，即可：

```
if(node.hasAttribute(("p-model"))
                && node.tagName.toLocaleUpperCase()=="INPUT" || node.tagName.toLocaleUpperCase()=="TEXTAREA"){
                node.addEventListener("input", (function () {

                    var attributeValue = node.getAttribute("p-model");

                    if(_this._data.__bindings[attributeValue]) _this._data.__bindings[attributeValue].push(new Directive(node,_this,"value",_this.data[attributeValue])) ;
                    else _this._data.__bindings[attributeValue] = [new Directive(node,_this,"value",_this.data[attributeValue])];

                    return function (event) {
                        _this.data[attributeValue]=event.target.value
                    }
                })());
}
```

请注意，上面调用了一个`IIFE`，实际绑定的函数只有返回的函数那一小部分。

最后我们处理事件的情况：`<button p-click="test(2)">button1</button>`

实际上这比处理`p-model`还简单，但是我们为了支持函数参数的情况，处理了一下传入参数，另外我实际上将`event`始终作为一个参数传递，这也许并不是好的实践，因为使用的时候还要多注意。

```
if(node.hasAttribute("p-click")) {
                node.addEventListener("click",function(){
                    var attributeValue=node.getAttribute("p-click");
                    var args=/\(.*\)/.exec(attributeValue);
                    //允许参数
                    if(args) {
                        args=args[0];
                        attributeValue=attributeValue.replace(args,"");
                        args=args.replace(/[\(\)\'\"]/g,'').split(",");
                    }
                    else args=[];
                    return function (event) {
                        _this.methods[attributeValue].apply(_this,[event,...args]);
                    }
                }());
}
```

现在我们已经将所有的代码分析完了，是不是很清爽？代码除去注释约100行，所有源代码可以在[这里下载](https://github.com/aircloud/Polar.js)。这当然不能算作一个框架了，不过可以学习学习，这学期有时间的话，还要继续完善，也欢迎大家一起探讨。

一起学习，一起提高，做技术应当是直接的，有问题欢迎指出～

---


最后说的第三点：是自己还是一个学生，做这些内容也仅仅是出于兴趣，因为找暑期实习比较艰难，在等待鹅厂面试间隙写的这个程序，压压惊(然而并没有消息)。