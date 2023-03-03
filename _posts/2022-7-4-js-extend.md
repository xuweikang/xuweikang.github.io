---
layout: post
title: js实现继承的多种方式
tags: js 原型 对象 继承
categories: jsBase
---

* TOC 
{:toc}

#### 1. 原型链继承

```javascript
function Parent() {
  this.name = 'xwk'
}
Parent.prototype.getName = function() {
  console.log(this.name)
}
function Child() {}
Child.prototype = new Parent()
var child = new Child()
console.log(child.getName()) // xwk
```

缺点：

1. 引用类型的属性被所有实例共享，举个例子：

```javascript
function Parent () {
    this.names = ['kevin', 'daisy'];
}
function Child () {}
Child.prototype = new Parent();
var child1 = new Child();
child1.names.push('yayu');
console.log(child1.names); // ["kevin", "daisy", "yayu"]
var child2 = new Child();
console.log(child2.names); // ["kevin", "daisy", "yayu"]
```

2. 在创建Child的实例时，不能向Parent传参数
3. 实例丢失了自己的construct属性

#### 2. 经典继承（借用构造函数（使用call））

```javascript
function Parent() {
  this.names = ["kevin", "daisy"]
}
Parent.prototype.getName = function() {
  console.log(this.names)
}
function Child() {
  Parent.call(this)
}
var child1 = new Child()
child1.names.push('yayu');
console.log(child1.names); // ["kevin", "daisy", "yayu"]
var child2 = new Child();
console.log(child2.names); // ["kevin", "daisy"]
```

缺点：Parent原型上的属性和方法不能被继承

优点：

1.  在继承的时候可以向Parent传参
2.  可以避免引用类型的属性被不同实例所共享

#### 3. 组合继承

原型链继承 + 经典继承

```javascript
function Parent(name) {
  this.name = name
  this.colors = ['red','blue']
}
Parent.prototype.getName = function() {
  console.log(this.name)
}
function Child(name, age) {
  Parent.call(this, name)
  this.age = age
}
Child.prototype = new Parent()
Child.prototype.constructor = Child

var child1 = new Child('kevin', '18');

child1.colors.push('black');

console.log(child1.name); // kevin
console.log(child1.age); // 18
console.log(child1.colors); // ["red", "blue", "green", "black"]

var child2 = new Child('daisy', '20');

console.log(child2.name); // daisy
console.log(child2.age); // 20
console.log(child2.colors); // ["red", "blue", "green"]
```

分析下上述代码：

1.  Parent.call(this, name) ，解决了传参问题，并且将Parent构造函数内的属性复制到Child里，可以避免引用类型被共享；
2.  Child.prototype = new Parent() 同时使用原型链继承，可以保证Parent原型上的属性和方法能被Child继承到。



#### 4. 原型式继承 （Object.create）

```javascript
function createObj(obj) {
  function F() {}
  F.prototype = obj
  return new F()
}
```

上述代码，其实就是ES5 ```Object.create``` 方法的实现，将传入的对象作为一个新对象的原型返回。

缺点：和原型链继承的缺点一样，引用类型的属性会被子实例所共享

```javascript
var person = {
    name: 'kevin',
    friends: ['daisy', 'kelly']
}

var person1 = createObj(person);
var person2 = createObj(person);

person1.name = 'person1';
console.log(person2.name); // kevin

person1.friends.push('taylor');
console.log(person2.friends); // ["daisy", "kelly", "taylor"]
```

#### 5. 寄生式继承

创建一个仅用于封装继承过程的函数，该函数在内部以某种形式来增强对象，最后返回对象。

```javascript
function createObj (o) {
    var clone = Object.create(o);
    clone.sayName = function () {
        console.log('hi');
    }
    return clone;
}
```

缺点：和经典模式一样，方法都在构造函数中定义，每次创建实例都会创建一遍方法。

#### 6. 寄生组合继承

其实就是对组合继承的优化，

我们可以看组合继承的代码，发现一共掉了两次Parent构造函数，

一次是Child.prototype = new Parent()，

一次是Child构造函数中，Parent.call(this，name)，

这样导致的结果就是Child和Child.prototype中都有colors属性。

那么怎么优化呢，避免这一次重复调用呢？

如果我们不使用 Child.prototype = new Parent() ，而是间接的让 Child.prototype 访问到 Parent.prototype 呢？

```javascript
function Parent(name) {
  this.name = name
  this.colors = ['red','blue']
}
Parent.prototype.getName = function() {
  console.log(this.name)
}
function Child(name, age) {
  Parent.call(this, name)
  this.age = age
}
Child.prototype = Object.create(Parent.prototype)
Child.prototype.constructor = Child

var child1 = new Child('kevin', '18');

child1.colors.push('black');

console.log(child1.name); // kevin
console.log(child1.age); // 18
console.log(child1.colors); // ["red", "blue", "green", "black"]

var child2 = new Child('daisy', '20');

console.log(child2.name); // daisy
console.log(child2.age); // 20
console.log(child2.colors); // ["red", "blue", "green"]
```

注意⚠️：

使用 ```Child.prototype = Object.create(Parent.prototype)  ``` 替换

```Child.prototype = new Parent()```

虽然目的都是一样，让Child.prototype 的原型对象 指向 Parent.prototype，

但是使用后者会把Parent构造函数内部的多余属性也继承过来，前者不会。