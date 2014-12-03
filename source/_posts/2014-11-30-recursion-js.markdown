---
layout: post
title: "Node.js中递归算法的异步实现"
date: 2014-11-30 13:24:11 +0800
comments: true
categories: nodejs
---

## 问题背景

Node.js最大的特点在于其事件循环和异步回调的编程模式。基于此，很多I/O的处理变得高效，错误处理也很直接。

很多递归算法，在同步语言(如C、Python等)中实现起来很熟悉顺手，但其在node.js类似的异步环境中实现起来思路却完全不同。
尤其是当算法实现中包含文件读取、数据库访问、网络请求等异步调用时。所以总结如下。

本文的目标是研究并解决:

在node.js中实现递归算法的异步版本，及其错误处理。

## 阶乘(Factorial) 与 斐波纳契数列(Fibonacci Sequence)
先从这两个教科书般的案例入手，说明基本原理和实现。
#### 阶乘
version 0: 同步版
```javascript
var fact = function(n) {
  if (n === 0 || n === 1) return n;
  else return n * fact(n-1);
};
```
version 1: 异步版
```javascript
var fact = function(n, cb) {
  if (n === 0 || n === 1) {
    cb(n);
  }
  else {
    fact(n-1, function(m) {
      cb(n * m);
    });
  }
};
```
version 2: 加入错误处理
```javascript fact.js
var assert = require('assert');

var fact = function(n, cb) {
  var limit = 120;
  var _fact = function(err, n, cb) {
    assert.strictEqual(err, null);
    if (n === 0 || n === 1) cb(null, n);
    else {
      _fact(null, n-1, function(err, m) {
        if (err) cb(err);
        else if (n * m > limit) cb(new Error('Exceed limit: n = ' + n));
        else cb(null, n * m);
      });
    }
  };
  _fact(null, n, function(err, result) {
    if (err) cb(err);
    else cb(null, result);
  });
};

var n = +process.argv[2];
fact(n, function(err, result) {
  if (err) console.log(err);
  else console.log(result);
});
```
注意两点：

1. 调用递归函数时，第一个参数err必须是null(第16, 9行);

2. 在调用自己时的回调处理中回调错误，并向上传递(第11行);

测试结果:

    $ node fact.js 5
    120
    $ node fact.js 10
    [Error: Exceed limit: n = 6]

#### 斐波纳契数列
初级版
```javascript
var fib = function(n, cb) {
  var limit = 55;
  var _fib = function(err, n, cb) {
    assert.strictEqual(err, null);
    if (n === 0 || n === 1) cb(null, n);
    else {
      _fib(null, n-1, function(err, m1) {
          if (err) cb(err);
          else if (m1 > limit) cb(new Error('Execced limit: n = ' + n));
          else {
          _fib(null, n-2, function(err, m2) {
            if (err) cb(err);
            else if (m2 > limit) cb(new Error('Execced limit: n = ' + n));
            else cb(null, m1 + m2);
            });
          }
          });
    }
  };
  _fib(null, n, function(err, result) {
    if (err) cb(err);
    else cb(null, result);
  });
};
```
这样做的问题显而易见: 在每个递归调用的回调中都需要写重复的错误处理逻辑。因此需要改写为以下版本:

优化版
```javascript fib.js
var assert = require('assert');
var async = require('async');
var _ = require('underscore');

var fib = function(n, cb) {
  var limit = 4.346655768693743e+208; // fib(1000)
  var memo = {};
  var _fib = function(err, n, cb) {
    assert.strictEqual(err, null);
    if (n === 0 || n === 1) cb(null, n);
    else if (memo[n]) {
      cb(null, memo[n]);
    }
    else {
      var a = [n-1, n-2];
      async.map(a,
        function(nn, cbk) {
          _fib(null, nn, function(err, m) {
            if (err) cbk(err);
            else if (m > limit) cbk(new Error('Execced limit: n = ' + n));
            else cbk(null, m);
          });
        },
        function(err, results) {
          if (err) cb(err);
          else {
            var sum = _.reduce(results, function(m, n){return m + n}, 0);
            memo[n] = sum;
            cb(null, sum);
          }
        });
    }
  };
  _fib(null, n, function(err, result) {
    if (err) cb(err);
    else cb(null, result);
  });
};

var n = +process.argv[2];
fib(n, function(err, result) {
  if (err) console.log(err);
  else console.log(result);
});
```
1. 利用async.map避免了多次自我调用，各自返回结果后\_.reduce求和;

2. 利用memo的技术，缓存已经有的结果;

当然，不论是阶乘还是斐波纳契数列，实际求值时都不会用递归的方法实现的，因为这两个问题都有更高效的非递归的实现方法。所以以上两个仅用于说明基本原理。

## 深度优先搜索(Depth-first search)
图相关的算法中很多都离不开递归，以深度优先搜索为例实现。

