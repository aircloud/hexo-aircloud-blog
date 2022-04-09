---
title: 使用 fst-json 自动生成更快的 json 序列化方法
date: 2022-02-11 18:45:54
tags:
    - 前端综合
---

> fst-json 的全称是 "fast-safe-typescript json"，它的本质就是直接使用你定义好的 Typescript 文件，来生成更加高效的序列化方法。
> 其目的是利用现有的资源（开发过程编写的 Typescript 文件），在编译和开发阶段尽可能提高运行时性能，同时这个过程并没有额外的开发负担。

github: https://github.com/aircloud/fst-json/blob/master/README.zh-cn.md

知乎：https://zhuanlan.zhihu.com/p/466572196

## 背景

由于 JSON schema 这个概念是由 fastify 引入，我们先对此进行介绍。

[fastify](https://github.com/fastify/fastify) 是一个高性能 Node.JS 服务端框架，其特点就是高性能，而之所以高性能主要的原因就是它引入了 JSON schema，通过对参数增加约束，来获得更快的序列化速度。

同时，fastify 也开源了一个独立的 json 序列化库 [fast-json-stringify](https://github.com/fastify/fast-json-stringify)，可以在**非 fastify 的项目中使用**。

在 fastify 中，JSON schema 的大致写法如下：

```
const schema = {
  schema: {
    response: {
      200: {
        type: 'object',
        properties: {
          hello: {
            type: 'string'
          }
        }
      }
    }
  }
}
fastify
  .get('/', schema, function (req, reply) {
    reply
      .send({ hello: 'world' })
  })
```

我们可以看出，这一套写法不仅会带来额外的学习成本，而且由于目前大多数项目开发都是采用 Typescript，这套定义也会和我们的 Typescript 定义有所重复。

事实上，虽然上面的示例代码比较短小，但是在实际的项目中，接口比较多的情况下，这些代码的开发量和额外的学习/维护成本还是不容小视的。

那么有没有可能直接使用 Typescript，而不用重新定义 JSON schema 呢？

答案是有的。

[fst-json](https://github.com/aircloud/fst-json/blob/master/README.zh-cn.md) 就是这样一个工具，它可以通过复用我们在 Typescript 中定义的 schema，通过工具自动生成 fastify 需要的 schema，这样我们就无需额外维护 schema 定义了。

## 使用方式

接下来，我们简单介绍 fst-json 的使用方式，首先安装（全局或者安装到项目中）：

```
npm i fst-json -g
```

假设我们项目采用了 Typescript，事先已经有了 schema 文件：

```
export interface HellWorld {
  attr1: string;
  attr2: string;
  attr3?: string;
}
```

我们在项目目录下新建 .fstconfig.js，用于声明配置，配置如下：

```
module.exports = {
  sourceFiles: [
    './src/schema/*.ts'
  ],
  distFile: "./src/schema-dist.ts",
  format: 'fastify'
}
```

之后我们运行：

```
fst-json gen
```

然后此时会生成一个 `src/schema-dist.ts`，这里会有自动生成的 JSON schema 定义，接下来我们在项目中可以同时使用 JSON schema 定义和我们之前定义好的 Typescript 类型：

```
import * as schemas from './schema-dist';
import type { HellWorld  } from "./schema";

const schema = {
  schema: {
    response: {
      200: schemas.HellWorldSchema
    }
  }
}

server
  .get('/', schema, function (req, reply) {
    let res: HellWorld = {
      attr1: 'hello', 
      attr2: 'world', 
      attr3: 'optional'
    }

    reply
      .send(res);
  })
```

当然，fst-json 不仅仅可以在 fastify 中使用，也可以在任何其他需要 JSON 加速的地方使用，用法也都很简单，可以参考这个 [HelloWorld](https://github.com/aircloud/fst-json/tree/master/examples/helloworld)

## 原理和优势

fst-json，实际上是通过对 Typescript 进行语法树解析，针对 export 导出的各种类型生成对应的 fast-json-stringify 的 JSON schema，所以运行速度和手写是没有区别的。因此，它不仅仅能完全使用 fast-json-stringify 的效率优势，除了减少重复开发量以外还有如下优点：

* **根据 schema 进行字段校验：** 首先会进行 Tyepscript 语法校验，另外当缺失必须的属性（例如，当定义 interface 时没有被 `?` 修饰符修饰的属性缺失）的时候也会直接报错。
* **过滤不需要的 schema 字段：** 例如当把 Node.JS 当作 BFF 层的时候，可以严格按照 Typescript 的定义来返回字段，避免返回不需要的字段，从而避免上游服务的敏感字段被直接透传出去，也意味着从接口层面开始，真正做到 Fully Typed。
* **更快的序列化速度：** 根据 [fast-json-stringify](https://github.com/fastify/fast-json-stringify/issues) 的测试，能达到接近 2 倍的 JSON 序列化速度。

目前，fst-json 对常用的各类 interface、class、type 等类型定义都进行了支持，并且增加了各类 examples 和 90% 的覆盖率测试。

当然，由于 Typescript 的写法比较灵活。出于 JSON schema 本身的局限性，我们无法覆盖所有场景，所以也可以参考这里的[注意事项](https://github.com/aircloud/fst-json/blob/master/README.zh-cn.md#%E6%B3%A8%E6%84%8F%E4%BA%8B%E9%A1%B9)，有针对性的对比较容易出问题的写法进行规避。

### 局限性

fst-json 只是语法解析和生成工具，具体的运行时，实际上就是在使用 fast-json-stringify，也因此项目中需要安装 fast-json-stringify 依赖。

另外，针对 fast-json-stringify 的测试，在比较小的 payload 的情况下，它的速度是有优势的，当 payload 过大的时候，它的优势不再明显，甚至还不如 JSON.stringify。官方的描述是：

> fast-json-stringify is significantly faster than JSON.stringify() for small payloads.
> Its performance advantage shrinks as your payload grows.

不过事实上，这个时候你仍然可以使用 fst-json 做一些事情，例如笔者使用 fst-json 来做 bff 层对下游服务接口的持续集成兼容测试，在 Typescript 已经提前定义好了的情况下，每次测试的时候只需要请求依赖服务并且把响应字段序列化，如果没有报错并且字段序列化之后也没有变成 null（在比较复杂的接口定义中，如果个别属性定义类型和返回类型不一致，fast-json-stringify 是会直接转换成 null），就说明接口是没有变化的。可以有效避免依赖服务接口变化，却又没有及时同步到位造成暗坑的情况。


另外，其实目前 fast-json-stringify 生成序列化代码还是在运行时做的，这里的问题可能在于代码不透明，以及运行时开销和风险，笔者是希望将它的生成代码变成编译时去做，不过这样的话实际上有一点重复造轮子的错觉，所以目前还没有做这个事情。

---

最后 [fst-json](https://github.com/aircloud/fst-json/) 作为一个开源不久的小项目，肯定还有些需要优化和完善的地方，欢迎 star 支持和提出建议。

