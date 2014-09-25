---
layout: post
title: "JavaScript函数上下文环境"
date: 2014-09-25 14:22:25 +0800
comments: true
categories: nodejs
---
## this引用对象

在JavaScript中，上下文对象就是**this**，即被调用函数所处的环境。上下文对象的作用是在一个函数内部引用调用它的对象本身，JavaScript的任何函数都是被某个对象调
用的，包括全局对象（在nodejs中是global对象，在browser中是window对象）。

## call(), apply()

**call**和**apply**的功能是以不同的对象作为上下文来调用某个函数。

## bind()

**bind**方法来永久地绑定函数的上下文，使其无论被谁调用，上下文都是固定的。

**bind**方法返回值：被绑定的函数对象。

##Example
```javascript call.js
var util = require('util');

function foo(a1, a2) {
  if (this === global) {
    console.log('this: glabol');
  }
  else {
    console.log('this: ' + util.inspect(this));
  }
  console.log('args: ' + util.inspect(arguments));
  console.log();
}

foo('A', 'B');

// call(), forward args by args list
foo.call({attr1: 'hello', attr2: 'call'}, 'AA', 'BB');
// apply(), forward args by array
foo.apply({attr1: 'world', attr2: 'apply'}, ['AAA', 'BBB']);

// bind context and arg
f = foo.bind({attr1: 'cannot-be-set', attr2: 'binded'}, 'BINDED_ARG1');
// call binded function, no need to set binded 'this' and args
f('BBBB');

// cannot change binded 'this' by call, additional args will append after the binded args
f.call({attr1: 'try-to-set', attr2: 'f-call-binded'}, 'AAAAA', 'BBBBB');
```

output:

    this: glabol
    args: { '0': 'A', '1': 'B' }

    this: { attr1: 'hello', attr2: 'call' }
    args: { '0': 'AA', '1': 'BB' }

    this: { attr1: 'world', attr2: 'apply' }
    args: { '0': 'AAA', '1': 'BBB' }

    this: { attr1: 'cannot-be-set', attr2: 'binded' }
    args: { '0': 'BINDED_ARG1', '1': 'BBBB' }

    this: { attr1: 'cannot-be-set', attr2: 'binded' }
    args: { '0': 'BINDED_ARG1', '1': 'AAAAA', '2': 'BBBBB' }
