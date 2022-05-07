---
title: 以SWC为例分析Javascript转译器的工作原理
date: 2022-04-17 15:54:38
tags:
    - javascript
---

> swc 是一个使用 Rust 开发的 Javascript 转译器，其可以用于将 Typescript、JSX 以及目前浏览器尚未支持的写法转译成兼容性较高的 JavaScript 代码，这部分之前比较典型的项目有 babel。实际上，现在 swc 已经在开发或者在计划中的功能还包括 Typescript 检查、代码打包、代码压缩等更多的功能，有望在将来直接替换 webpack 等构建工具。
> 和 babel 相比，只因 swc 采用了 Rust 开发，就已经快很多倍了，但其相近的竞争对手还有 ESBuild（使用 golang 开发），在某些场景下是比 swc 更快的。不过 swc 的核心开发者（已入职 vercel ）也在持续不断地在优化性能，笔者希望通过一些文章，来分析 swc 的原理和性能优化。

本文试图尝试回答以下问题：

1. swc 转译 JavaScript 的原理
2. swc 插件系统如何工作，如何编写一个 swc 插件


## 整体流程

swc 转译 JavaScript，主要分为三个部分 ：

1. parser：词法分析和语法分析，这个也就是一般编译器的前两个步骤的工作，输入为用户的代码文本，输出为语法树对象。
2. transformer：转换器部分，这部分可以把高级的 Typescript/Javascript/JSX 语法转换成目标版本兼容的语法，同时也支持用户自己写转换插件。
3. emitter：发射器，遍历转换之后的语法树，生成实际的代码。

## parser 

`parser` 的主要部分在 `crates/swc_ecma_parser` 里面，大概的流程是构造一个 Lexer 和 Parser，然后分析转换。

这部分代码虽然比较多，但是结构还是比较明晰的，主要就是逐个 statement 进行解析，对应的不同类别的 statement 解析都在 parser 这个目录下分到了不同的文件中。

最后解析完成，我们能够得到如下结构体的一个模块：

```rust
#[ast_node("Module")]
#[derive(Eq, Hash, EqIgnoreSpan)]
pub struct Module {
    pub span: Span,

    pub body: Vec<ModuleItem>,

    #[serde(default, rename = "interpreter")]
    #[cfg_attr(feature = "rkyv", with(crate::EncodeJsWord))]
    pub shebang: Option<JsWord>,
}
```

## transformer

在上一部分，我们已经获取到了一个 `Module`，这个 `Module` 包含了我们解析好的语法树信息。

接下来这部分是转换代码到我们的目标版本。

这里有两个变量：

* pass：pass 是多个转换器的集合，它们通过 `chain!` 相连接，可以负责转换代码。

```rust
let pass = chain!(config.pass, custom_after_pass(&config.program, &comments));

// chain! 的实现非常简单：
#[macro_export]
macro_rules! chain {
    ($a:expr, $b:expr) => {{
        use $crate::AndThen;
        AndThen {
            first: $a,
            second: $b,
        }
    }};
    // ... 省略一些内容
    ($a:expr, $b:expr,  $($rest:tt)+) => {{
        use $crate::AndThen;
        AndThen{
            first: $a,
            second: chain!($b, $($rest)*),
        }
    }};
}
```

* fold_with：实际的作用是，根据传入的 pass，处理当前的语法树

```rust

let program = helpers::HELPERS.set(&Helpers::new(config.external_helpers), || {
      HANDLER.set(handler, || {
          // Fold module
          program.fold_with(&mut pass)
      })
});
```

这里的 pass，实际上已经通过 `chain!` 宏变成了 `AndThen` 的实例，而 `AndThen` 的实例通过以下代码，可以保证会依次被调用到：

```rust
impl<A, B> VisitMut for AndThen<A, B>
where
    A: VisitMut,
    B: VisitMut,
{
    fn visit_mut_module(&mut self, n: &mut Module) {
        self.first.visit_mut_module(n);
        self.second.visit_mut_module(n)
    }
    fn visit_mut_script(&mut self, n: &mut Script) {
        self.first.visit_mut_script(n);
        self.second.visit_mut_script(n)
    }
}
```

其中的每一个 `transform` 实例，都是实现了一定功能的转换器，实现的方式就是把对应的方法的默认方法覆盖即可。

### transform 插件

swc 支持我们自定义 transform 插件，来实现自定义的转换，不过**swc 的插件是需要使用 Rust 开发的，编译产物为 WebAssembly 模块。**

> 当前版本的 swc 也支持我们使用 js 开发插件，不过由于这种方式消耗较大，之后会被放弃。

TODO: 补充一个自定义插件的实现

## emitter



TODO: 补充 emitter 的代码，可能需要结合 Typescript 一起看