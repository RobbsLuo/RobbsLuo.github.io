---
title: Implementing Sequential Execution of Asynchronous Tasks in JavaScript
date: 2018-06-14 17:10:10
tags:    
    - JavaScript    
    - Async
categories:
    - JavaScript
lang: en
description: Given n asynchronous tasks, these n asynchronous tasks must be executed sequentially, and each subsequent task depends on the result of the previous one as its parameter. How do we implement this?
---

## Problem Statement

Given n asynchronous tasks, these n asynchronous tasks must be executed sequentially, and each subsequent asynchronous task depends on the result of the previous one as its parameter. How do we implement this?

## Solution 1: for loop + await

A simple for loop iterates sequentially, unlike Array.forEach and Array.map, which execute concurrently. Leveraging this characteristic along with async / await, it is easy to write code like the following:

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

Output:

```
now is 0
now is 1
now is 2
now is 3
```

Although this achieves the desired effect, the `param` local variable is rather unpleasant to look at. See Solution 2.


## Solution 2: Array.prototype.reduce

When most people first encounter the `Array.prototype.reduce` method, they use it to sum an array. If you are not familiar with it, you can click the link to learn about [reduce](https://developer.mozilla.org/zhCN/docs/Web/JavaScript/Reference/Global_Objects/Array/Reduce).

`reduce` has the concepts of an `initial value`, an `accumulator`, and a `current value`. The `accumulator` can be thought of as the previous value, and by returning the `accumulator` it can also be thought of as the next value (this may sound convoluted; you can refer to the source code of Redux's middleware execution order, which also uses reduce). The code that uses `reduce` to solve the problem is:

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

Output:

```
now is 0
now is 1
now is 2
now is 3
```

You can understand `prev` and `task` as follows:

* prev: the previous asynchronous task (promise)
* task: the current asynchronous task

The current asynchronous task needs the result of the previous asynchronous task as its parameter, so it obviously needs to `await prev`.
