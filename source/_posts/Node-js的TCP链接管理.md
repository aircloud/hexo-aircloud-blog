---
title: Node.js 的 TCP 链接管理
date: 2018-11-25 19:23:09
tags:
    - TCP
    - Node.js
---

在 Node.js 的微服务中，一般不同的服务模块我们会采用 TCP 进行通信，本文来简单谈一谈如何设计 TCP 服务的基础管理。

>在具体设计上，本文参考了微服务框架 [Seneca](https://github.com/senecajs/seneca) 所采用的通信方案 [Seneca-transport](https://github.com/senecajs/seneca-transport)，已经被实践所证明其可行性。

一提到 TCP 通信，我们肯定离不开 `net` 模块，事实上，借助 `net` 模块，我们也可以比较快速地完成一般的 TCP 通信的任务。

为了避免对基础的遗忘，我们还是先附上一个基本的 TCP 链接代码：

```javascript
//server.js:
const net = require('net');

const server = net.createServer((socket) => {
    socket.write('goodbye\n');
    socket.on('data', (data) => {
        console.log('data:', data.toString());
        socket.write('goodbye\n');
    })
}).on('error', (err) => {
    throw err;
});

// grab an arbitrary unused port.
server.listen(8024, () => {
    console.log('opened server on', server.address());
});

//client.js:
const net = require('net');

const client = net.createConnection({ port: 8024 }, () => {
    //'connect' listener
    console.log('connected to server!');
    client.write('world!\r\n');
    setInterval(() => {
        client.write('world!\r\n');
    }, 1000)
});
client.on('data', (data) => {
    console.log(data.toString());
    // client.end();
});
client.on('end', () => {
    console.log('disconnected from server');
});
```

其实，上述已经是一个几乎最简单的客户端和服务端通信 Demo，但是并不能在实际项目中使用，首先我们需要审视，其离生产环境还差哪些内容：

1. 以上要求 Server 端要在 Client 端之前启动，并且一旦因为一些错误导致 Server 端重启了并且这个时候 Client 端正好和 Server 端进行通信，那么肯定会 crash，所以，我们需要一个更为平滑兼容的方案。
2. 以上 TCP 链接的 Server 部分，并没有对 connection 进行管理的能力，并且在在以上的例子中，双方都没有主动释放链接，也就是说，建立的是一个 TCP 长连接。
3. 以上链接的处理数据能力有限，只能处理纯文本的内容，并且还有一定的风险性（你也许会说可以用 JSON 的序列化反序列化的方法来处理 JSON 数据，但是你别忘了 `socket.on('data'...` 很可能接收到的不是一个完整的 JSON，如果 JSON 较长，其可能只接收到一般的内容，这个时候如果直接 `JSON.parse())` 很可能就会报错）。

以上三个问题，便是我们要解决的主要问题，如果你看过之后立刻知道该如何解决了，那么这篇文章可能你不需要看了，否则，我们可以一起继续探索解决方案。

### 使用 reconnect-core

[reconnect-core](https://www.npmjs.com/package/reconnect-core) 是一个和协议无关的链接重试算法，其工作方式也比较简单，当你需要在 Client 端建立链接的时候，其流程是这样的：

* 调用事先传入的链接建立函数，如果这个时候返回成功了，即成功建立链接。
* 如果第一次建立链接失败了，那么再隔一段时间建立第二次，如果第二次还是失败，那么再隔一段更长的时间建立第三次，如果还是失败，那么再隔更长的一段时间……直到到达最大的尝试次数。

实际上关于尝试的时间间隔，也会有不同的策略，比较常用的是 Fibonacci 策略和 exponential 策略。

当然，关于策略的具体实现，reconnect-core 采用了一个 [backoff](https://www.npmjs.com/package/backoff) 的库来管理，其可以支持  Fibonacci 策略和 exponential 策略以及更多的自定义策略。

对于上面提到的 DEMO 代码。我们给出 Client 端使用 reconnect-core 的一个实现：

```javascript
//client.js:
const Reconnect = require('reconnect-core');
const net = require('net');
const Ndjson = require('ndjson');

const Connect = Reconnect(function() {
    var args = [].slice.call(arguments);
    return net.connect.apply(null, args)
});

let connection = Connect(function(socket) {
    socket.write('world!\r\n');
    socket.on('data', (msg) => {
        console.log('data', msg.toString());
    });
    socket.on('close', (msg) => {
        console.log('close', msg).toString();
        connection.disconnect();
    });
    socket.on('end', () => {
        console.log('end');
    });
});

connection.connect({
    port: 8024
});
connection.on('reconnect', function () {
    console.log('on reconnect...')
});
connection.on('error', function (err) {
   console.log('error:', err);
});
connection.on('disconnect', function (err) {
   console.log('disconnect:', err);
});
```
>采用 Reconnect 实际上相比之前是多了一层内容，我们在这里需要区分 connection 实例和 socket 句柄，并且附加正确的时间监听。

现在，我们就不用担心到底是先启动服务端还是先启动客户端了，另外，就算我们的服务端在启动之后由于某些错误关闭了一会，只要没超过最大时间（而这个也是可配置的），仍然不用担心客户端与其建立连接。


### 给 Server 端增加管理能力

给 Server 端增加管理能力是一个比较必要的并且可以做成不同程度的，一般来说，最重要的功能则是及时清理链接，常用的做法是收到某条指令之后进行清理，或者到达一定时间之后定时清理。

这里我们可以增加一个功能，达到一定时间之后，自动清理所有链接：

```javascript
//server.js
const net = require('net');

var connections = [];

const server = net.createServer((socket) => {
    connections.push(socket);
    socket.write('goodbye\n');
    socket.on('data', (data) => {
        console.log('data:', data.toString());
        socket.write('goodbye\n');
    })
}).on('error', (err) => {
    throw err;
});

setTimeout(() => {
    console.log('clear connections');
    connections.forEach((connection) => {
        connection.end('end')
        // connection.destory()
    })
}, 10000);

// grab an arbitrary unused port.
server.listen(8024, () => {
    console.log('opened server on', server.address());
});
```

我们可以通过`connection.end('end')` 和 `connection.destory()` 来清理，一般来说，前者是正常情况下的关闭指令，需要 Client 端进行确认，而后者则是强制关闭，一般在出错的时候会这样调用。

### 使用 ndjson 来格式化数据

[ndjson](https://www.npmjs.com/package/ndjson) 是一个比较方便的 JSON 序列化/反序列化库，相比于我们直接用 JSON，其好处主要体现在：

* 可以同时解析多个 JSON 对象，如果是一个文件流，即其可以包含多个 `{}`，但是要求则是每一个占据一行，其按行分割并且解析。
* 内部使用了 [split2](https://www.npmjs.com/package/split2)，好处就是其返回时可以保证该行的所有内容已经接受完毕，从而防止 ndjson 在序列化的时候出错。

关于 ndjson 的基本使用，可以根据上述链接查找文档，这里一般情况下，我们的使用方式如下（以下是一个 demo）：

```javascript
//server.js:
const net = require('net');

var connections = [];

const server = net.createServer((socket) => {
    connections.push(socket);
    socket.on('data', (data) => {
        console.log('data:', data.toString());
        socket.write('{"good": 1234}\r\n');
        socket.write('{"good": 4567}\n\n');
    })
}).on('error', (err) => {
    throw err;
});

// grab an arbitrary unused port.
server.listen(8024, () => {
    console.log('opened server on', server.address());
});

//client.js:
const Reconnect = require('reconnect-core');
const net = require('net');
const Ndjson = require('ndjson');
var Stream = require('stream');

const Connect = Reconnect(function() {
    var args = [].slice.call(arguments);
    return net.connect.apply(null, args)
});

let connection = Connect(function(socket) {
    socket.write('world!\r\n');
    var parser = Ndjson.parse();
    var stringifier = Ndjson.stringify();

    function yourhandler(){
        var messager = new Stream.Duplex({ objectMode: true });
        messager._read = function () {
            // console.log('data:', data);
        };
        messager._write = function (data, enc, callback) {
            console.log(typeof data, data);
            // your handler
            return callback()
        };
        return messager
    }
    socket // 链接句柄
        .pipe(parser)
        .pipe(yourhandler())
        .pipe(stringifier)
        .pipe(socket);

    socket.on('close', (msg) => {
        console.log('close', msg).toString();
        connection.disconnect();
    });
    socket.on('end', (msg) => {
        console.log('end', msg);
    });
});
connection.connect({
    port: 8024
});
connection.on('reconnect', function () {
    console.log('on reconnect...')
});
connection.on('error', function (err) {
   console.log('error:', err);
});
connection.on('disconnect', function (err) {
   console.log('disconnect:', err);
});
```

其中，用户具体的逻辑代码，可以是 `yourhandler` 函数 `_write` 里面的一部分，其接收的是一个一个处理好的对象。

