---
title: JavaScript Currying
date: 2018-06-11 08:55:54
tags:
    - JavaScript
    - Currying
categories:
    - JavaScript
description: Currying柯里化是函数式语言都有的一个特性，如Perl，Python，JavaScript。本篇就借用一下JavaScript，介绍一下柯里化的思想及应用。
---

`Currying`柯里化是函数式语言都有的一个特性，如Perl，Python，JavaScript。本篇就借用一下JavaScript，介绍一下柯里化的思想及应用。假设函数库里提供这样一个拼接URL地址的函数：

```javascript
function simpleURL(protocol, domain, path) {
    return protocol + "://" + domain + "/" + path;
}
simpleURL('http','www.jackzxl.net', 'index.html');     //http://www.jackzxl.net/index.html
```

这是个最普通的函数毫无新意。但对于你的站点来说，第一个参数固定为`http`，第二个参数固定为`www.jackzxl.net`，唯一需要改变的是第三个参数。即你的站点中的任何页面或资源，前两个参数永远固定，只需要改变第三个参数。

显然你不想每次调用时都手动敲一下前两个参数，麻烦不说，还容易出错。怎么办呢？你会想直接将库函数改成单参不就行了？

```javascript
function simpleURL(path) {
    return "http://www.jackzxl.net/" + path;
}
```

这样改有两个问题，首先如果该库函数还需要被其他人或其他地方使用，直接改库函数这条路是绝对行不通的。其次就算你对函数有绝对的控制权，这样改显得也非常的不灵活，如果哪天你的站点要加上SSL呢？总不能把第一个参数再放回去吧。因此你正确的选择是柯里化。所谓柯里化就是：将函数与其参数的一个子集绑定起来后返回个新函数。如果感觉比较抽象，可以做一些类比，比如C++模板里的偏特化，这样理解起来能容易点。将上例柯里化一下：

```javascript
var myURL = simpleURL.bind(null, 'http', 'www.jackzxl.net');
myURL('myfile.js');     //http://www.jackzxl.net/myfile.js

//站点加上SSL
var mySslURL = simpleURL.bind(null, 'https', 'www.jackzxl.net');
mySslURL('myfile.js');  //https://www.jackzxl.net/myfile.js
```

上述代码用`bind`来实现柯里化。再回过头体会一下柯里化定义：将函数与其参数的一个子集绑定起来后返回个新函数。柯里化后发现函数变得更灵活，更流畅，是一种简洁的实现函数委托的方式为何用bind来实现柯里化呢？因为简单嘛，有现成的就不必自己造轮子了。但因为本篇介绍的是柯里化，所以我们自己实现一下柯里化，来加深理解。它需要满足两点：参数子集，返回新函数：

```javascript
var currying = function(fn) {
    var args = [].slice.call(arguments, 1);
    return function() {
        var newArgs = args.concat([].slice.call(arguments));
        return fn.apply(null, newArgs);
    };
};

var myURL2 = currying(simpleURL, 'https', 'www.jackzxl.net');
myURL2('myfile.js');    //http://www.jackzxl.net/myfile.js
```

效果和用bind是一样的，我们仔细分析一下自定义的currying函数，首先参数fn是需要柯里化的simpleURL函数，后面均为可变参数（函数的arguments可参考这里），currying里每行代码的执行结果如下：

```javascript
var currying = function(fn) {
    var args = [].slice.call(arguments, 1);
    //args为["https", "www.jackzxl.net"]

    return function() {
        var newArgs = args.concat([].slice.call(arguments));
        //newArgs为["https", "www.jackzxl.net", "myFile.js"]

        return fn.apply(null, newArgs);
        //相当于return simpleURL("https", "www.jackzxl.net", "myFile.js");
    };
};
```

上面已经说明了柯里化的原理和实现。那究竟柯里化有什么作用呢？常见的作用是：

* 参数复用
* 延迟运行
* 扁平化

`参数复用`上面例子已经展示了，不赘述。

`延迟运行`其实非常直观，因为不是返回运算结果，而是返回新函数，当然是延迟运行啦。例如bind就是延迟执行的代表，不赘述

`扁平化`的函数更加易读。例如你要从站点的JSON数据里获取所有文章的title：

```javascript
//JSON数据
{
    "user": "Jack",
    "posts": [
        { "title": "JavaScript Curry", "contents": "..." },
        { "title": " JavaScript Function", "contents": "..." }
    ]
}

//从JSON数据中获取所有文章的title
fetchFromServer()
    .then(JSON.parse)
    .then(function(data){ return data.posts })
    .then(function(posts){
        return posts.map(function(post){ return post.title })
    })
```

当然你可能写出更优雅的代码…但这不是重点。重点是用柯里化将代码更加易读易维护：

```javascript
var curry = require('curry');
var get = curry(function(property, object){ return object[property] });

fetchFromServer()
    .then(JSON.parse)
    .then(get('posts'))
    .then(map(get('title')))
```

**提前返回？**

最后网上还有个作用是提前返回，例如IE的事件和其他浏览器不同，为实现兼容性，可以这样实现：

```javascript
var addEvent = (function(){
    if (target.addEventListener) {
        return function(target, eventType, handler) {
            target.addEventListener(eventType, handler, false);
        };
    } else {        //IE
        return function(target, eventType, handler) {
            target.attachEvent("on" + eventType, handler);
        };
    }
})();  
```

但在我看来，这里用柯里化意义不大。因为柯里化虽然优点很多，缺点同样明显，就是学习成本有点高。用柯里化实现“提前返回”，维护的成本大于收益。

不用柯里化怎么实现呢？一个三元运算符就搞定了：

```javascript
var addHandler = document.body.addEventListener ?
    function(target, eventType, handler){
        target.addEventListener(eventType, handler, false);
    } :
    function(target, eventType, handler){
        target.attachEvent("on" + eventType, handler);
    };
```

或者函数内部重写该函数也行：

```javascript
function addHandler(target, eventType, handler){
    if (target.addEventListener){
        addHandler = function(target, eventType, handler){  //重写该函数
            target.addEventListener(eventType, handler, false);
        };
    } else {        //IE
        addHandler = function(target, eventType, handler){  //重写该函数
            target.attachEvent("on" + eventType, handler);
        };
    }
    addHandler(target, eventType, handler); //调用新函数
}
```

两种方法都非常直观，简单明了，不要为了用柯里化而用柯里化。

## 总结

柯里化虽然有一个神秘的名字，但其实说穿了并不神秘。在前端它的应用场并不多（当然也可能我经验比较浅），更多的应该是用在后端异步函数里，如Node.js，对于异步API用柯里化可以减少回调嵌套。

---
> 来源：https://www.jianshu.com/p/9b6b5c7527fc