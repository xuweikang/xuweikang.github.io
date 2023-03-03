---
layout: post
title: call、apply和bind的模拟实现
tags: apply call bind
categories: Practice
---

* TOC 
{:toc}

#### 1. call和apply

​	call方法和apply方法，实现一个就行了，两个方法除了第二个参数以外，其他完全一样。

​	apply方法第二个参数传入的是个参数数组。



> call()方法在使用一个指定的this值和若干个指定的参数的前提下调用某个函数

可以看下面这个例子：

```javascript
var foo = {
  value: 1
}
function bar() {
  console.log(this.value)
}
bar.call(foo) // 1
```

#### call的模拟实现

```javascript
Function.prototype.call2 = function (context,...args) {
  // this 参数可以传基本类型数据，原生的 call 会自动用 Object() 转换
  context = context ? Object(context) : window
  context.fn = this
  // 用es6解构的方法如下
  // var result = context.fn(...(args || []))
  var args = [] // 模拟参数列表
  for (var i = 1; i < arguments.length; i ++) {
    args.push('arguments['+i+']')
  }
  var result = eval('context.fn('+args+')') // eval方法会把数组转成 .toString()
  delete context.fn
  return result
}
```

注意：上述实现方案，其实还有一个问题，就是`context.fn` 这里面，确保context里面没有fn属性，不然就会有问题。

判断是否有某个属性并重新赋值，有如下3个方案：

方案1:

```javascript
function fnFactory(contetxt) {
  var unique_fn = 'fn'
  while (contetxt.hasOwnProperty(unique_fn)) {
    unique_fn += Math.random()
  }
  return unique_fn
}
```

方案2:

```js
function fnFactory(contetxt) {
  var unique_fn = 'fn' + Math.random()
  if (context.hasOwnProperty(unique_fn)) {
    return fnFactory(context)
  } else {
    return unique_fn
  }
}
```

方案3:

方案1和方案2都是es3下的模拟方案，es6下可以直接使用Symbol() 进行实现。

`var fn = Symbol()`

#### apply的模拟实现

```javascript
Function.prototype.apply2 = function (context,args) {
  context = context ? Object(context) : window	
  var fn = Symbol()
  context[fn] = this
  // 用es6解构的方法如下
  var result = context.fn(...(args || []))
  delete context.fn
  return result
}
```



#### 2. bind

>  bind方法会创建一个新函数。当这个函数被调用的时候，第一个桉树将会作为它运行时的this值，之后一系列参数将会在作为它的参数。

可以看下面这个例子：

```javascript
var foo = {
  value: 1
}
function bar(a,b,c) {
  console.log(this.value,a,b,c)
}
var bindFoo = bar.bind(foo,1,2) 
bindFoo(3) // 1 1 2 3
```

#### 版本一：

```javascript
Function.prototype.bind2 = function (context){
  const self = this
  const args1 = Array.prototype.slice.call(arguments,1)
  return function () {
    return self.apply(context,args1.concat(Array.prototype.slice.call(arguments)))
  }
}
```

bind方法还有一个特点，就是

> 一个绑定函数也能使用new操作符创建对象：这种行为就像把原函数当作构造器。提供的this值被忽略。同时调用时的参数被提供给模拟函数。

也就是说，当bind返回的函数作为构造函数用的时候，bind时指定的this值会失效，但传入的参数依然有效。

举个例子：

```js
var value = 2;
var foo = {
    value: 1
};
function bar(name, age) {
    this.habit = 'shopping';
    console.log(this.value);
    console.log(name);
    console.log(age);
}
bar.prototype.friend = 'kevin';

var bindFoo = bar.bind(foo, 'Jack');
var obj = new bindFoo(20);
// undefined
// Jack
// 20

obj.habit;
// shopping

obj.friend;
// kevin
```

#### 版本二：

```javascript
Function.prototype.bind2 = function(context, ...args1) {
    if (typeof this !== 'function') {
        throw new Error('error')
    }
    const self = this
    const FA = function () {}
    const FB = function (...args2) {
        return self.apply(this instanceof FA ? this : context, [...args1, ...args2])
    }
    FA.prototype = this.prototype	
    FB.prototype = new FA()
    // 这里也可以用Object.create
    // FB.prototype = Object.create(this.prototype)
    return FB
}
```

#### 3. new

> new运算符创建一个用户定义的对象类型的实例或具有构造函数的内置对象类型之一

举个例子：

``` javascript
function Animal (name) {
  this.name = name 
  this.isPerson = false
}
Animal.prototype.age = 3
Animal.prototype.sayHi = function () {
  console.log('hi')
}
var dog = new Animal('dog')
console.log(dog.name) // dog
console.log(dog.isPerson)  // false
console.log(dog.age)  // 3
dog.sayHi()  // hi
```

从这个例子中，可以看出，被new过的Animal，实例dog可以：

1.  访问到构造函数Animal里面的属性
2. 访问到Animal.prototype里的属性

我们用NewFactory这个函数来模拟new的行为，第一个参数是构造函数，第二个参数是一系列参数

#### 第一版：

```javascript
function NewFactory() {
  var obj = new Object()
  const Constructor = [].shift.call(arguments)
  obj.__proto__ = Constructor.prototype  // 使obj能够访问到构造函数原型上的属性
  Constructor.apply(obj, arguments) // 使obj能够访问构造函数内部的属性
  return obj
}
```

上述代码没考虑到一个问题，那就是，在使用new 构造函数的时候，如果构造函数内部有返回值：

- 返回了一个对象，那么内部属性将失效，实例只能访问到return的对象里面的属性值
- 返回了一个基本类型值，实例访问不受return影响

#### 第二版：

```javascript
function NewFactory() {
  var obj = new Object()
  const Constructor = [].shift.call(arguments)
  obj.__proto__ = Constructor.prototype  // 使obj能够访问到构造函数原型上的属性
  const ret = Constructor.apply(obj, arguments) // 使obj能够访问构造函数内部的属性
  return typeof ret === 'object' ? (ret || obj) : obj
}
```



#### 最终版：

```js
function NewFactory() {
  const Constructor = [].shift.call(arguments)
  var obj = Object.create(Con.prototype); 	// 链接到原型，obj 可以访问到构造函数原型中的属性
  const ret = Constructor.apply(obj, arguments) // 使obj能够访问构造函数内部的属性
  return typeof ret === 'object' ? (ret || obj) : obj
}
```



```
function fn1(){
   console.log(1);
}
function fn2(){
    console.log(2);
}

fn1.call(fn2);     //输出 1
 
fn1.call.call(fn2);  //输出 2
```

实现一个无限累加的函数add：

add(1)(2)(3)  // 6

add(1,2)(3)  // 6

```javascript
function add() {
    var args = [...arguments]
    var fn = function() {
        args.push(...arguments)
        return fn
    }
    fn.toString = function() {
        return args.reduce((a,b) => a+b, 0)
    }
    return fn
}
```

```javascript
function add() {
    var args = [...arguments]
    var fn = function() {
        args.push(...arguments)
        return fn
    }
    setTimeout(() => {
        console.log(args.reduce((a,b) => a+b, 0))
    },0)
    return fn
}
```