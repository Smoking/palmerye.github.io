---
title: 从setTimeout/setInterval看JS线程
date: 2017-12-20 22:53:12
tags: [event loop, 线程]
---

> 最近项目中遇到了一个场景，其实很常见，就是定时获取接口刷新数据。那么问题来了，假设我设置的定时时间为1s，而数据接口返回大于1s，应该用同步阻塞还是异步？我们先整理下js中定时器的相关知识，再来看这个问题。

<!--more-->

## 初识setTimeout 与 setInterval

> 先来简单认识，后面我们试试用setTimeout 实现 setInterval 的功能

- setTimeout 延迟一段时间执行一次 **(Only one)**

``` JavaScript
setTimeout(function, milliseconds, param1, param2, ...)
clearTimeout() // 阻止定时器运行

e.g.
setTimeout(function(){ alert("Hello"); }, 3000); // 3s后弹出
```

- setInterval 每隔一段时间执行一次 **(Many times)**

``` JavaScript
setInterval(function, milliseconds, param1, param2, ...)

e.g.
setInterval(function(){ alert("Hello"); }, 3000); // 每隔3s弹出
```

> setTimeout和setInterval的延时最小间隔是4ms（W3C在HTML标准中规定）；在JavaScript中没有任何代码是立刻执行的，但一旦进程空闲就尽快执行。这意味着无论是setTimeout还是setInterval，所设置的时间都只是n毫秒被添加到队列中，而不是过n毫秒后立即执行。

## 进程与线程，傻傻分不清楚

为了讲清楚这两个抽象的概念，我们借用阮大大借用的比喻，先来模拟一个场景:
- 这里有一个大型工厂
- 工厂里有若干车间，每次只能有一个车间在作业
- 每个车间里有若干房间，有若干工人在流水线作业

那么：
- 一个工厂对应的就是计算机的一个CPU，平时讲的多核就代表多个工厂
- 每个工厂里的车间，就是**进程**，意味着同一时刻一个CPU只运行一个**进程**，其余**进程**在怠工
- 这个运行的车间（**进程**）里的工人，就是**线程**，可以有多个工人（**线程**）协同完成一个任务
- 车间（**进程**）里的房间，代表内存。

再深入点：
- 车间（**进程**）里工人可以随意在多个房间（内存）之间走动，意味着一个**进程**里，多个**线程**可以共享内存
- 部分房间（内存）有限，只允许一个工人（**线程**）使用，此时其他工人（**线程**）要等待
- 房间里有工人进去后上锁，其他工人需要等房间（内存）里的工人（**线程**）开锁出来后，才能才进去，这就是**互斥锁**（Mutual exclusion，缩写 Mutex）
- 有些房间只能容纳部分的人，意味着部分内存只能给有限的**线程**

再再深入:
- 如果同时有多个车间作业，就是**多进程**
- 如果一个车间里有多个工人协同作业，就是**多线程**
- 当然不同车间之间的工人也可以有相互协作，就需要协调机制

## JavaScript 单线程

总所周知，JavaScript 这门语言的核心特征，就是单线程（是指在**JS引擎**中负责解释和执行JavaScript代码的线程只有一个）。这和 JavaScript 最初设计是作为一门 GUI 编程语言有关，最初用于浏览器端，单一线程控制 GUI 是很普遍的做法。但这里特别要划个重点，虽然JavaScript是单线程，但**浏览器是多线程的！！！**例如Webkit或是Gecko引擎，可能有javascript引擎线程、界面渲染线程、浏览器事件触发线程、Http请求线程，读写文件的线程(例如在Node.js中)。ps：可能要总结一篇浏览器渲染的文章了。

> HTML5提出Web Worker标准，允许JavaScript脚本创建多个线程，但是子线程完全受主线程控制，且不得操作DOM。所以，这个新标准并没有改变JavaScript单线程的本质。

## 同步与异步，傻傻分不清楚

