## `stream` 是什么
`stream` 译作流，是对[生产者和消费者问题](https://zh.wikipedia.org/zh-hans/%E7%94%9F%E4%BA%A7%E8%80%85%E6%B6%88%E8%B4%B9%E8%80%85%E9%97%AE%E9%A2%98)的解法，`stream` 是类似于数组或者字符串那样的数据集合，它还包含一系列的状态控制，stream 就是一个状态管理单元。
> 在node中，流可以帮助我们将事情的重点分为几份，因为使用流可以帮助我们将实现接口的部分分割成一些连续的接口，这些接口都是可重用的。接着，你可以将一个流的输出口接到另一个流的输入口，然后使用使用一些库来对流实现高级别的控制。  

流（`stream`）在 `Node.js` 中是处理流数据的抽象接口（`abstract interface`）。 `stream` 模块提供了基础的 API 。使用这些 API 可以很容易地来构建实现流接口的对象。  

`Node.js` 提供了多种流对象。 例如， HTTP 请求 和 `process.stdout` 就都是流的实例。  

流可以是可读的、可写的，或是可读写的。所有的流都是 [`EventEmitter`](http://nodejs.cn/api/events.html#events_class_eventemitter) 的实例。  

## 为什么要使用 `stream`
在node中，I/O都是异步的，node 通过回调函数的方式来传递状态。比如一段响应客户端请求的代码：
```javascript
const http = require('http');
const fs = require('fs');
const path = require('path');

const server = http.createServer((req, res) => {
    // req 是 http.IncomingMessage 的实例，是一个 Readable Stream
    // res 是 http.ServerResponse 的实例，是一个 Writable Stream
    fs.readFile(path.resolve(__dirname, './data.txt'), (err, data) => {
        res.end(data);
    });
});
server.listen(8080);
```
上面的代码可以达到预期的效果，但是在接收到客户端的请求后，程序需要将`data.txt`整个文件加载到内存之后再发送给客户端。这样既耗服务器资源又影响用户体验（等待`data.txt`完整从磁盘加载到内存的过程）。  
由于请求实例`req, res`的特殊性，可以很容易的使用流API改善以上代码：
```js
const http = require('http');
const fs = require('fs');
const path = require('path');

const server = http.createServer((req, res) => {
    let stream = fs.createReadStream(path.resolve(__dirname, './data.txt'));
    stream.pipe(res);
});
server.listen(8080);
```
`ReadbaleStream`的 `pipe()` 方法会自动帮助我们监听 `data` 和 `end` 事件，`data.txt` 中的数据像水流源源不断的被发送到客户端。使用 `pipe()` 还可以自动控制后端的压力，以便在客户端连接缓慢的时候node可以将尽可能少的缓存放到内存中。

## `stream` 的种类
可以分两个维度来理解流`(stream)`:  
- 使用者角度：可以分为**消费者流(Stream Consumers)**和**实施者流(Stream Implementers)**
- 类型角度：可以分为以下四种
    - Readable - 可读的流 (例如 fs.createReadStream()).
    - Writable - 可写的流 (例如 fs.createWriteStream()).
    - Duplex - 可读写的流 (例如 net.Socket).
    - Transform - 在读写过程中可以修改和变换数据的 Duplex 流 (例如 zlib.createDeflate()).

大多数讨论`stream`的时候，习惯按类型来将其分成4类，但是从使用者角度能更方便理解`stream`，node 官网也是从这个角度介绍`stream`的API。  
这里不过多罗列 API ，按常规的四种类型来加强对 `Stream` 的理解。

## `Readable Stream`
可读流（`Readable streams`）是对提供数据的 源头 （`source`）的抽象。  
所有的 `Readable` 都实现了 `stream.Readable` 类定义的接口。

### 可读流有**两种模式**，**三个状态**
可读流事实上工作在下面两种模式之一：`flowing` 和 `paused (non-flowing)` 。  
在 flowing 模式下， 可读流自动从系统底层读取数据，并通过 `EventEmitter` 接口的事件尽快将数据提供给应用。  
在 paused 模式下，必须显式调用 `stream.read()` 方法来从流中读取数据片段。  

在任意时刻，任意可读流应确切处于下面三种状态之一：  
- `readable._readableState.flowing = null`  
不存在数据消费者，可读流将不会产生数据。 在这个状态下，监听 `'data'` 事件，调用 `readable.pipe()` 方法，或者调用 `readable.resume()` 方法， `readable._readableState.flowing` 的值将会变为 `true` 。这时，随着数据生成，可读流开始频繁触发事件。
- `readable._readableState.flowing = false`
调用 `readable.pause()` 方法， `readable.unpipe()` 方法， 或者接收 “背压”（back pressure）， 将导致 `readable._readableState.flowing` 值变为 `false`。 这将暂停事件流，但 **不会** 暂停数据生成。 在这种情况下，为 `'data'` 事件设置监听函数不会导致 `readable._readableState.flowing` 变为 `true`。
- `readable._readableState.flowing = true`
监听 `'data'` 事件，调用 `readable.pipe()` 方法，或者调用 `readable.resume()` 方法后

---

在 `stream` 上绑定 `ondata` 方法会自动触发流动模式(`Flowing Mode`):  
```js
const readable = getReadableStreamSomehow();
readable.on('data', (chunk) => {
  console.log(`Received ${chunk.length} bytes of data.`);
});
```
这种模式流程图如下：  
![flowing mode](https://fedt-blog.b0.upaiyun.com/uploads/1525507688000.png)

 资源的数据不是直接流向消费者，而是先`push`到缓冲池，缓存池有一个水位标记 `highWatermark`，超过这个标记阈值，`push` 的时候会返回 `false`。这一现象叫做 *背压*。有两种情况会产生背压：
 - 消费者主动调用了 `.pause()`
 - 消费速度比数据`push`到缓冲池的生产速度慢

 调用 `readable.pause()` 方法， `readable.unpipe()` 方法， 或者接收 “背压”（back pressure）进入暂停模式  
 这种模式流程图如下：  
 ![non-flowing mode](https://fedt-blog.b0.upaiyun.com/uploads/1525513164000.png)

> `Readble Stream` 相关代码：[readableStream.js](https://github.com/DOTA2mm/learning-code/blob/master/Srteam/readableStream.js)

作为实现者产生可读流，需要实现`_read()` 方法，在该方法内调用`push()`方法将数据写入缓冲区，消费者则不应该调用`push()`方法，如我们不应该在data事件或者readable事件中调用该方法。这样做只有在消费者消费数据的时候才缓冲区才产生数据，能减轻缓存的压力。  

### `pipe`
`pipe` 可以理解为管道，使用它可以实现各种不同流的连接，如将输入流和输出流连接，将输入流输入的数据，输出到输出流中。  
一般的做法就是监听`data`事件，然后创建一个输出流，然后往输出流写数据。然而使用pipe管道，直接将输出流接入到pipe中即可。  
其实只是pipe内部替我们实现了data事件，所以也会将流的模式转化为流动模式，并且，这个方法会自动调节流量，所以当快速读取可读流时目标不会溢出。而且该方法返回流本身，因此可以不断调用pipe，形成链式调用：  
```js
var r = fs.createReadStream('file.txt');
var z = zlib.createGzip();
var w = fs.createWriteStream('file.txt.gz');
r.pipe(z).pipe(w);
```

## `Writable Stream`
原理与 Readable Stream 是比较相似的，数据流过来的时候，会直接写入到资源池，当写入速度比较缓慢或者写入暂停时，数据流会进入队列池缓存起来，如下图所示：  
![writable stream](https://fedt-blog.b0.upaiyun.com/uploads/1525514931000.png)

当生产者写入速度过快，把队列池装满了之后，就会出现「背压」，这个时候是需要告诉生产者暂停生产的，当队列释放之后，Writable Stream 会给生产者发送一个 drain 消息，让它恢复生产。  
```js
var fs = require('fs');
var rs = fs.createReadStream('source/file');
var ws = fs.createWriteStream('dest/file');

rs.on('data', function(chunk){
  if(ws.write(chunk) === false){ // 尚未写完，停止读取
    rs.pause();
  }
});

ws.on('drain', function(){
  rs.resume(); // 数据已经写完，继续读取
});

rs.on('end', function(){ // 已经没有跟多数据，关闭可写流
  ws.end();
});
```
上面代码可以用 `pipe` 大幅度简化： `rs.pipe(ws)`;

作为实施者来实现可写流需要实现`_write`方法：  
```js
/**
* @param {} chunk 被写入的资源
* @param {} encoding 如果数据块是一个字符串，那么这就是编码的类型。如果是一个buffer，那么则会忽略它
* @param {} callback 当你处理完给定的数据块后调用这个函数。回调函数使用标准的callback(error)模式来表示这个写操作成功或发生了错误
*/
_write(chunk, encoding, callback) {...}
```

> `Writable Stream` 相关代码：[writableStream.js](https://github.com/DOTA2mm/learning-code/blob/master/Srteam/writableStream.js)


## `Duplex Stream`
这是一种“双工流”，既是可读的，也是可写的。由于javascript不具备多重继承，所以该类是继承了`Readable`类，并寄生于`Writable`类。所以实现该类的时候，需要我们去重写`_read(n)`和`_write(chunk,encoding,cb)`方法。  

我们可以通过 options 参数来配置它为只可读、只可写或者半工模式，一个简单的 Demo：
```js
var Duplex = require('stream').Duplex

const duplex = Duplex();

// readable
let i = 2;
duplex._read = function () {
  this.push(i-- ? 'read ' + i : null);
};
duplex.on('data', data => console.log(data.toString()));

// writable
duplex._write = function (chunk, encoding, callback) {
  console.log(chunk.toString());
  callback();
};
duplex.write('write');

// write
// read 1
// read 0
```

## `Transform Stream`
转换流也是一个双工流，用以处理输入输出是因果相关，位于管道中间层的 Transform 是即可读也可写的。  
![Transform Stream](https://fedt-blog.b0.upaiyun.com/uploads/1525517229000.png)  

该流不仅要实现`_read()`和`_write()`方法，还有实现`_transform()`方法，并且可选的实现`_flush()`方法。

`new stream.Transform([options])`  
options Object 同时传递给`Writable`和`Readable`构造函数。如`ObjectMode:true`  
`_transform` 方法在每次 stream 中有数据来了之后都会被执行  
```js
_transform = function(chunk, encoding, cb){...}
```
在该方法中，可以进行数据处理，例如小写字母变大写。
调用`transform.push()`，则可以往输出流中写入数据。给后续的输入流使用。
仅当目前的数据块被完全消费后，才会调用回调函数。  
需要注意的是如果将数据传入回调函数的**第二个**参数，那么数据将会被传递给push方法，也就等价于调用了`push()`。下面的两种情况是等价的：  
```js
transform.prototype._transform = function (data, encoding, callback) {
  this.push(data);
  callback();
}

transform.prototype._transform = function (data, encoding, callback) {
  callback(null, data);
}
```
`_flush(callback)`  
在所有的数据块都被 _transform 方法处理过后，才会调用 _flush 方法。所以它的作用就是处理残留数据的。  

关于transform，这里有一篇示例，[通过Node.js Stream API 实现逐行读取的实例](http://segmentfault.com/a/1190000000740718)  

> `Transform  Stream` 相关代码：[transform.js](https://github.com/DOTA2mm/learning-code/blob/master/Srteam/transform.js)

## 参考链接
- [Node.js中文网 - stream](http://nodejs.cn/api/stream.html)
- [深入理解 Node Stream 内部机制](https://www.barretlee.com/blog/2017/06/06/dive-to-nodejs-at-stream-module/)
- [Node.js关于Stream的理解](https://github.com/zhengweikeng/blog/issues/4)
- [stream-handbook中文版](https://github.com/jabez128/stream-handbook)
