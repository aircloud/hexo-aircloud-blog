---
title: 深入浏览器web渲染与优化
date: 2017-08-27 17:37:22
tags:
    - 性能优化
---
>本文主要分析和总结web内核渲染的相关内容，以及在这方面前端可以做的性能优化工作。

文章主要分为以下几个部分：

* blink内核的渲染机制
* chrome内核架构变迁
* 分层渲染
* 动画 & canvas & WebGl

*这里的前两部分可能会有些枯燥，如果是前端工程师并且想立即获得实际项目的建议的，可以直接阅读第三部分和第四部分*

### blink内核的渲染机制

blink内核是Google基于Webkit内核开发的新的分支，而实际上，目前Chrome已经采用了blink内核，所以，我们接下来的有关分析大多基于blink内核的浏览器(Chrome)，就不再详细指明，当然，部分内容也会涉及到腾讯研发的X5内核(X5内核基于安卓的WebView，目前已经在手机QQ等产品中使用，基于X5内核的项目累计有数亿UV，上百亿PV)。

一个页面的显示，实际上主要经历了下面的四个流程：

加载 => 解析 => 排版 => 渲染

实际上，这里的渲染主要是指排版之后到最后的上屏绘制(这个时候内容已经排版好了)，一部分前端工程师通常会把一部分的排版工作理解到“渲染”的流程中(也就是下图中全部工作)，实际上这个理解是不准确的。

