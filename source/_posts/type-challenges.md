---
title: 读完 type-challenges，我总结了如下常用的内容
date: 2022-07-30 15:39:38
tags:
  - Typescript
---

近期用了一定时间通读了 [type-challenges](https://github.com/type-challenges/type-challenges)，感慨 Typescript 也可以写的如此复杂之余，也出于以下几个原因，对题目做了整理，形成本文：

* type-challenges 整体题目较多，而且很多时候我们只是学习为主，并不是以“做题”为目的，这样我们在列表-问题-答案列表-答案详情中跳来跳去很麻烦，也很难甄别优质解答。
* type-challenges 仓库中某些题目，实属偏门，可能对于大多数业务开发来说，永远也用不到，投入时间在这部分，ROI 就会非常低。
* type-challenges 有些题目比较类似，但是却放到了不同的地方，甚至难度不同，笔者认为结合起来一起看可能更高效。

因此，本文对笔者认为重要的、有业务使用场景或者可以给我们以很大启发的题目进行罗列和解析，同时规避了一些复杂度很高，但实际大多数人用不到的题目（比如 Typescript 实现 JSON 解析），对于普通的 Typescript 开发者而言，看完本文的题目就基本能够以一个比较小的代价掌握 type-challenges 中贴近日常开发和业务的部分。

> 感谢 type-challenges 仓库的贡献者们

## Typescript 内置工具函数

实际上，Typescript 自身除了定义类型以外，自己内置了很多非常有用的工具函数，这部分是我们日常 Typescript 开发应当必须掌握，信手拈来的，我建议如果对这部分不熟悉的话，先多读几遍这部分。

这部分内容，可以在[这里](https://www.typescriptlang.org/docs/handbook/utility-types.html) 了解，也可以在自己的项目中，点进 `node_modules/typescript/lib/lib.es5.d.ts` 直接了解，注释比较详尽。

> 实际上，type-challenges 的一部分简单和中等的题目就是实现这些内置工具函数的二次实现，这部分内容我在本文基本没有重复罗列，但是建议大家去阅读官方实现，一般来说也都比较短小，容易理解

## type-challenges 部分重点题目

### 数组第一个元素

[原文地址](https://github.com/type-challenges/type-challenges/blob/main/questions/00014-easy-first/README.md)

使用：
```typescript
type arr1 = ['a', 'b', 'c']
type head1 = First<arr1> // expected to be 'a'
```

实现：

```typescript
export type First<T extends any[]> = T extends [infer first, ...any[]]
  ? first
  : never;
```

除了本题目，还有许多其他类似的实现：：

```typescript
// 数组最后一个元素:
export type Last<T extends any[]> = T extends [...any, infer L] ? L : never;

// Pop：
// type arr1 = ['a', 'b', 'c', 'd']
// type re1 = Pop<arr1> // expected to be ['a', 'b', 'c']
export type Pop<T extends any> = T extends [...infer P, infer R] ? P : never;

// Push:
// type Result = Push<[1, 2], '3'> // [1, 2, '3']
export type Push<T extends any[], U> = [...T, U];

// UnShift:
// type Result = UnShift<[1, 2], 0> // [0, 1, 2,]
export type UnShift<T extends any[], U> = [U, ...T];

// Shift:
// type Result = Shift<[3, 2, 1]> // [2, 1]
export type Shift<T> = T extends [infer F,... infer R] ? R : undefined

// Concat：
// type Result = Concat <[1], [2]> // expected to be [1, 2]
export type Concat <T extends any[], V extends any[]> = [...T, ...V];

// Reverse:
// type b = Reverse<['a', 'b', 'c']> // ['c', 'b', 'a']
export type Reverse<T> = T extends [infer Head, ...infer Rest] ? [...Reverse<Rest>, Head] : T

// FilterOut：
// type Filtered = FilterOut<[1, 2, null, 3], null> // [1, 2, 3]
type Filtered = FilterOut<[1, 2, null, 3], null> // [1, 2, 3]
export type FilterOut<T extends any[], F> = 
  T extends [infer First, ...infer Rest] 
    ? First extends F 
      ? FilterOut<Rest, F> 
      : [First, ...FilterOut<Rest, F>] 
    : []
```

技巧提示：

* 通过 extends 加三目运算符，完成条件判断。
* 通过类似 `infer first, ...any[]` 或者 `[infer First, ...infer Rest]` 这种方式来展开数组类型。
* 另外我们还可以使用 `[...T, U]` 来扩充数组的类型定义。

### Tuple to Union

[原文地址](https://github.com/type-challenges/type-challenges/blob/main/questions/00010-medium-tuple-to-union/README.md)

使用：
```typescript
type Arr = ['1', '2', '3']
type Test = TupleToUnion<Arr> // expected to be '1' | '2' | '3'
```

实现比较简单：

```typescript
export type TupleToUnion<T extends any[]> = T[number];
```

技巧提示：

* `T[number]` 这种语法可以比较方便地实现 Tuple to Union


### Deep Readonly

[原文地址](https://github.com/type-challenges/type-challenges/blob/main/questions/00009-medium-deep-readonly/README.md#deep-readonly----)

使用：
```typescript
type X = { 
  x: { 
    a: 1
    b: 'hi'
  }
  y: 'hey'
}

type Expected = { 
  readonly x: { 
    readonly a: 1
    readonly b: 'hi'
  }
  readonly y: 'hey' 
}

type Todo = DeepReadonly<X> // should be same as `Expected`
```

实现：

```typescript
export type DeepReadonly<T> = {
  readonly [P in keyof T]: keyof T[P] extends never ? T[P] : DeepReadonly<T[P]>;
};
```

技巧提示：

* 通过 `keyof T[P] extends never` 这种方式判断是否有子属性，可以完成深度遍历

### Merge 以及其他 interface 相关

[原文地址](https://github.com/type-challenges/type-challenges/blob/main/questions/00599-medium-merge/README.md)

使用：
```typescript
type foo = {
  name: string;
  age: string;
}
type coo = {
  age: number;
  sex: string
}

type Result = Merge<foo,coo>; // expected to be {name: string, age: number, sex: string}
```

Merge 的功能和 typescript 提供的很多内置功能函数很像，这个题目实现的方式很多，我这里给出一个比较简洁的方式：

```typescript
export type Merge<T, U> = {
  [P in Exclude<keyof T, keyof U>]: T[P];
} & {
  [G in keyof U]: U[G];
};
```

类似 Merge，我们在再给出一些其他的简单的类似 type 相关的操作

```typescript
// Diff：选出两个类型中不同属性：
export type Diff<T, U> = Omit<T & U, keyof T & keyof U>;

// PickByType：按照类型选择
// type OnlyBoolean = PickByType<{
//   name: string
//   count: number
//   isReadonly: boolean
//   isEnable: boolean
// }, boolean> // { isReadonly: boolean; isEnable: boolean; }
export type PickByType<T, U> = {
  [K in keyof T as T[K] extends U ? K : never]: T[K];
};

// RequiredByKeys：按照 key 来设置成 Require
// interface User {
//   name?: string
//   age?: number
//   address?: string
// }
// type UserRequiredName = RequiredByKeys<User, 'name'> // { name: string; age?: number; address?: string }
export type RequiredByKeys<T, K = keyof T> = SimpleMerge<
  Partial<Pick<T, Exclude<keyof T, K>>>,
  Required<Pick<T, Extract<keyof T, K>>>
  // **可以看一下 Required 的写法，能学到一点新的东西
>;

// Mutable
// interface Todo {
//   readonly title: string
//   readonly description: string
//   readonly completed: boolean
// }
// type MutableTodo = Mutable<Todo> // { title: string; description: string; completed: boolean; }
// your answers
export type Mutable<T> = {
  -readonly [K in keyof T]: T[K]
}

// Get Required
// type I = GetRequired<{ foo: number, bar?: string }> // expected to be { foo: number }
export type GetRequired<T> = {
  [P in keyof T as T[P] extends Required<T>[P] ? P : never]: T[P]
}
export type RequiredKeys<T> = keyof GetRequired<T>

// Get Optional
// type I = GetOptional<{ foo: number, bar?: string }> // expected to be { bar?: string }
// https://github.com/type-challenges/type-challenges/blob/main/questions/00059-hard-get-optional/README.md
export type GetOptional<T> = {
  [P in keyof T as T[P] extends Required<T>[P] ? never : P]: T[P]
}
export type OptionalKeys<T> = keyof GetOptional<T>
```

一些技巧提示：

* 通过组合 Typescript 内置的功能函数，我们可以完成很多复杂的业务需求。
* 通过 `-readonly` 来减去修饰符。
* 通过 `[P in keyof T]-?: T[P];` 这种方式来减去可选修饰符（Typescript 内置的 Required 就是这么实现的）。
* 通过 `K in keyof T as T[K]` 获取一个属性的类型，结合 extends 做条件判断。

### TrimLeft 以及字符串相关

[原文地址](https://github.com/type-challenges/type-challenges/blob/main/questions/00106-medium-trimleft/README.md)

使用：

```typescript
type trimed = TrimLeft<'  Hello World  '> // expected to be 'Hello World  '
```

实际上字符串操作在 Typescript 中应该用的并不是很多，而这也是 Typescript 比较后面（4.0+）才逐渐完善的功能。
这个题目的解答：

```typescript
type Space = " " | "\n" | "\t";
export type TrimLeft<S extends string> = S extends `${Space}${infer Rest}`
  ? TrimLeft<Rest>
  : S;
```

实际上，我们如果知道可以这样写，还可以实现很多的类似的方式，比如可以很方便地实现 `Trim` 和 `TrimRight` （由于相似度非常高，这两个不再罗列答案），以及 `Replace`、`ReplaceAll`、`DropChar` 等，甚至还可以比较方便地实现类型转换，如 `ParseInt`。

```typescript
// Replace
// type replaced = Replace<'types are fun!', 'fun', 'awesome'> // expected to be 'types are awesome!'
// https://github.com/type-challenges/type-challenges/blob/main/questions/00116-medium-replace/README.md
export type Replace<
  S extends string,
  From extends string,
  To extends string
> = From extends ""
  ? S
  : S extends `${infer Left}${From}${infer Right}`
  ? `${Left}${To}${Right}`
  : S;

// 近似 Replace，替换全部
// type replaced = ReplaceAll<'t y p e s', ' ', ''> // expected to be 'types'
// https://github.com/type-challenges/type-challenges/blob/main/questions/00119-medium-replaceall/README.md
export type ReplaceAll<
  S extends string,
  From extends string,
  To extends string
> = From extends ""
  ? S
  : S extends `${infer Left}${From}${infer Right}`
  ? `${Left}${To}${ReplaceAll<`${Right}`, From, To>}`
  : S;

// Drop Char
// 和上面的比较类似
// type Butterfly = DropChar<' b u t t e r f l y ! ', ' '> // 'butterfly!'
// https://github.com/type-challenges/type-challenges/blob/main/questions/02070-medium-drop-char/README.md
export type DropChar<S, C> = S extends `${infer H}${infer R}`
  ? H extends C
    ? `${DropChar<R, C>}`
    : `${H}${DropChar<R, C>}`
  : "";

// ParseInt：字符串转数字：
export type ParseInt<T extends string> = T extends `${infer Digit extends number}` ? Digit : never
```

另外，基于操作字符串的能力，我们还可以实现更多，比如：

* [StringToUnion](https://github.com/type-challenges/type-challenges/blob/main/questions/00531-medium-string-to-union/README.md) 字符串转联合类型，`123` -> `"1" | "2" | "3"`
* [KebabCase](https://github.com/type-challenges/type-challenges/blob/main/questions/00612-medium-kebabcase/README.md) 字符串格式转换，`FooBarBaz -> foo-bar-baz`

等等，这些笔者没有列举具体实现，是因为认为大部分开发中用的还是不多的，如果你直接一眼就能想出方案，可能也不用去看了。

### Append Argument

[原文地址](https://github.com/type-challenges/type-challenges/blob/main/questions/00191-medium-append-argument/README.md)

使用：

```typescript
type Fn = (a: number, b: string) => number

type Result = AppendArgument<Fn, boolean> 
// expected be (a: number, b: string, x: boolean) => number
```

这个例子可能会在我们日常开发中用到，而且可以让我们回顾如何操作函数类型：

```typescript
export type AppendArgument<Fn extends (...args: any) => any, A> = Fn extends (
  ...args: infer Args
) => infer Res
  ? (...arg: [...Args, A]) => Res
  : never;
```

### Append to object

[原文地址](https://github.com/type-challenges/type-challenges/blob/main/questions/00527-medium-append-to-object/README.md)

使用：

```typescript
type Test = { id: '1' }
type Result = AppendToObject<Test, 'value', 4> // expected to be { id: '1', value:
```

注意和 `Merge` 有所区别，这里是针对 `Object` 来操作

```typescript
export type AppendToObject<T, U, V> = T extends Record<string, any>
  ? U extends string
    ? { [K in keyof T | U]: K extends U ? V : T[K] }
    : T
  : T;
```

## 其他

Typescript 特性相对完备，基于此可以完成非常复杂的需求，甚至使用 [Typescript 来编写一个 Typescript Checker](https://github.com/ronami/HypeScript)，不过笔者认为，对于时间精力有限的一般工作中的开发者来说，知道“可以这样做”，并且在适当的时候可以通过简单的资料查阅完成需求，这一点可能更重要。

经过权衡，本文中只列举了部分 [type--challenges](https://github.com/type-challenges/type-challenges) 中的内容，如果你还想了解更多，不妨看看原 github 仓库。