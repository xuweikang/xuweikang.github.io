---
layout: post
title: 拓展一个构造函数
tags: 构造函数 extend
categories: Practice
---

* TOC 
{:toc}

### 如何使用proxy扩展一个构造函数

效果如下：
```js
var Person = function (name) {
  this.name = name
};
Person.prototype.height = 180
var Boy = extend(Person, function (name, age) {
  this.age = age;
});

Boy.prototype.sex = "M";

var Peter = new Boy("Peter", 13);
console.log(Peter.sex);  // "M"
console.log(Peter.name); // "Peter"
console.log(Peter.age);  // 13
console.log(Peter.height);  // 13

```

那么extend函数内部如何实现？
```js
function extend(sup, base) {
  // 获取base的原型对象上的构造函数constructor描述信息
  let descriptor = Object.getOwnPropertyDescriptor(base.prototype, 'constructor')
  // 将base的原型对象的__proto__指向sup的原型
  // 构成原型链，方便查找到sup上的原型数据
  base.prototype = Object.create(sup.prototype)
  const handle = {
  // 拦截 new 
    construct(target, args) {
      const obj = Object.create(base.prototype)
      // 将sup 和 base内的this指向obj，也就是Peter
      this.apply(target, obj, args)
      return obj
    },
    apply(target, that, args) {
      sup.apply(that, args)
      base.apply(that, args)
    }
  }
  const proxy = new Proxy(base, handle)
  // 描述对象value赋值
  descriptor.value = proxy
  // 修改base.prototype.constructor的值，使其指向proxy
  // 构成原型关系指向闭环
  Object.defineProperty(base.prototype, 'constructor', descriptor)
  return proxy
}

```