> 之前阮大大写了一篇[《JavaScript 运行机制详解：再谈Event Loop》](http://www.ruanyifeng.com/blog/2014/10/event-loop.html)，然后被[朴灵评注](https://app.yinxiang.com/shard/s8/sh/b72fe246-a89d-434b-85f0-a36420849b84/59bad790bdcf6b0a66b8b93d5eacbead)了，特别是同步异步的理解上，两位大牛有很大的歧义。

- 同步(synchronous)：假如一个函数返回时，调用者就能够得到预期结果(即拿到了预期的返回值或者看到了预期的效果)，这就是同步函数。

```
e.g.
alert('马上能看到我拉');
console.log('也能马上看到我哦');
```

- 异步(asynchronous)：假如一个函数返回时，调用者不能得到预期结果，需要通过一定手段才能获得，这就是异步函数。

```
e.g.
setTimeout(function() {
    // 过一段时间才能执行我哦
}, 1000);
```

### 异步构成要素

> 一个异步过程通常是这样的：主线程发起一个异步请求，相应的工作线程（比如浏览器的其他线程）接收请求并告知主线程已收到(异步函数返回)；主线程可以继续执行后面的代码，同时工作线程执行异步任务；工作线程完成工作后，通知主线程；主线程收到通知后，执行一定的动作(调用回调函数)。

- 发起（注册）函数 -- 发起异步过程
- 回调函数 -- 处理结果

```
e.g.
setTimeout(fn, 1000);
// setTimeout就是异步过程的发起函数，fn是回调函数
```

## 通信机制

> 异步过程的通信机制：工作线程将消息放到消息队列，主线程通过事件循环过程去取消息。

### 消息队列 Message Queue

> 一个先进先出的队列，存放各类消息。

### 事件循环 Event Loop

> 主线程（js线程）只会做一件事，就是从消息队列里面取消息、执行消息，再取消息、再执行。消息队列为空时，就会等待直到消息队列变成非空。只有当前的消息执行结束，才会去取下一个消息。这种机制就叫做事件循环机制**Event Loop**，取一个消息并执行的过程叫做一次循环。

![](https://github.com/palmerye/pictureBed/raw/master/blog/bV5mEF.png?fromMac)

> 工作线程是生产者，主线程是消费者。工作线程执行异步任务，执行完成后把对应的回调函数封装成一条消息放到消息队列中；主线程不断地从消息队列中取消息并执行，当消息队列空时主线程阻塞，直到消息队列再次非空。

## setTimeout(function, 0) 发生了什么

其实到这儿，应该能很好解释setTimeout(function, 0) 这个常用的“奇技淫巧”了。很简单，就是为了将`function`里的任务异步执行，0不代表立即执行，而是将任务推到消息队列的最后，再由主线程的事件循环去调用它执行。

> HTML5 中规定setTimeout 的最小时间不是0ms，而是4ms。

## setInterval 缺点

> 再次强调，定时器指定的时间间隔，表示的是何时将定时器的代码添加到**消息队列**，而**不是**何时执行代码。所以真正何时执行代码的时间是不能保证的，取决于何时被主线程的事件循环取到，并执行。

```
setInterval(function, N)
```

那么显而易见，上面这段代码意味着，每隔N秒把function事件推到消息队列中，什么时候执行？母鸡啊！

![](https://github.com/palmerye/pictureBed/raw/master/blog/bV5yai.png?fromMac)

上图可见，setInterval每隔100ms往队列中添加一个事件；100ms后，添加T1定时器代码至队列中，主线程中还有任务在执行，所以等待，`some event`执行结束后执行T1定时器代码；又过了100ms，T2定时器被添加到队列中，主线程还在执行T1代码，所以等待；又过了100ms，理论上又要往队列里推一个定时器代码，但由于此时T2还在队列中，所以T3不会被添加，结果就是此时被跳过；这里我们可以看到，T1定时器执行结束后马上执行了T2代码，所以并没有达到定时器的效果。

综上所述，setInterval有两个缺点：
- 使用setInterval时，某些间隔会被跳过；
- 可能多个定时器会连续执行；

### 链式setTimeout

```
setTimeout(function () {
    // 任务
    setTimeout(arguments.callee, interval);
}, interval)
```

> [警告](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Functions/arguments/callee)：在严格模式下，第5版 ECMAScript (ES5) 禁止使用 arguments.callee()。当一个函数必须调用自身的时候, 避免使用 arguments.callee(), 通过要么给函数表达式一个名字,要么使用一个函数声明.

上述函数每次执行的时候都会创建一个新的定时器，第二个setTimeout使用了arguments.callee()获取当前函数的引用，并且为其设置另一个定时器。好处：
- 在前一个定时器执行完前，不会向队列插入新的定时器（解决缺点一）
- 保证定时器间隔（解决缺点二）


## So...

回顾最开始的业务场景的问题，用同步阻塞还是异步，答案已经出来了...

PS：其实还有macrotask与microtask等知识点没有提到，总结了那么多，其实JavaScript深入下去还有很多，任重而道远呀。

----------

参考:

[进程与线程的一个简单解释 -- 阮大大](http://www.ruanyifeng.com/blog/2013/04/processes_and_threads.html)

[【译】JavaScript 如何工作的: 事件循环和异步编程的崛起 + 5 个关于如何使用 async/await 编写更好的技巧](https://juejin.im/post/5a221d35f265da43356291cc)
