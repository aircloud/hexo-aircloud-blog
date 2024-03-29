---
title: 了解 StackOverFlow 上面最受欢迎的语言 Rust
date: 2021-09-02 19:15:51
tags:
    - Rust
---

本文希望从宏观角度，来介绍和分析 Rust 语言。

Rust 是一门**专注安全**的现代系统编程语言，发布于 2015 年。

自从 2015 年发布起，Rust 就一直是 StackOverFlow 上面最受欢迎的语言，而且和第二名还能拉开不小的差距，例如最近两年的统计数据：

[2020](https://insights.stackoverflow.com/survey/2020#most-loved-dreaded-and-wanted):
![2020_rust](/img/rust_2020.png)

[2021](https://insights.stackoverflow.com/survey/2021#most-loved-dreaded-and-wanted-language-love-dread):
![2021_rust](/img/rust_2021.png)

接下来笔者通过性能、可靠性、生产力、面向前端友好等几个维度来介绍 Rust，之后会对 Rust 的部分重点语言特性进行介绍。

## 性能

首先作为一种编译型语言，可以直接将编译产物作为二进制可执行文件部署，无需随程序一起分发解释器和大量的依赖项，因此相对于 Python、Ruby 以及 Javascript 等解释型语言，会效率更高。

同时，Rust 提供了大量的零成本抽象（如泛型、async/await、迭代和闭包等），在保证开发效率的同时避免了运行时开销。

一般来说，Rust 的性能和 C/C++ 相似，无虚拟机，无 GC，运行时仅依赖 libc，**在需要高性能场景中使用已经足够**。

## 可靠性

在一个 c++ 项目中，我们经常会遇到各类 crash，处理这些 crash 通常会花费大量的人力。

而在一个 Rust 项目中，如果规范使用基本上在编译阶段就可以避免几乎所有的 crash，也就是说，**使用 Rust 的项目只要编译通过，就只有逻辑错误，不会再有 crash**（自身使用了 unsafe 除外）。

Rust 之所以能做到这一点，得益于其设计的所有权、生命周期、Option 机制以及智能指针等，这一点我们在下文的语言特性中也会更详细地分开介绍。

> 实际上，笔者现在负责的项目中 Rust 和 C++ 大约各有一半的代码，在这其中 Rust 几乎没有出现过 crash，而 c++ 基本上每双周（一个迭代）都会新增一些种类的 crash。

## 生产力

### 代码开发效率

得益于 Rust 的大量零成本抽象，以及 Rust 提供的高度灵活的宏机制，我们的代码开发效率还是比较高的。就笔者的体验而言，使用 Rust 完成功能的开发效率略低于 Typescript，但是远高于 c++（和 Typescript 相比，Rust 通常会需要花费额外的一些时间来解决编译问题，但是换来的是高性能和稳定性，我认为这是值得的）。

另外随着 Rust 语言的逐渐成熟，配套的 IDE 和编辑器（Clion、VSCode）也逐渐成熟，日常代码开发提示、以及代码调试等都非常方便。

### 包管理系统

另外值得一提的是，Rust 的包管理系统非常强大，这一点我认为 Rust 也参考了 npm，包管理系统的使用体验也和 npm 比较接近，新增一个依赖，只需要在 Cargo.toml 配置文件中新增一行配置即可。方便的包管理系统，让我们可以方便地复用[社区各类优秀的资源](https://crates.io/)。

相对来说，c++ 这类老牌语言的包分发和管理就会麻烦很多，甚至在一个项目内也会比较麻烦。

### 现有资源的复用

这一点主要是 Rust 和 C/C++ 的互相调用，Rust 支持调用 C 接口和封装成 C 接口被其他语言所调用，因此对于现有的项目，如果可以提供一层 C 接口的封装，就会比较方便地被 Rust 直接调用。

## 面向前端友好

我认为 Rust 面向前端友好主要体现在两个方面：

### 虽然学习路线陡峭，但和 Typescript 相近点较多

很多人劝退 Rust 理由之一是其学习曲线陡峭，但是实际上前端同学学习 Rust 会比学习 C++ 容易的多，一方面，Rust 的很多机制（async/await、类的设计、包管理）等都和 Typescript 有相似之处，另外一方面写 Rust 只要编译能够通过基本上能够保证你代码质量的下限，也就是说基本上可以上线生产环境。而 C++ 新手写出来的代码通常会有各种 crash 隐患，而且排查通常较为困难，导致容易背锅，这一点来说对新手非常不友好。

### 面向 WASM 友好

对于 WebAssembly，笔者并不推荐 AssemblyScript，因为其虽然是 “Typescript”，但是使用起来和 Typescript 相差太多，而且无法完全直接使用 JavaScript 的第三方库，调试等也并不是很方便。

而剩余的几类语言中（Rust、C++、Kotin、Golang），相对来说 Rust 和 C++ 的 wasm 编译都较为成熟，Rust 更是在设计之初即考虑支持 WebAssembly 并且将其作为一个主要亮点，被 Rust 官方团队直接维护（包含了勤劳的 Alex Crichton，其也是 tokio 的作者）。因此我们使用 Rust 编译 WebAssembly 非常方便，并且可以直接使用大多数第三方 Rust 库，使用体验和 Rust Native 开发基本上没有差异。

目前笔者的项目中，有一部分模块即使用了 Rust + WebAssembly，同时支持了 Windows/Mac/iOS/Android/Web 五种平台，并且几乎都做到了最高性能。

> 当然不得不承认，Rust 编译 WebAssembly 在使用到 C++ 的资源时也并不是十分方便，Rust 的 WASM 编译器和 Emscripten 也有诸多差异，适配起来会比较头疼，如果现有项目主要是 C++，还是建议直接使用 Emscripten。

## 重点语言特性

Rust 拥有大多现代语言具备的特性，比如 RAII、动态数据类型等，另外还有不少设计是 Rust 中独有的，下面我们对一些 Rust 中比较独特的语言特性进行一些介绍。

### 所有权

Rust 的所有权机制，即一个值同一个时刻只能被一个变量所引用，我们来看一个简单的例子：

```
let a = "hello".to_string();
let b = a;
println!("a: {:?}", a); // 提示报错：value borrowed here after move
```

因为我们把 a 对应的数据的所有权给到了 b，也就是说 a 不再拥有对应的数据的所有权，因此也无法访问，**这种机制保证了数据安全，能够有效避免悬垂指针的发生**。

当然，对于实现了 Copy（一般来说，都是存储在栈上面的简单数据类型），在赋值阶段会自动拷贝，或者对于没有实现 Copy，但是实现了 Clone（需要主动调用）的类型我们显式调用 Clone，都可以编译通过，这些设计给我们的日常开发中带来了极大的便利：

```
let a: i32 = 1024;
let b = a;
println!("a: {:?}", a); // pass

let a = "hello".to_string();
let b = a.clone();
println!("a: {:?}", a); // pass
```
### 智能指针

在 Rust 中，一般情况下并没有空指针的概念，并不像 c++ 有 nullptr、java 有 null，Rust 中如果表示一个可空的内容只能使用 Option（有点类似 C++ 的 std::optional）。

除了 Option，Rust 还封装了若干种高级指针，并对不同类型的指针的行为进行限制，以提高其安全性：

* Box：用于在堆上存储数据，**单一所有权**（即一般情况下不会存在一个指针乱飞的情况），可以用于封装在编译时未知大小的类型。
* Rc：引用计数指针，不支持多线程
* Arc：多线程版本的引用计数指针
* RefCell：保持内部可变性的指针，即我们如果希望多个所有者共同拥有并且都可以修改的指针，需要结合 Rc 或 Arc 加 RefCell 一起使用。‘

Rust 还提供了一些其他类型的智能指针，在这里不再过多介绍，虽然这里的大部分概念 c++ 也存在，但是 Rust 基本只能是强制你使用这些内容，而无法使用不安全的裸指针。

### 多态

Rust 中的多态有基于泛型的静态派发和基于 trait 的动态派发。

* 静态派发：是一种零成本抽象，在 C++ 中也有类似的概念，静态派发是通过对不同类型的调用在编译期间生成不同版本的代码来实现的，不会引入运行时开销（但请注意可能会造成代码体积膨胀）。
* 动态派发：有些场景下，我们没有办法在编译期间确定变量的实际类型，进而无法确定其占用内存大小，Rust 也提供了 trait 机制来实现动态派发，同时 Rust 将此类 trait 使用 dyn 进行显式指定：

```
// 动态派发：
trait Speak {
    fn hello(&self);
}
struct Human;
impl Speak for Human {
    fn hello(&self) {
        println!("hello, I am a Human");
    }
}
fn test_hello(someone: Box<dyn Speak>) {
    someone.hello();
}
fn main() {
    let human = Human {};
    test_hello(Box::new(human));
}
```

### 宏

Rust 中的宏的能力非常强大，其不同于 C/C++ 中的宏简单地按字符串替换代码，而是基于语法树进行操作，在编译阶段被展开成源代码进行嵌入。

具体 Rust 中的宏也分为声明宏和过程宏，能够实现的需求非常多样，在一个大型项目中，我们可以通过宏的使用解放生产力，并且使代码更清晰。不过，宏这一部分的具体学习相对比较复杂，在这里便不再进行举例。

---

综合来说，Rust 作为一个比较先进的语言，没有太多的历史包袱，从各个语言中吸收了不少的优质特性，比较适合我们在新项目的技术选型中作为一个考虑因素。

## 如何开始学习 rust

### 起步

rust 本身的文档和学习资料官方提供的比较全面，一个必读的内容是[“The Rust Programming Language”](https://doc.rust-lang.org/book/)。

不过，rust 的官方文档读起来可能略有枯燥，这个时候我建议可以先开始读*张汉东*的《Rust 编程之道》，相对来说更加深入浅出，不过还是后续还是建议读一遍文档。

### 项目应用

在我们将上述内容读完之后（如果每天两个小时的话，大约需要一个月的时间），具备了一定的 Rust 语言基础，可能需要思考下如何在现有项目中落地，我个人的一个建议是：

* 如果现有项目是偏 web 的，可以先考虑通过 wasm 来落地，相对来说上手成本很低，我之前也对[入门 Rust 开发 WebAssembly](https://zhuanlan.zhihu.com/p/104299612)有所总结。
* 如果现有项目是偏 native 的，可以考虑将部分新模块、或者 crash 告警比较多的逻辑部分，使用 rust 实现并且通过 C FFI 和现有模块进行交互，渐进式引入 Rust 技术栈。