![](https://www.10000h.top/images/data_img/webRender/P6.PNG)

目前，浏览器的渲染采用的是分块渲染的机制，所谓的分块渲染的机制，其实应该这么理解：

* 浏览器首先把整个网页分成一些低分辨率的块，再把网页分成高分辨率的块，然后给这些块排列优先级。
* 处在可视区域内的低分辨率块的优先级会比较高，会被较先绘制。
* 之后浏览器会把高分辨率的块进行绘制，同样也是先绘制处于可视区域内的，再绘制可视区域外的(由近到远)。

以上讲的这些策略可以使可以使得浏览器优先展示可视区域内的内容，并且先展示大致内容，再展示高精度内容(当然，由于这个过程比较快，实际上我们大多时候是感受不到的)。

另外这里值得提醒的一点是，分块的优先级是会根据到可视区域的距离来决定的，所以有些横着的内容(比如banner的滚动实现，通常会设置横向超出屏幕来表示隐藏)，也是会按照到可视区域的距离来决定优先级的。

绘制的过程，可以被硬件加速，这里硬件加速的主要手段主要是指：

* 硬件加速合成上屏
* 2D Canvas、Video的硬件加速
* GPU光栅化
	* GPU光栅化速度更快，内存和CPU的消耗更少
	* 目前还没有办法对包含复杂矢量绘制的页面进行GPU光栅化
	* GPU光栅化是未来趋势


### chrome内核架构变迁

在渲染架构上，chrome也是经历了诸多变迁，早期的Chrome是这样的：

![](https://www.10000h.top/images/data_img/webRender/P1.PNG)

早期的chrome的架构实际上有以下缺点：

* Renderer线程任务繁重
* 无法实时响应缩放滑动操作
* 脏区域与滑动重绘区域有冲突
	* 这里举个场景，假设一个gif，这个时候如果用户滑动，滑动新的需要绘制的内容和gif下一帧内容就会产生绘制冲突

当然，经过一系列的发展，Chrome现在是这样的：

![](https://www.10000h.top/images/data_img/webRender/P2.PNG)

在安卓上，Android 4.4的 Blink内核架构如下(4.4之前并不支持OpenGL)

![](https://www.10000h.top/images/data_img/webRender/P3.PNG)

当然，这种架构也有如下缺点：

* UI线程过于繁忙
* 无法支持Canvas的硬件加速以及WebGL

所以，后期发展成了这样：

![](https://www.10000h.top/images/data_img/webRender/P4.PNG)

总结看来，内核发展的趋势是：

* 多线程化(可以充分利用多核心CPU)
* 硬件加速(可以利用GPU)

### 分层渲染

在阅读这一章之前，我建议读者先去亲自体验一下所谓的“分层渲染”：

>打开Chrome浏览器，打开控制台，找到"Layers"，如果没有，那么在控制台右上角更多的图标->More tools 找到"Layers"，然后随便找个网页打开即可

网页的分层渲染流程主要是下面这样的：

![](https://www.10000h.top/images/data_img/webRender/P7.PNG)

(*注意：多个RenderObject可能又会对应一个或多个RenderLayer*)

既然才用了分层渲染，那么肯定可以来分层处理，分层渲染有如下优点：

* 减少不必要的重新绘制
* 可以实现较为复杂的动画
* 能够方便实现复杂的CSS样式

当然，分层渲染是会很影响渲染效率的，可以有好的影响，使用不当也会有差的影响，我们需要合理的控制和使用分层：

* 如果小豆腐块分层较多，页面整体的分层数量较大，会导致每帧渲染时遍历分层和计算分层位置耗时较长啊(比较典型的是腾讯网移动端首页)。
* 如果可视区域内分层太多且需要绘制的面积太大，渲染性能非常差，甚至无法达到正常显示的地步(比如有一些全屏H5)。
* 如果页面几乎没有分层，页面变化时候需要重绘的区域较多。元素内容无变化只有位置发生变化的时候，可以利用分层来避免重绘。

那么，是什么原因可以导致分层呢？目前每一个浏览器或者不同版本的浏览器分层策略都是有些不同的(虽然总体差不太多)，但最常见的几个分层原因是：transform、Z-index；还有可以使用硬件加速的video、canvas；fixed元素；混合插件(flash等)。关于其他更具体的内容，可以见下文。

```
//注:Chrome中符合创建新层的情况：
Layer has 3D or perspective transform CSS properties(有3D元素的属性)
Layer is used by <video> element using accelerated video decoding(video标签并使用加速视频解码)
Layer is used by a <canvas> element with a 3D context or accelerated 2D context(canvas元素并启用3D)
Layer is used for a composited plugin(插件，比如flash)
Layer uses a CSS animation for its opacity or uses an animated webkit transform(CSS动画)
Layer uses accelerated CSS filters(CSS滤镜)
Layer with a composited descendant has information that needs to be in the composited layer tree, such as a clip or reflection(有一个后代元素是独立的layer)
Layer has a sibling with a lower z-index which has a compositing layer (in other words the layer is rendered on top of a composited layer)(元素的相邻元素是独立layer)
```

最后，我们总结一下如何合理的设计分层：分层总的原则是，减少渲染重绘面积与减少分层个数和分层总面积：

* 相对位置会发生变化的元素需要分层(比如banner图、滚动条)
* 元素内容更新比较频繁的需要分层(比如页面中夹杂的倒计时等)
* 较长较大的页面注意总的分层个数
* 避免某一块区域分层过多，面积过大

(*如果你给一个元素添加上了-webkit-transform: translateZ(0);或者 -webkit-transform: translate3d(0,0,0);属性，那么你就等于告诉了浏览器用GPU来渲染该层，与一般的CPU渲染相比，提升了速度和性能。(我很确定这么做会在Chrome中启用了硬件加速，但在其他平台不做保证。就我得到的资料而言，在大多数浏览器比如Firefox、Safari也是适用的)*)

另外值得一提的是，X5对分层方面做了一定的优化工作，当其检测到分层过多可能会出现显示问题的时候会进行层合并，牺牲显示性能换取显示正确性。

最后再提出一个小问题：

以下哪种渲染方式是最优的呢？

![](https://www.10000h.top/images/data_img/webRender/P8.PNG)

这里实际上后者虽然在分层上满足总体原则，但是之前讲到浏览器的分块渲染机制，是按照到可视区域的距离排序的，考虑到这个因素，实际上后者这种方式可能会对分块渲染造成一定的困扰，并且也不是最优的。

### 动画 & canvas & WebGl

讲最后一部分开始，首先抛出一个问题：CSS动画 or JS动画?

对内核来说，实际上就是Renderer线程动画还是Compositor线程动画，二者实际上过程如下：

![](https://www.10000h.top/images/data_img/webRender/P9.PNG)

所以我们可以看出，Renderer线程是比Compositor线程动画性能差的(在中低端尤其明显)

另外，无论是JS动画还是CSS动画，动画过程中的重绘以及样式变化都会拖慢动画执行以及引起卡顿
以下是一些不会触发重绘或者排版的CSS动画属性：

* cursor
* font-variant
* opacity
* orphans
* perspective
* perspecti-origin
* pointer-events
* transform
* transform-style
* widows

想要了解更多内容，可以参考[这里](https://csstriggers.com/)

这方面最终的建议参考如下：

* 尽量使用不会引起重绘的CSS属性动画，例如transform、opacity等
* 动画一定要避免触发大量元素重新排版或者大面积重绘
* 在有动画执行时，避免其他动画不相关因素引起排版和重绘


#### requestAnimationFrame

另外当我们在使用动画的时候，为了避免出现掉帧的情况，最好采用requestAnimationFrame这个API，这个API迎合浏览器的流程，并且能够保证在下一帧绘制的时候上一帧一定出现了：

![](https://www.10000h.top/images/data_img/webRender/P11.PNG)

### 3D canvas

还有值得注意的是，有的时候我们需要涉及大量元素的动画(比如雪花飘落、多个不规则图形变化等)，这个时候如果用CSS动画，Animation动画的元素很多。，导致分层个数非常多，浏览器每帧都需要遍历计算所有分层，导致比较耗时、

这个时候该怎么办呢？

2D canvas上场。 

和CSS动画相比，2D canvas的优点是这样的：

* 硬件加速渲染
* 渲染流程更优

其渲染流程如下：

![](https://www.10000h.top/images/data_img/webRender/P10.PNG)

实际上以上流程比较耗时的是JS Call这一部分，执行opengl的这一部分还是挺快的。

HTML 2D canvas 主要绘制如下三种元素：

* 图片
* 文字
* 矢量

这个过程可以采用硬件加速，硬件加速图片绘制的主要流程：

![](https://www.10000h.top/images/data_img/webRender/P12.PNG)

硬件加速文字绘制的主要流程：

![](https://www.10000h.top/images/data_img/webRender/P13.PNG)

但对于矢量绘制而言，简单的图形，比如点、直线等可以直接使用OpenGL渲染，复杂的图形，如曲线等，无法采用OpenGL绘制。

对于绘制效率来说，2D Canvas对绘制图片效率较高，绘制文字和矢量效率较低(**所以建议是，我们如果能使用贴图就尽量使用贴图了**)

还有，有的时候我们需要先绘制到离屏canvas上面，然后再上屏，这个可以充分利用缓存。

### 3D canvas(WebGL)

目前，3D canvas(WebGL)的应用也越来越多，对于这类应用，现在已经有了不少已经成型的庫:


* 通用引擎：threeJS、Pixi
* VR视频的专业引擎：krpano、UtoVR
* H5游戏引擎：Egret、Layabox、Cocos

WebGL虽然包含Web，但本身对前端的要求最低，但是对OpenGL、数学相关的知识要求较高，所以如果前端工程师没有一定的基础，还是采用现在的流行庫。

X5内核对于WebGl进行了性能上和耗电上的优化，并且也对兼容性错误上报和修复做了一定的工作。

___

本文参考腾讯内部讲座资料整理而成，并融入一部分笔者的补充，谢绝任何形式的转载。

其他优质好文：

[Javascript高性能动画与页面渲染](http://qingbob.com/javascript-high-performance-animation-and-page-rendering/)


