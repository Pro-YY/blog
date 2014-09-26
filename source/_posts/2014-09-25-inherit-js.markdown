---
layout: post
title: "Node.js中实现类的继承"
date: 2014-09-25 20:07:52 +0800
comments: true
categories: nodejs
---

## Class
```javascript device.js
var EventEmitter = require('events').EventEmitter;
var util = require('util');

function Device (name) {
  this.name = name;
}

util.inherits(Device, EventEmitter);

Device.prototype.powerOn = function () {
  console.log('Device: ' + this.name + ' is powered on');
}

module.exports = Device;
```

## Subclass
```javascript phone.js
var Device = require('./device');
var util = require('util');

/* override/customize parent constructor */
function Phone (name, conn) {
  Device.apply(this, arguments);
  this.conn = conn; 
}

util.inherits(Phone, Device);

/* override/customize parent method */
Phone.prototype.powerOn = function () {
  Device.prototype.powerOn.apply(this, arguments);
  console.log('Phone: ' + this.name + ' is powered on');
}

/* new method of susbclass */
Phone.prototype.call = function () {
  console.log(this.name + ' maked a ' + this.conn + ' call');
}

/* customize ancester method */
Phone.prototype.on('incoming', function (who) {
  console.log('Phone: ' + this.name + ' got a call from ' + who);
});

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
console.log('---------------------------------');

var p1 = new Phone('Redmi 1S', 'GSM');
var p2 = new Phone('Mi 3', 'WCDMA');
var p3 = new Phone('Mi 4', 'FDD-LTE');

p1.powerOn();
p1.call();

p2.powerOn();
p2.call();

p3.powerOn();
p3.call();
console.log('---------------------------------');

setTimeout(function () {
  console.log('\n3 seconds later...');
  p1.emit('incoming', 'Alice');
  p2.emit('incoming', 'Bob');
  p3.emit('incoming', 'Wendy');
  p3.emit('myincoming', 'Wendy');
}, 3000);

/* add additional event listener*/
p3.on('myincoming', function (friend) {
  console.log('My Mi 4 Got A Call From: ' + friend);
});
```

    Device: Mi Box is powered on
    Device: Mi TV is powered on
    ---------------------------------
    Device: Redmi 1S is powered on
    Phone: Redmi 1S is powered on
    Redmi 1S maked a GSM call
    Device: Mi 3 is powered on
    Phone: Mi 3 is powered on
    Mi 3 maked a WCDMA call
    Device: Mi 4 is powered on
    Phone: Mi 4 is powered on
    Mi 4 maked a FDD-LTE call
    ---------------------------------

    3 seconds later...
    Phone: Redmi 1S got a call from Alice
    Phone: Mi 3 got a call from Bob
    Phone: Mi 4 got a call from Wendy
    My Mi 4 Got A Call From: Wendy

