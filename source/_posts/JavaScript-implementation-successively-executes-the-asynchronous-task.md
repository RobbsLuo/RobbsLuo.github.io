---
title: JavaScript实现依次执行异步任务
date: 2018-06-14 17:10:10
tags:    
    - JavaScript    
    - Async
categories:
    - JavaScript
description: 现有 n 个异步任务，这 n 个异步任务是依次执行且下一个异步任务依赖上一个异步任务的结果作参数，问如何实现。
---

## 问题描述
现有 n 个异步任务，这 n 个异步任务是依次执行且下一个异步任务依赖上一个异步任务的结果作参数，问如何实现。

## 解法1：for 循环 + await

简单的 for 循环是依次进行循环的，不像 Array.forEach，Array.map 方法是并发执行的，利用这一特点加 async / await 很容易写出下面这样的代码：

```Javascript
(async () => {
  const sleep = delay => {
    return new Promise((resolve, reject) => {
      setTimeout(_ => resolve(), delay)
    })
  }
  
  const task = (i) => {
    return new Promise(async (resolve, reject) => {
      await sleep(500)
      console.log(`now is ${i}`)
      ++i
      resolve(i)
    })
  }
  
  let param = 0
  for (let i = 0; i < 4; i++) {
    param = await task(param)
  }  
})()
```

输出：

```
now is 0
now is 1
now is 2
now is 3
```

效果虽然做到了，但是看到 param 这个局部变量就很不爽，请看解法2。


## 解法2：Array.prototype.reduce

关于 `Array.prototype.reduce` 方法相信大部分小伙伴初见时都是用来数组求和。不了解的小伙伴可以点击链接了解下 [reduce](https://developer.mozilla.org/zhCN/docs/Web/JavaScript/Reference/Global_Objects/Array/Reduce) 。

`reduce`有`初始值`，`积累值`，以及`当前值`的概念。其中 `积累值`可以看作是前一个值，通过返回`积累值`又可以看作是 下一个值（可能说的比较绕，可以参照 Redux 的 中间件执行顺序 的源码，也是用的 reduce）。使用`reduce`来解决问题的代码为：

```javascript
const sleep = delay => {
  return new Promise((resolve, reject) => {
    setTimeout(_ => resolve(), delay)
  })
}

const task = (i) => {
  return new Promise(async (resolve, reject) => {
    await sleep(500)
    console.log(`now is ${i}`)
    ++i
    resolve(i)
  })
}

[task, task, task, task].reduce(async (prev, task) => {
  const res = await prev
  return task(res)
}, 0)
```

输出：

```
now is 0
now is 1
now is 2
now is 3
```
可以这样理解 `prev` 和 `task`：

* prev：前一个 异步任务（promise）
* task：当前的异步任务

当前的异步任务需要上一个异步任务的结果作参数，故很显然要 `await prev`。