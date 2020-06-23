---
title: 'Rust初探:实现二叉树的增删与遍历'
date: 2019-09-07 23:57:28
tags:
    - Rust
---

### Rust 简介

实际上自己接触 Rust 的时间还是很有限的，这里也不会对 Rust 进行长篇大论地介绍，简单来说，Rust 是一个性能和 c++ 相近的系统级编程语言，同时，由于其所有权与变量生命周期等机制的设计，使其相对于 c++ 来说拥有内存安全的优势，几乎不会出现诸如悬垂指针、数组越界、段错误等问题，在微软、百度、字节跳动等公司均有所使用。

关于 Rust 的特性以及未来，知乎[这个问题中的一些高赞回答以及相关的评论](https://www.zhihu.com/question/30407715)，非常值得一看。

本文会以二叉树这样一个具体的例子出发，来对 Rust 的一部分知识内容进行学习。

### 实现二叉树数据结构

#### 定义结构

之前在 Javascript 等语言中，我们只要对对象有所了解，实现一个二叉树的数据结构是非常简单的事情，而在 Rust 中，可能对于新手来说仅仅是实现基本的数据结构就是一个比较脑壳疼的事情。

我们一般会写出类似这样的代码：

```
struct Tree {
    value: i32,
    left: Tree, // 直接使用 Tree 是不行的
    right: Tree  
}
```

自然不会通过 Rust 的编译检查，会报错例如：`recursive type has infinite size`，不过其同时给我们提供了解决方案，这里我们使用 `Box<T>` 指针。

另外，考虑到二叉树的左右子树可能为空，所以这里我们还需要增加一个 `Option`。

最终我们的二叉树数据结构定义如下：

```
#[derive(Debug, Default)]
struct Tree {
    value: i32,
    left: Option<Box<Tree>>,
    right: Option<Box<Tree>>   
}
```

#### 实现基本的方法

这里我们实现一些二叉树的基本的方法，作为上述结构体的方法，我们将实现以下方法：

* 获取二叉树节点的值（其实也可以没有这个方法）。
* 修改二叉树节点的值。
* 设置子树。
* 删除子树。

这里除了第一个，其余我们都需要传递 `self` 的可变引用，我们的实现如下：

```
impl Tree {
    fn get_val(&self) -> i32 {
        return self.value;
    }
    fn set_val(&mut self, val: i32) -> i32 {
        self.value = val;
        return self.value;
    }
    fn insert(&mut self, dir: &String, val: Tree) {
        assert!(dir == "left" || dir == "right");
        match dir.as_ref() {
            "left" => self.left = Some(Box::new(val)),
            "right" => self.right = Some(Box::new(val)),
            _ => { 
                println!("Insert Error: only left and right supported");
                process::exit(1);
            }
        }
    }
    fn delete(&mut self, dir: &String) {
        assert!(dir == "left" || dir == "right");
        match dir.as_ref() {
                "left" => self.left = None,
                "right" => self.right = None,
                 _ => { 
                    println!("Insert Error: only left and right supported");
                    process::exit(1);
                }
        }
    }
}
```

### 遍历二叉树

这里遍历二叉树我们作为一个单独的方法，而不是属性方法来实现，这样会更符合我们平时的业务场景，这里其实问题比较多的，我们先简易实现一个版本：

```
fn traverse(tree: Tree) {
    println!("Node Value: {:?}", tree.value);
    if tree.left.is_some() {
        traverse(*tree.left.unwrap()); // 手动解引用
    }
    if tree.right.is_some() {
        traverse(*tree.right.unwrap()); // 手动解引用
    }
}
```

如果我们测试一下这个版本，发现的确能够正常遍历的，但是实际上这有一个致命的问题：

这里采用的是所有权的移动，而不是不可变借用，这会导致我们的函数执行完后原来变量的所有权已经被移动了，换一种说法则是会消耗掉这个变量，这显然不是我们预期的。

虽然我们也可以在函数中返回 tree 的方式来最后再次移动所有权，但这样非常不便于实现，经过重构，我们采用了如下的方式实现：

```
fn traverse(tree: &Tree) {
    println!("Node Value: {:?}", tree.value);
    match tree.left {
        Some(ref x) => traverse(x),
        _ => {}
    }
    match tree.right {
        Some(ref x) => traverse(x),
        _ => {}
    }
}
```

>另外一个注意点则是由于 `unwrap()` 本身是一个消耗性操作，我们这里不能使用 `unwrap`，参考[stackOverflow的提问1](https://stackoverflow.com/questions/22282117/how-do-i-borrow-a-reference-to-what-is-inside-an-optiont)、[stackOverflow的提问2](https://stackoverflow.com/questions/32338659/cannot-move-out-of-borrowed-content-when-unwrapping)。

我们最终的完整代码如下：

```
use::std::process;
use std::borrow::Borrow;
#[derive(Debug, Default)]
struct Tree {
    value: i32,
    left: Option<Box<Tree>>,
    right: Option<Box<Tree>>   
}

impl Tree {
    fn get_val(&self) -> i32 {
        return self.value;
    }
    fn set_val(&mut self, val: i32) -> i32 {
        self.value = val;
        return self.value;
    }
    fn insert(&mut self, dir: &String, val: Tree) {
        assert!(dir == "left" || dir == "right");
        match dir.as_ref() {
            "left" => self.left = Some(Box::new(val)),
            "right" => self.right = Some(Box::new(val)),
            _ => { 
                println!("Insert Error: only left and right supported");
                process::exit(1);
            }
        }
    }
    fn delete(&mut self, dir: &String) {
        assert!(dir == "left" || dir == "right");
        match dir.as_ref() {
                "left" => self.left = None,
                "right" => self.right = None,
                 _ => { 
                    println!("Insert Error: only left and right supported");
                    process::exit(1);
                }
        }
    }
}

// 原始的非消耗性遍历:
// fn traverse(tree: &Tree) {
//     println!("Node Value: {:?}", tree.value);
//     if tree.left.is_some() {
//         // cannot move out of borrowed content
//         // 首先 unwrap 是一个消耗性操作
//         // 这是由于 unwrap 函数造成?  as_ref 也不行
//         traverse((tree.left.as_ref().map(|x| **x).unwrap()).borrow());
//     }
//     // if tree.right.is_some() {
//     //     // cannot move out of borrowed content
//     //     traverse(tree.right.unwrap().borrow());
//     // }
// }

// 非消耗性遍历
fn traverse(tree: &Tree) {
    println!("Node Value: {:?}", tree.value);
    match tree.left {
        Some(ref x) => traverse(x),
        _ => {}
    }
    match tree.right {
        Some(ref x) => traverse(x),
        _ => {}
    }
}

// 消耗性遍历：
// fn traverse(tree: Tree) {
//     println!("Node Value: {:?}", tree.value);
//     if tree.left.is_some() {
//         traverse(*tree.left.unwrap()); // 手动解引用
//     }
//     if tree.right.is_some() {
//         traverse(*tree.right.unwrap()); // 手动解引用
//     }
// }

fn main() {
    println!("begin rust tree test:");
    let mut tree = Tree { value : 12, ..Default::default() };
    let mut left = Tree { value : 121, ..Default::default() };
    tree.insert(&String::from("left"), left);
    let mut right = Tree { value : 122, ..Default::default() };
    tree.insert(&String::from("right"), right);
    // tree.delete(&String::from("right"));
    // println!("Tree val: {:?}", left.get_val()); 不能这样写，所有权已经被移动
    traverse(&tree);
    // traverse(tree);
}
```