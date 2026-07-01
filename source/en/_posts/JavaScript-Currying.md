---
title: JavaScript Currying
date: 2018-06-11 08:55:54
tags:
    - JavaScript
    - Currying
categories:
    - JavaScript
description: Currying is a feature common to functional languages such as Perl, Python, and JavaScript. This post uses JavaScript to illustrate the idea of currying and its applications.
lang: en
---

`Currying` is a feature common to functional languages such as Perl, Python, and JavaScript. This post uses JavaScript to illustrate the idea behind currying and its applications. Suppose a function library provides the following function that assembles a URL:

```javascript
function simpleURL(protocol, domain, path) {
    return protocol + "://" + domain + "/" + path;
}
simpleURL('http','www.jackzxl.net', 'index.html');     //http://www.jackzxl.net/index.html
```

This is a perfectly ordinary function with nothing remarkable about it. But for your own site, the first argument is fixed to `http`, and the second is fixed to `www.jackzxl.net`; only the third argument ever changes. In other words, for any page or resource on your site, the first two arguments are always the same, and you only need to vary the third.

Obviously you don't want to manually type those first two arguments every time you call the function — it's tedious and error-prone. So what can you do? You might think about simply rewriting the library function to take a single argument:

```javascript
function simpleURL(path) {
    return "http://www.jackzxl.net/" + path;
}
```

This change has two problems. First, if the library function is also used by other people or in other places, modifying it directly is out of the question. Second, even if you have full control over the function, this approach is inflexible — what if one day you want to enable SSL on your site? You'd have to put the first argument back. The correct choice, therefore, is currying. Currying means: binding a function to a subset of its arguments and returning a new function. If that feels abstract, you can draw an analogy to partial specialization in C++ templates, which makes it easier to grasp. Let's curry the example above:

```javascript
var myURL = simpleURL.bind(null, 'http', 'www.jackzxl.net');
myURL('myfile.js');     //http://www.jackzxl.net/myfile.js

// The site adds SSL
var mySslURL = simpleURL.bind(null, 'https', 'www.jackzxl.net');
mySslURL('myfile.js');  //https://www.jackzxl.net/myfile.js
```

The code above uses `bind` to implement currying. Now revisit the definition of currying: binding a function to a subset of its arguments and returning a new function. After currying, the function becomes more flexible and more fluent — it's a concise way to delegate behavior. Why use `bind` to implement currying? Because it's simple — there's no need to reinvent the wheel when a built-in is already available. But since this post is about currying, let's implement it ourselves to deepen our understanding. It needs to satisfy two requirements: accept a subset of arguments, and return a new function:

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

The effect is the same as using `bind`. Let's examine the custom `currying` function more closely: the first argument, `fn`, is the function to be curried (`simpleURL`), and the rest are variadic arguments (for more on a function's `arguments`, see here). The result of each line in `currying` is as follows:

```javascript
var currying = function(fn) {
    var args = [].slice.call(arguments, 1);
    // args is ["https", "www.jackzxl.net"]

    return function() {
        var newArgs = args.concat([].slice.call(arguments));
        // newArgs is ["https", "www.jackzxl.net", "myFile.js"]

        return fn.apply(null, newArgs);
        // equivalent to return simpleURL("https", "www.jackzxl.net", "myFile.js");
    };
};
```

We've now covered the principle and the implementation of currying. So what is currying actually good for? The common benefits are:

* Parameter reuse
* Deferred execution
* Flattening

`Parameter reuse` was already demonstrated in the example above, so I won't belabor it.

`Deferred execution` is actually very intuitive: because the function returns a new function instead of a computed result, execution is naturally deferred. For example, `bind` is a representative case of deferred execution, so again I won't dwell on it.

`Flattening` makes functions more readable. For instance, suppose you want to extract the titles of all posts from your site's JSON data:

```javascript
// JSON data
{
    "user": "Jack",
    "posts": [
        { "title": "JavaScript Curry", "contents": "..." },
        { "title": " JavaScript Function", "contents": "..." }
    ]
}

// Get the titles of all posts from the JSON data
fetchFromServer()
    .then(JSON.parse)
    .then(function(data){ return data.posts })
    .then(function(posts){
        return posts.map(function(post){ return post.title })
    })
```

Of course you could write more elegant code than this... but that's beside the point. The point is to use currying to make the code more readable and maintainable:

```javascript
var curry = require('curry');
var get = curry(function(property, object){ return object[property] });

fetchFromServer()
    .then(JSON.parse)
    .then(get('posts'))
    .then(map(get('title')))
```

**Early return?**

Finally, another use case you'll find online is early return. For example, IE handles events differently from other browsers; to achieve compatibility you could write:

```javascript
var addEvent = (function(){
    if (target.addEventListener) {
        return function(target, eventType, handler) {
            target.addEventListener(eventType, handler, false);
        };
    } else {        // IE
        return function(target, eventType, handler) {
            target.attachEvent("on" + eventType, handler);
        };
    }
})();  
```

In my view, however, currying doesn't add much value here. Currying certainly has many advantages, but it has equally obvious drawbacks — notably a non-trivial learning curve. Using currying to implement "early return" costs more in maintenance than it gains.

How can you achieve the same thing without currying? A single ternary operator does the trick:

```javascript
var addHandler = document.body.addEventListener ?
    function(target, eventType, handler){
        target.addEventListener(eventType, handler, false);
    } :
    function(target, eventType, handler){
        target.attachEvent("on" + eventType, handler);
    };
```

Or you can rewrite the function from within itself:

```javascript
function addHandler(target, eventType, handler){
    if (target.addEventListener){
        addHandler = function(target, eventType, handler){  // rewrite this function
            target.addEventListener(eventType, handler, false);
        };
    } else {        // IE
        addHandler = function(target, eventType, handler){  // rewrite this function
            target.attachEvent("on" + eventType, handler);
        };
    }
    addHandler(target, eventType, handler); // call the new function
}
```

Both approaches are straightforward and crystal clear. Don't reach for currying just for the sake of using currying.

## Summary

Although currying has a somewhat mysterious name, it isn't really mysterious once you see through it. On the frontend its use cases are limited (though that may just reflect my own limited experience); it's more often applied to asynchronous functions on the backend, such as in Node.js, where currying async APIs can reduce callback nesting.

---
> Source: https://www.jianshu.com/p/9b6b5c7527fc
