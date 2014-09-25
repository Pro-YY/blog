---
layout: post
title: "Node.js中循环与回调函数的陷阱"
date: 2014-09-24 20:22:33 +0800
comments: true
categories: nodejs
---

##问题描述

访问3个url，按顺序输出HTTP响应。先后做出以下两种实现：

###Version 1 (nearly official solution, worked of course)

``` javascript 09-juggling-async/09-juggling-async.js
var http = require('http');
var bl = require('bl');
var util = require('util');

var result = {};

function printResult() {
  for (var i = 0; i < 3; i++) {
    console.log(result[process.argv[2+i]]);
  }
}

function get(i) {
  http.get(process.argv[2+i], function(res) {
    console.log('requested url: ' + res.headers.location);
    res.pipe(bl(function(err, data) {
      result[process.argv[2+i]] = data.toString();
      console.log('result length:' + Object.keys(result).length);
      if (Object.keys(result).length === 3) {
        console.log('---- GOT ALL ----\n');
        // printResult();
      }
    }));
  }).on('error', console.error);
}

for (var i = 0; i < 3; i++) {
  get(i);
}
```

    $ js 09-juggling-async/09-juggling-async.js  http://mi.com/1 http://mi.com/2 http://mi.com/3
    requested url: http://www.mi.com/3
    result length:1
    requested url: http://www.mi.com/1
    result length:2
    requested url: http://www.mi.com/2
    result length:3
    ---- GOT ALL ----

###Version 2 (WRONG implementation)
``` javascript 09-juggling-async/09-juggling-async.js
for (var i = 0; i < 3; i++) {
  http.get(process.argv[2+i], function(res) {
    console.log('requested url: ' + res.headers.location);
    res.pipe(bl(function(err, data) {
      result[process.argv[2+i]] = data.toString();
      console.log('result length:' + Object.keys(result).length);
      if (Object.keys(result).length === 3) {
        console.log('---- GOT ALL ----\n');
        // printResult();
      }
    }));
  }).on('error', console.error);
}
```

output:

    requested url: http://www.mi.com/1
    result length:1
    requested url: http://www.mi.com/3
    result length:1
    requested url: http://www.mi.com/2
    result length:1

## 错误分析

看似类似的函数，结果完全不同，其中第二种实现是错误的。加入debug，输出每次执行的i值，清楚了错误原因。

#### debug it
``` javascript 09-juggling-async/09-juggling-async.js
for (var i = 0; i < 3; i++) {
  http.get(process.argv[2+i], function(res) {
    console.log('i: ' + i);
    console.log('requested url: ' + res.headers.location);
    res.pipe(bl(function(err, data) {
      result[process.argv[2+i]] = data.toString();
      console.log('result length:' + Object.keys(result).length);
      if (Object.keys(result).length === 3) {
        console.log('---- GOT ALL ----\n');
        // printResult();
      }
    }));
  }).on('error', console.error);
}
```

output:

    i: 3
    requested url: http://www.mi.com/3
    result length:1
    i: 3
    requested url: http://www.mi.com/1
    result length:1
    i: 3
    requested url: http://www.mi.com/2
    result length:1

其错误在于：回调函数执行时，循环已经结束，i值已经等于3，接着，回调函数将响应存在了同一个地方result[process.argv[3]]。

## 解决方法

通过函数（匿名函数或闭包）等方式，使得回调函数保留对i的引用，不释放即可。

###Version 3: Closure (worked as expected)
``` javascript 09-juggling-async/09-juggling-async.js
for (var i = 0; i < 3; i++) {
  (function(i) {
  http.get(process.argv[2+i], function(res) {
    console.log('requested url: ' + res.headers.location);
    res.pipe(bl(function(err, data) {
      result[process.argv[2+i]] = data.toString();
      console.log('result length:' + Object.keys(result).length);
      if (Object.keys(result).length === 3) {
        console.log('---- GOT ALL ----\n');
        // printResult();
      }
    }));
  }).on('error', console.error);
  })(i);
}
```

###Version 4: forEach traversal (worked too)
``` javascript 09-juggling-async/09-juggling-async.js
[0, 1, 2].forEach(function (i) {
  http.get(process.argv[2+i], function(res) {
    console.log('requested url: ' + res.headers.location);
    res.pipe(bl(function(err, data) {
      result[process.argv[2+i]] = data.toString();
      console.log('result length:' + Object.keys(result).length);
      if (Object.keys(result).length === 3) {
        console.log('---- GOT ALL ----\n');
        // printResult();
      }
    }));
  }).on('error', console.error);
});
```