```javascript dfs.js
var assert = require('assert');
var async = require('async');
var _ = require('underscore');

var typeChartAttack = {
  '普': [],
  '火': ['草', '冰', '虫', '钢'],
  '水': ['火', '地', '岩'],
  '电': ['水', '飞'],
  '草': ['水', '地', '岩'],
  '冰': ['草', '地', '飞', '龙'],
  '超': ['格', '毒'],
  '格': ['普', '冰', '岩', '恶', '钢'],
  '毒': ['草', '妖'],
  '地': ['火', '电', '毒','岩', '钢'],
  '飞': ['草', '格', '虫'],
  '虫': ['草', '超', '恶'],
  '岩': ['火', '冰', '飞', '虫'],
  '鬼': ['超', '鬼'],
  '龙': ['龙'],
  '恶': ['超', '鬼'],
  '钢': ['冰', '岩', '妖'],
  '妖': ['格', '龙', '恶']
};

var traverse = function(graph, start, cb) {
  var traversedNodes = [];
  var _traverse = function(err, root, cbk) {
    assert.strictEqual(err, null);
    if (!graph[root]) {
      return cbk(new Error('start point not found in graph!'));
    }
    traversedNodes.push(root);
    var leaves = graph[root];
    async.map(leaves,
      function(leaf, callback) {
        if (!_.contains(traversedNodes, leaf)) {
          _traverse(null, leaf, function(err) {
            if (err) callback(err);
            else callback(null);
          });
        }
        else {
          callback(null);
        }
      },
      function(err) {
        if (err) cbk(err);
        else cbk(null);
      });
  };
  _traverse(null, start, function(err) {
    if (err) cb(err);
    else cb(null, traversedNodes);
  });
};

var start = process.argv[2];
traverse(typeChartAttack, start, function(err, results) {
  if (err) console.log(err);
  else {
    results.forEach(function(v) {
      process.stdout.write(v + ' ');
    });
    console.log();
  }
});
```

至于图的意义，口袋妖怪玩家都懂的，或者看这里科普http://pokemondb.net/type
#### 测试结果

    $ node dfs.js 虫
    虫 草 水 火 冰 地 电 飞 格 普 岩 恶 超 毒 妖 龙 鬼 钢
    $ node dfs.js 恶
    恶 超 格 普 冰 草 水 火 虫 钢 岩 飞 妖 龙 地 电 毒 鬼
    $ node dfs.js 龙
    龙

## 应用案例: 文件包含

#### 问题描述
找出一个文件所包含引用的其他所有文件。

若一个文件包含其他文件，其内容为:

``` c file A
include B
include C
```

问题本质: 图遍历。

``` javascript include.js
var assert = require('assert');
var fs = require('fs');
var _ = require('underscore');
var async = require('async');

var include = function(start, cb) {
  var results = [];
  var _leaves = function(node, cb) {
    fs.readFile(node, 'utf-8', function(err, data) {
      if (err) cb(err);
      else {
        var leaves = [];
        var lines = data.split('\n');
        var pattern = /^include\s+(.*)$/;
        lines.forEach(function(line) {
          var match = line.match(pattern);
          if (match) {
            var node = match[1];
            leaves.push(match[1]);
          }
        });
        cb(null, leaves);
      }
    });
  };
  var _include = function(err, node, cb) {
    assert.strictEqual(err, null);
    results.push(node);
    _leaves(node, function(err, leaves) {
      if (err) {
        console.log('find leaves error: ' + err);
        cb(err);
      }
      else {
        async.map(leaves,
          function(leaf, cbk) {
            if(!_.contains(results, leaf)) {
              _include(null, leaf, function(err) {
                if (err) cbk(err);
                else cbk(null);
              });
            }
            else cbk(null);
          },
          function(err) {
            if (err) cb(err);
            else cb(null);
          });
      }
    });
  };

  _include(null, start, function(err) {
    if (err) cb(err);
    else cb(null, results);
  });
};

var start = process.argv[2];
include(start, function(err, results) {
  if (err) console.log(err);
  else console.log(results);
});
```

#### 测试实例

    A -> B, C
    B -> C, D
    C -> D
    E -> F
    F -> C

####运行结果

    $ node include.js A
    [ 'A', 'B', 'C', 'D' ]
    $ node include.js B
    [ 'B', 'C', 'D' ]
    $ node include.js E
    [ 'E', 'F', 'C', 'D' ]

    $ mv C CC
    $ node include.js E
    find leaves error: Error: ENOENT, open 'C'
    { [Error: ENOENT, open 'C'] errno: 34, code: 'ENOENT', path: 'C' }

当文件读取失败时，可以看到错误被一层层传递到最上。

而异步回调的好处就在于，可以根据错误类型选择不同的处理方式。
如当文件不存在时，只需要无视该包含文件的内容，而不是让整个程序出错，
而出现其他错误是仍然将错误向上传递，则可以按以下方式修改。


``` javascript
  var _include = function(err, node, cb) {
    assert.strictEqual(err, null);
    results.push(node);
    _leaves(node, function(err, leaves) {
      if (err) {
        console.log('find leaves error: ' + err);
        //cb(err);
        if (err.code === 'ENOENT') cb(null);
        else cb(err);
      }
      else {
        async.map(leaves,
          function(leaf, cbk) {
            if(!_.contains(results, leaf)) {
              _include(null, leaf, function(err) {
                if (err) cbk(err);
                else cbk(null);
              });
            }
            else cbk(null);
          },
          function(err) {
            if (err) cb(err);
            else cb(null);
          });
      }
    });
  };
```
    $ node include.js E
    find leaves error: Error: ENOENT, open 'C'
    [ 'E', 'F', 'C' ]

    $ mv CC C
    $ chmod -r D
    $ node include.js E
    find leaves error: Error: EACCES, open 'D'
    { [Error: EACCES, open 'D'] errno: 3, code: 'EACCES', path: 'D' }


所以，

1. 尽量不要用同步的方法，像readFileSync之类的，错误不好处理，原因如下:

同步函数的错误处理只能用try/catch块，一旦抛出，直接传到最上层，完全无法根据具体情况在当前函数或上一层闭包中处理。

2. 异步的错误处理要比同步的灵活稳定很多。

### 总结
至此，nodejs中递归函数的写法及其异步回调和错误处理的原理和实现基本清晰了。
