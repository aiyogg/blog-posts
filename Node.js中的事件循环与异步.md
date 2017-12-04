## Node.js 中的事件循环（Event loop）与异步

Node.js的特点是事件循环，其中不同的事件会分配到不同的事件观察者身上，比如idle观察者，定时器观察者，I/O观察者等等，事件循环每次循环称为一次Tick，每次Tick按照先后顺序从事件观察者中取出事件进行处理。

下图是Node.js v0.9版本的事件循环示意图：  
![Node.js v0.9版本的事件循环示意图](https://fedt-blog.b0.upaiyun.com/article_img/node-event-loop.jpg)  

**当Node.js启动时会初始化event loop, 每一个event loop都会包含按如下顺序六个循环阶段，**  
```
   +-----------------------+
+->+        timers         |
|  +-----------------------+
|  +-----------------------+
|  |     I/O callbacks     |
|  +-----------------------+
|  +-----------------------+
|  |     idle, prepare     |
|  +-----------------------+      +---------------+
|  +-----------------------+      |   incoming:   |
|  |         poll          +<-----+  connections, |
|  +-----------------------+      |   data, etc.  |
|  +-----------------------+      +---------------+
|  |        check          |
|  +-----------------------+
|  +-----------------------+
+--+    close callbacks    |
   +-----------------------+
```
- timers 阶段: 这个阶段执行setTimeout(callback) and setInterval(callback)预定的callback;
- I/O callbacks 阶段: 执行除了 close事件的callbacks、被timers(定时器，setTimeout、setInterval等)设定的callbacks、setImmediate()设定的- callbacks之外的callbacks;
- idle, prepare 阶段: 仅node内部使用;
- poll 阶段: 获取新的I/O事件, 适当的条件下node将阻塞在这里;
- check 阶段: 执行setImmediate() 设定的callbacks;
- close callbacks 阶段: 比如socket.on(‘close’, callback)的callback会在这个阶段执行.
  
每一个阶段都有一个装有callbacks的fifo queue(队列)，当event loop运行到一个指定阶段时，
node将执行该阶段的fifo queue(队列)，当队列callback执行完或者执行callbacks数量超过该阶段的上限时，
event loop会转入下一下阶段.
> 注意上面六个阶段都不包括 process.nextTick()

### poll阶段
poll阶段是衔接整个event loop各个阶段比较重要的阶段，**在node.js里，任何异步方法（除timer,close,setImmediate之外）完成时，都会将其callback加到poll queue里,并立即执行。**
poll 阶段有两个主要的功能：  
- 处理poll队列（poll queue）的事件(callback);
- 执行timers的callback,当到达timers指定的时间时;
  
**如果event loop进入了 poll阶段，且代码未设定timer，将会发生下面情况：**
- 如果poll queue不为空，event loop将同步的执行queue里的callback,直至queue为空，或执行的callback到达系统上限;
- 如果poll queue为空，将会发生下面情况：
  - 如果代码已经被setImmediate()设定了callback, event loop将结束poll阶段进入check阶段，并执行check阶段的queue (check阶段的queue是 setImmediate设定的)
  - 如果代码没有设定setImmediate(callback)，event loop将阻塞在该阶段等待callbacks加入poll queue;

**如果event loop进入了 poll阶段，且代码设定了timer：**
- 如果poll queue进入空状态时（即poll 阶段为空闲状态），event loop将检查timers,如果有1个或多个timers时间时间已经到达，event loop将按循环顺序进入 timers 阶段，并执行timer queue.

### 示例
```js
var fs = require('fs');

function someAsyncOperation (callback) {
  var time = Date.now();
  // 花费9毫秒
  fs.readFile('/path/to/xxxx.pdf', callback);
}

var timeoutScheduled = Date.now();
var fileReadTime = 0;
var delay = 0;

setTimeout(function () {
  delay = Date.now() - timeoutScheduled;
}, 5);

someAsyncOperation(function () {
  fileReadtime = Date.now();
  while(Date.now() - fileReadtime < 20) {

  }
  console.log('setTimeout: ' + (delay) + "ms have passed since I was scheduled");
  console.log('fileReaderTime',fileReadtime - timeoutScheduled);
});
```
结果：setTimeout callback先执行，someAsyncOperation callback后执行
```
-> node eventloop.js
setTimeout: 7ms have passed since I was scheduled
fileReaderTime 9
```
解释：
当时程序启动时，event loop初始化：

1. timer阶段（无callback到达，setTimeout需要10毫秒）
2. i/o callback阶段，无异步i/o完成
3. 忽略
4. poll阶段，阻塞在这里，当运行5ms时，poll依然空闲，但已设定timer,且时间已到达，因此，event loop需要循环到timer阶段,执行setTimeout callback,1 由于从poll --> timer中间要经历check,close阶段,这些阶段也会消耗一定时间，因此执行setTimeout callback实际是7毫秒 然后又回到poll阶段等待异步1 i/o完成，在9毫秒时fs.readFile完成，其callback加入poll queue并执行。

### `setTimeout` 和 `setImmediate`

二者非常相似，但是二者区别取决于他们什么时候被调用.
- setImmediate 设计在poll阶段完成时执行，即check阶段；
- setTimeout 设计在poll阶段为空闲时，且设定时间到达后执行；但其在timer阶段执行

但当二者在异步i/o callback内部调用时，总是先执行setImmediate，再执行setTimeout:
```js
var fs = require('fs')

fs.readFile(__filename, () => {
  setTimeout(() => {
    console.log('timeout')
  }, 0)
  setImmediate(() => {
    console.log('immediate')
  })
})
// immediate
// timeout
```
理解了event loop的各阶段顺序这个例子很好理解：
因为fs.readFile callback执行完后，程序设定了timer 和 setImmediate，因此poll阶段不会被阻塞进而进入check阶段先执行setImmediate，后进入timer阶段执行setTimeout

### `process.nextTick()`
**process.nextTick()不在event loop的任何阶段执行，而是在各个阶段切换的中间执行,即从一个阶段切换到下个阶段前执行。**
```js
var fs = require('fs');

fs.readFile(__filename, () => {
  setTimeout(() => {
    console.log('setTimeout');
  }, 0);
  setImmediate(() => {
    console.log('setImmediate');
    process.nextTick(()=>{
      console.log('nextTick3');
    })
  });
  process.nextTick(()=>{
    console.log('nextTick1');
  })
  process.nextTick(()=>{
    console.log('nextTick2');
  })
});
/**
-> node eventloop.js
nextTick1
nextTick2
setImmediate
nextTick3
setTimeout
*/
```
从poll —> check阶段，先执行process.nextTick，  
nextTick1  
nextTick2  
然后进入check,setImmediate，  
setImmediate  
执行完setImmediate后，出check,进入close callback前，执行process.nextTick  
nextTick3  
最后进入timer执行setTimeout  
setTimeout  

process.nextTick()是node早期版本无setImmediate时的产物，node作者推荐我们尽量使用setImmediate。

### 事件循环与`Promise`

直接看例子： 
```js
setTimeout(function() {
  console.log(1)
}, 0);
new Promise(function executor(resolve) {
  console.log(2);
  for( var i=0 ; i<10000 ; i++ ) {
    i == 9999 && resolve();
  }
  console.log(3);
}).then(function() {
  console.log(4);
});
console.log(5);
```
首先先碰到一个 setTimeout，于是会先设置一个定时，在定时结束后将传递这个函数放到任务队列里面，因此开始肯定不会输出 1 。  

然后是一个 Promise，里面的函数是直接执行的，因此应该直接输出 2 3 。  

然后，Promise 的 then 应当会放到当前 tick 的最后，但是还是在当前 tick 中。   

因此，应当先输出 5，然后再输出 4 。  

最后在到下一个 tick，就是 1 。  

结果： 依次输出 2 3 5 4 1

### 参考资料
- [Node.js Event Loop 的理解 Timers，process.nextTick()](https://cnodejs.org/topic/57d68794cb6f605d360105bf)
- [Excuse me？这个前端面试在搞事！](https://zhuanlan.zhihu.com/p/25407758)
