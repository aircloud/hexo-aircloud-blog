---
title: 使用 Rust WebAssembly 0拷贝进行计算加速
date: 2020-06-26 14:47:03
tags:
    - Rust
---

demo: https://github.com/aircloud/rust-wasm-demo  

其他资料：[入门 Rust 开发 WebAssembly](https://zhuanlan.zhihu.com/p/104299612)

一般来说，使用 WebAssembly 能够在一定程度上提高性能，不过有的时候我们也许会发现，使用 WebAssembly 之后，有的时候我们不仅发现性能没有提升，反而下降了许多甚至数倍，实际上这是因为，使用 WebAssembly 需要非常谨慎，有很多细节都会大幅度影响性能，比如：

* 我们编译采用的是 debug 还是 release 方式。
* 最后编译的结果是否采用了不同级别的优化，如果使用了 `opt-level = 's'` 那么通常速度也会下降很多。
* 是否在 JS 和 rust 之间存在大量的数据拷贝，因为很多代码是工具链生成的，也许有的时候我们会忽视这一点。

本文针对以上等一些问题特别是第三点，给出一个 wasm 优化的参考方案，并给出示例代码。

### 编译优化

我们在优化数据拷贝之前，对于编译我们可以做一些前置的简单的工作。

* 检查 Cargo.toml 脚本中 `[profile.release]` 中的 `opt-level` 选项，确认我们所使用的值：

```
This flag controls the optimization level.

0: no optimizations, also turns on cfg(debug_assertions) (the default).
1: basic optimizations.
2: some optimizations.
3: all optimizations.
s: optimize for binary size.
z: optimize for binary size, but also turn off loop vectorization.
Note: The -O flag is an alias for -C opt-level=2.

The default is 0.
```

如果我们使用了 ‘s’ 或者 'z'，那么通常会牺牲一部分性能（对于 demo 而言，使用 'z'， wasm 的性能也只有 js 的 20%），因为其主要是对体积进行一定的优化，所以如果优化前的体积我们可以接受的话，通常不需要这样的优化。

在以上的前提下，我们使用 `--release` 的方式编译，通常就可以了。

### 减少拷贝

在这之前，我们需要有一个认知：

**通过 rust 工具链编译的 wasm 代码，所有参数传入都是需要拷贝一次的，包括我们传入 ArrayBuffer 等 Buffer 类型的参数。**这是由于 wasm 只能访问自己的线性内存，而这个拷贝，通常是我们在处理大规模计算的一个坎，有的时候虽然 wasm 计算快一点，但是拷贝的消耗还是比较大的，加之 js 有若干 v8 优化的加持，可能和 wasm 也相差不多。

所以我们要把计算移植到 wasm 中的话，首先要解决的就是大规模数据拷贝的问题。

这里的一般思路为：

1. wasm 分配内存：调用 wasm 的方法，在 wasm 内存中分配空间，返回指针位置
2. js 写入数据：js 端在 wasm 的 memory arraybuffer 上，按指针位置和数据量建立 view，把数据写入
3. wasm 计算：调用 wasm 方法完成计算， 返回计算好的批量结果的指针位置和大小
4. js 读取数据：js 端在 wasm 的 memory arraybuffer上，按指针位置和数据量建立 view，把数据读出

接下来，我们通过一个 demo 来完成以上几点，demo 的主要功能为：

* 初始化一个 ImageData，内容随机。
* 分别使用 js 和 WebAssembly 进行高斯模糊计算，并计算二者的时间，进行对比。

这里的 demo 只是辅助进行验证改方案的可行性并且给出一个示例，并不作为一个标准的 benchmark 去对比 js 和 WebAssembly 的性能，同时，也并没有 UI 展示，计算结果输出在控制台中。

最终笔者运行的结果为，js 比 WebAssembly 慢 30% 左右。

#### 1. wasm 分配内存

这部分的通用做法，即我们在 wasm 的 rust 中分配一个数组（Vec），然后把其指针传递给 js：

```
// rust：
#[wasm_bindgen]
pub fn new_buffer(key: String, len: usize) -> *const u8 {
  // GlobalBufferStorage 是一个 lazy_static
  let mut global_buffer_storage = GlobalBufferStorage.lock().unwrap();
  let mut buffer = vec![255; len];
  let ptr = buffer.as_ptr();
  global_buffer_storage.buffer_map.insert(key, buffer);
  ptr
}
```

为了后续方便寻找到这段数据，我们可以使用一个 key 将这个 Vec 联系起来，并且在 Rust 中放入全局（可以使用 lazy_static!，因为这种类型的数据没有办法直接定义在全局），之后通过 key 来查找数据。

在 js 中，我们就可以建立各种 TypedArray 对其进行操作：

```
const ptr = this.wasm!.new_buffer(key, len);
const u8Arr = new Uint8ClampedArray(this.wasm!.get_wasm_buffer(), ptr, len);
```

**这个时候，我们在 js 或 rust 任何一侧改了这个数据之后，都可以在另外一侧访问到。**

实际上，在 js 侧的比如 [ImageData](https://developer.mozilla.org/en-US/docs/Web/API/ImageData/ImageData) 等一些对象中，也支持我们传递一个 TypedArray 进行初始化，这让我们在比如 canvas 等应用场景下，使用 wasm 分配的内存更为方便。

```
const imageData = new ImageData(u8Arr, width, height);
```

#### 2. js 写入数据

如果我们需要在 js 侧写入数据，实际上这个时候我们得到的 TypedArray 已经和直接使用 js new 的 TypedArray 在使用上没有差别，可以正常按照数组的方式进行数据写入。

不过，这里需要注意的是，js 写入通过 wasm 分配内存建立的 TypedArray，有些场景下在一定程度上速度要慢于直接使用 js new 的 TypedArray（不过在笔者的测试数据中，wasm 分配的方式反而是更快的），所以如果我们是一个高频的数据写入的场景，比如帧数据等，这个时候最好进行一次对比测试。


#### 3. wasm 计算

当我们真正需要进行计算的时候，我们可以调用 wasm 的计算函数，并且传入上文中定义的 key，这样 wasm 的 rust 函数可以直接找到这段数据，这里我们的 demo 为一段计算卷积的函数：

```
#[wasm_bindgen]
pub fn convolution(key: String, width: usize, height: usize, kernel: Vec<i32>) {
  let mut global_buffer_storage = GlobalBufferStorage.lock().unwrap();
  let kernel_length = kernel.iter().sum::<i32>() as i32;
  if let Some(buffer) = global_buffer_storage.buffer_map.get_mut(&key) {
    for i in 1..width-1 {
      for j in 1..height-1 {
        let mut newR: i32 = 0;
        let mut newG: i32 = 0;
        let mut newB: i32 = 0;
        for x in 0..3 { // 取前后左右共9个格子
          for y in 0..3 {
            newR += buffer[width * (j + y - 1) * 4 + (i + x - 1) * 4 + 0] as i32 * kernel[y * 3 + x] / kernel_length;
            newG += buffer[width * (j + y - 1) * 4 + (i + x - 1) * 4 + 1] as i32 * kernel[y * 3 + x] / kernel_length;
            newB += buffer[width * (j + y - 1) * 4 + (i + x - 1) * 4 + 2] as i32 * kernel[y * 3 + x] / kernel_length;
          }
        }
        buffer[width * j * 4 + i * 4 + 0] = newR as u8;
        buffer[width * j * 4 + i * 4 + 1] = newG as u8;
        buffer[width * j * 4 + i * 4 + 2] = newB as u8;
      }
    }
  } else {
    return ();
  }
}
```

因为这段函数对应操作的内存数据实际上已经在 wasm 和 js 之间共享了，所以也是不需要返回值的，等计算完成后 js 直接去读之前建立的 TypedArray，甚至直接使用通过 TypedArray 创建的 ImageData，进行绘制上屏等后续操作。

#### 4. js 读取数据

在 demo 中，我们可以直接通过 `CanvasRenderingContext2D.putImageData()` 传入之前获取的 imageData，绘制上屏。

### 其他方案

实际上，我们如果目的是加速 js 计算，不仅仅有 WebAssembly 这一个方案可以选择，如果我们的环境中拥有可以访问 Node 的能力或者可以访问原生模块的能力（比如，我们的应用运行在 electron 环境，或者是一些移动客户端），也可以采用比如 addon 的方式来运行我们的计算部分，相比于 wasm，这部分的优缺点在于：

优点：

* 通常可以更好的控制优化，甚至做到汇编级别的优化定制，性能提升空间更高（同样也可能会面临数据拷贝的问题，也需要一定方式减少拷贝）。
* 在重 addon 的环境下（例如，其他大量功能也依赖 addon），可以更好的处理函数调用关系、依赖库使用等，一定程度上减少体积和增加开发的便捷性，而 wasm 会被编译成一个独立的二进制文件，处于沙盒环境中，无法直接调用其他的动态库。

缺点：

* 无法做到像 wasm 一样跨平台，并且可以同时运行在网页、桌面环境、移动端等任何 Webview 存在的环境中。

不过总之，如果使用得当，二者的性能都是可以优于原生的 js，都可以作为优化方案考虑。
