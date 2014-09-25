---
layout: post
title: "Nodejs中实现类的继承"
date: 2014-09-25 20:07:52 +0800
comments: true
categories: nodejs
---

## Class
```javascript device.js
function Device (name) {
  this.name = name;
}

Device.prototype.powerOn = function () {
  console.log(this.name + ' is powered on');
}

Device.prototype.getName = function () {
  return this.name;
}

module.exports = Device;
```

## Subclass
```javascript phone.js
var Device = require('./device');
var util = require('util');

function Phone (name, conn) {
  Phone.super_.apply(this, arguments);
  this.conn = conn; 
}

util.inherits(Phone, Device);

Phone.prototype.getConn = function() {
  return this.conn;
}

Phone.prototype.call = function() {
  console.log(this.name + ' maked a ' + this.conn + ' call');
}

module.exports = Phone;
```

## Run
```javascript index.js
var Device = require('./device');
var Phone = require('./phone');

var d1 = new Device('Mi Box');
var d2 = new Device('Mi TV');

d1.powerOn();
d2.powerOn();

var p1 = new Phone('Redmi 1S', 'GSM');
var p2 = new Phone('Mi 3', 'WCDMA');

p1.powerOn();
p1.call();

p2.powerOn();
p2.call();

console.log(p1 instanceof Phone);
console.log(p1 instanceof Device);
console.log(Phone.super_ === Device);
```

    Mi Box is powered on
    Mi TV is powered on
    Redmi 1S is powered on
    Redmi 1S maked a GSM call
    Mi 3 is powered on
    Mi 3 maked a WCDMA call
    true
    true
    true
