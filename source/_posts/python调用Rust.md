---
title: 使用 Rust 开发 python 模块
date: 2022-06-04 17:34:05
tags:
    - Rust
---

关于 Rust 的基本介绍，我在之前的[文章](http://niexiaotao.cn/2021/09/02/%E4%BA%86%E8%A7%A3%20StackOverFlow%20%E4%B8%8A%E9%9D%A2%E6%9C%80%E5%8F%97%E6%AC%A2%E8%BF%8E%E7%9A%84%E8%AF%AD%E8%A8%80%20Rust/)有做过一些总结。

本篇文章我们关注如何在 python 中调用 Rust 开发的模块。

## Rust FFI 的一般思路

Rust 可以编译出兼容 C ABI 的动态库或者静态库，Rust 调用其他语言，以及 Rust 被其他语言调用，基本都是通过 C ABI 来进行 FFI 调用。

所以我们可以看出，实际上 C++ 调用 Rust *并不是特别方便*，需要使用 Rust 提供的 C 接口，也因此没有办法使用 C++ 提供的类型，而 Rust 在导出接口的时候，也没有办法使用 Rust 的类型系统，需要转换成 C 类型。

大多数时候我们都会在这种场景下写一层 wrapper 和 converter，用来自动生成 FFI 层的一些胶水代码。

对于 Python 这类高级语言调用 Rust，基本也是类似的思路，我们可以简单总结为下图：

![](/img/calling_rust_from_python_std_ffi_and_ctypes.png)

值得庆幸的是，对于 Python 调用 Rust，社区已经有非常多现成的成熟工具可以使用，基于这些工具，我们可以比较方便地专注于 Rust 实现逻辑本身，无需关注太多 FFI 和转换细节。

## 入门

一个比较方便的方法是使用 [PyO3](https://github.com/PyO3/pyo3)，PyO3 不仅仅提供了 rust binding，也提供了创建 python 包的开箱即用的脚手架工具 [maturin](https://github.com/PyO3/maturin)，使用 maturin 我们可以很方便地创建一个基于 rust 开发的 python 扩展模块。

我们这里整理一下官方文档中提供的最简单的方式，读者可以直接依次执行下面的 shell 脚本：

```shell
$ mkdir string_sum
$ cd string_sum
# 创建 venv 的这一步不能省略，否则后续运行的时候会报错
$ python -m venv .env
$ source .env/bin/activate
$ pip install maturin
# 直接使用 maturin 初始化项目即可，选择 pyo3，或者直接执行 maturin init --bindings pyo3
$ maturin init
✔ 🤷 What kind of bindings to use? · pyo3
  ✨ Done! New project created string_sum
```

这个时候，我们可以得到一个简单的 Rust 项目，并且包含了一个示例调用，我们无需修改任何代码，可以直接执行下面的命令测试：

```shell
# maturin develop 会自动打包出一个 wheel 包，并且安装到当前的 venv 中 
$ maturin develop
$ python
>>> import string_sum
>>> string_sum.sum_as_string(5, 20)
'25'
```

## 进阶工具

接下来，我们介绍几个方便我们使用 Rust 开发 python 包的进阶工具或引导。

### setuptools-rust

[setuptools-rust](https://github.com/PyO3/setuptools-rust) 是一个 setuptools 的插件，让我们可以比较方便地编写使用 pyo3 开发的 rust python 包。

我们可以 clone 它的源代码，直接使用它提供的示例，参考如下命令测试：

```shell
$ cd examples/rust_with_cffi
$ python ./setup.py develop
$ python
Python 3.9.7 (default, Sep  3 2021, 12:37:55)
[Clang 12.0.5 (clang-1205.0.22.9)] on darwin
Type "help", "copyright", "credits" or "license" for more information.
>>> from rust_with_cffi import rust
>>> rust.rust_func()
14
```

### dict-derive

这个 rust 库提供了 FromPyObject 和 IntoPyObject 两个宏，使用这两个宏，我们可以很方便地进行 python dict 结构和 Rust 结构体的转换。

例如我们声明这样一个结构体：

```rust
#[derive(FromPyObject, IntoPyObject)]
pub struct User {
    pub name: String,
    pub email: String,
    pub age: u32,
}
```

我们就直接可以在导出函数中这样使用了：

```rust
// Requires FromPyObject to receive a struct as an argument
#[pyfunction]
fn get_contact_info(user: User) -> PyResult<String> {
    Ok(format!("{} - {}", user.name, user.email))
}

// Requires IntoPyObject to return a struct
#[pyfunction]
fn get_default_user() -> PyResult<User> {
    Ok(User {
        name: "Default".to_owned(),
        email: "default@user.com".to_owned(),
        age: 27,
    })
}
```

我们通过宏展开可以发现，这两个宏所做的事情就是分别将 `pyo3::types::PyDict` 转换成 Rust 结构体和将 Rust 结构体转换成 `pyo3::types::PyDict`。

整体宏展开的代码不多，还是比较方便阅读的。

### rust-numpy

[rust-numpy](https://github.com/PyO3/rust-numpy) 是一个 rust 版本的 numpy C ABI 封装，使用这个库我们可以在 Rust 中调用 numpy

接下来我们运行该库的示例代码。

我们需要先安装 nox，nox 是一个 python 自动化任务工具。

```shell
$ python3 -m pip install nox
```

之后我们进入到命令行直接执行即可：
```
$ cd examples/simple
$ nox
```

顺利的情况下，我们可以看到它会输出测试成功：
```
tests/test_ext.py .....                                                                                                                                       [100%]

========================================================================= 5 passed in 0.32s =========================================================================
nox > Session tests was successful.
```

### pandas

在 Rust 中并没有直接和 python 中的 pandas 包对标的诸如 pandas-rs 包。

不过 Rust 标准库本身也提供了非常多的数据处理函数如筛选、过滤等，我们可以自己手写代码完成大部分 pandas 的工作。

在[这篇文章](https://able.bio/haixuanTao/data-manipulation-pandas-vs-rust--1d70e7fc)中，作者使用了大约 160,000行/ 130列，总大小为 150Mb 的数据， 分别使用 Rust 和 Pandas 处理并测试，我们可以看到提升还是比较显著的：

| | Time(s) | Mem Usage(Gb) | 
| ----- | ----- | ----- |
|Pandas | 3.0s | 2.5Gb |
| Rust | 1.6s 🔥 -50% | 1.7Gb 🔥 -32% |


## 其他

pyo3 的 README 里面还列举了一些其他的工具库，使用起来相对比较简单，这里就不做单独介绍了。

* [pyo3-log](https://github.com/vorner/pyo3-log)：在 Rust 中使用 python 的 logging 库。
* [pyo3-built](https://github.com/PyO3/pyo3-built)：可以在编译 rust 的 python 模块的时候写入一些构建信息，如 rust 版本等。
* [pyo3-asyncio](https://github.com/awestlake87/pyo3-asyncio)：python asyio 的 Rust binding，可以将 python 的 async 转换成 Rust 的 features。
* [rustimport](https://github.com/mityax/rustimport)：可以在 python 中直接引入 rust 代码，但因为引入的时候需要编译，笔者不是很建议在生产环境中直接使用。


---

虽然上面介绍了这么多工具，但是笔者认为，在实际使用中，还是远远不够的，我们应该还会结合业务，寻找和造出更多轮子，这部分工作就有待我们进一步开拓了。