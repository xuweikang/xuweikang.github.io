---
layout: post
title: JS的原型和原型链究竟是什么？
tags: js 原型 对象
categories: jsBase
---

* TOC 
{:toc}


### 1. 从JS创建一个对象开始说起：

####  1.1 工厂模式创建对象  （缺点是无法知道创建出来的对象是一个什么类型的对象）

```javascript
function createPerson(name, age, job) { 
    let o = new Object(); 
    o.name = name; 
    o.age = age; 
    o.job = job; 
    o.sayName = function() { 
        console.log(this.name); 
    }; 
    return o; 
   } 
let person1 = createPerson("Nicholas", 29, "Software Engineer"); 
let person2 = createPerson("Greg", 27, "Doctor");
```

####  1.2  构造函数创建对象  

```javascript
function Person(name, age, job){ 
    this.name = name; 
    this.age = age; 
    this.job = job; 
    this.sayName = function() { 
        console.log(this.name); 
    }; 
} 
let person1 = new Person("Nicholas", 29, "Software Engineer"); 
let person2 = new Person("Greg", 27, "Doctor"); 
person1.sayName(); // Nicholas 
person2.sayName(); // Greg 
console.log(person1.constructor == Person); // true 
console.log(person2.constructor == Person); // true
console.log(person1 instanceof Person); // true 
console.log(person2 instanceof Person); // true

console.log(person1.sayName == person2.sayName); // false
```

###### 优点：没有显示创建对象，只有在new 的时候在内存里创建一个新对象；person1 和person2都是Person的实例，相比于工厂模式，定义自定义构造函数可以确保实例被标识为特定类型。

###### 缺点：sayName方法在每次创建实例的时候都需要实例化一遍

```javascript
// 可以这样解决上面的缺点问题，但是这样做的后果，就是污染了全局作用域
function Person(name, age, job){ 
 this.name = name; 
 this.age = age; 
 this.job = job; 
this.sayName = sayName; 
} 
function sayName() { 
console.log(this.name); 
} 
let person1 = new Person("Nicholas", 29, "Software Engineer"); 
let person2 = new Person("Greg", 27, "Doctor"); 
person1.sayName(); // Nicholas 
person2.sayName(); // Greg
```


### 2. 原型模式

###### 每个函数都会创建一个prototype属性，这个是属性一个对象，是通过构造函数创建的对象的原型。使用原型对象的好处是，在它上面定义的属性和方法可以被对象实例共享

```javascript
function Person() {} 
Person.prototype.name = "Nicholas"; 
Person.prototype.age = 29; 
Person.prototype.job = "Software Engineer"; 
Person.prototype.sayName = function() { 
  console.log(this.name); 
}; 
let person1 = new Person(); 
person1.sayName(); // "Nicholas" 
let person2 = new Person(); 
person2.sayName(); // "Nicholas" 


console.log(person1.sayName == person2.sayName); // true

console.log(Person.prototype.__proto__)   // Object.prototype
console.log(Person.prototype.__proto__.constructor)  // Object
console.log(Person.prototype.__proto__.__proto__)  // null


```

####  2.1  理解原型

###### js在任何情况下，只要创建一个函数就会按照特定的规则为这个函数创建一个 prototype 属性（指向原型对象），所有原型对象默认会自动获得一个constructor属性，这个属性指回构造函数。理解原型的根本就是理解原型对象。

```javascript
Person.prototype.constructor = Person
```

构造函数和原型对象之间可以用prototype和constructor来进行指向的，那实例和原型对象通过什么联系的呢。每个构造函数的实例的内部都有一个[[Prototype]]指针，这个指针被赋值为构造函数的原型对象，js中没有访问这个[[Prototype]]特性的标准方式，有些浏览器厂商会在每个对象上暴露__proto__属性，通过这个属性可以访问对象的原型。
**注意：实例与构造函数原型之间有直接的联系，但实例与构造函数之间没有**

```javascript
function Person() {} 
/** 
 * 声明之后，构造函数就有了一个
 * 与之关联的原型对象：
 */ 
console.log(typeof Person.prototype); 
console.log(Person.prototype);

```

 实例通过"__proto__"链接到原型对象，它实际上指向隐藏特性[[Prototype]],构造函数通过 prototype 属性链接到原型对象。同一个构造函数创建的两个实例共享同一个原型对象。

```
  console.log(person1.__proto__ === Person.prototype); // true 
conosle.log(person1.__proto__.constructor === Person); // true

/** 
 * 同一个构造函数创建的两个实例
 * 共享同一个原型对象：
 */ 
console.log(person1.__proto__ === person2.__proto__); // true
```

<img src="https://i.ibb.co/kKSgqMj/08-13-3.png" style="width: 400px;height:400px" />

### 3. 原型链继承

###### js实现继承只能支持**实现继承**，不支持**接口继承**，这些因为js函数没有签名

```
function SuperType() { 
 this.property = true; 
} 
SuperType.prototype.getSuperValue = function() { 
 return this.property; 
}; 
function SubType() { 
 this.subproperty = false; 
} 
// 继承 SuperType 
let n 
SubType.prototype = new SuperType(); 
SubType.prototype.getSubValue = function () { 
 return this.subproperty; 
}; 
let instance = new SubType(); 
console.log(instance.getSuperValue()); // true
```








### 4. Funtion.__proto__ === Function.prototype
>我们知道，```Funtion.__proto__ === Function.prototype```是成立的，那么就有了经典的“鸡生蛋还是蛋生鸡的问题”。

Function.prototype 对象是一个函数（对象），其 [[Prototype]] 内部属性值指向内建对象 Object.prototype。Function.prototype 对象自身没有 valueOf 属性，其从 Object.prototype 对象继承了valueOf 属性。

###  **“所有的函数都有prototype属性”。这句话其实是错误的。Function.prototype就是一个函数，但是这个函数并没有prototype属性。**
```js
let o = {a: 1};
// 原型链:	o ---> Object.prototype ---> null

let a = ["yo", "whadup", "?"];
// 原型链:	a ---> Array.prototype ---> Object.prototype ---> null

function f(){
  return 2;
}
// 原型链:	f ---> Function.prototype ---> Object.prototype ---> null

let fun = new Function();
// 原型链:	fun ---> Function.prototype ---> Object.prototype ---> null

function Foo() {}
let foo = new Foo();
// 原型链:	foo ---> Foo.prototype ---> Object.prototype ---> null

function Foo() {
  return {};
}
let foo = new Foo();
// 原型链:	foo ---> Object.prototype ---> null
```
Function 构造函数是一个函数对象，其 [[Class]] 属性是 Function。Function 的 [[Prototype]] 属性指向了 Function.prototype，即
```js
Function.__proto__ === Function.prototype
// true
```
![image](http://resource.muyiy.cn/image/20191215220504.png)
到这里就有意思了，我们看下鸡生蛋蛋生鸡问题。

我们看下面这段代码:
```js
Object instanceof Function 		// true
Function instanceof Object 		// true

Object instanceof Object 			// true
Function instanceof Function 	// true
```
Object 构造函数继承了 Function.prototype，同时 Function 构造函数继承了Object.prototype。这里就产生了 鸡和蛋 的问题。为什么会出现这种问题，因为 Function.prototype 和 Function.__proto__ 都指向 Function.prototype。

```js
// Object instanceof Function 	即
Object.__proto__ === Function.prototype 					// true

// Function instanceof Object 	即
Function.__proto__.__proto__ === Object.prototype	// true

// Object instanceof Object 		即 			
Object.__proto__.__proto__ === Object.prototype 	// true

// Function instanceof Function 即	
Function.__proto__ === Function.prototype					// true
```
个人理解如下：
Function 是 built-in 的对象，也就是并不存在“Function对象由Function构造函数创建”这样显然会造成鸡生蛋蛋生鸡的问题。实际上，当你直接写一个函数时（如 function f() {} 或 x => x），也不存在调用 Function 构造器，只有在显式调用 Function 构造器时（如 new Function('x', 'return x') ）才有。即先有 Function.prototype 然后有的 function Function() ，所以就不存在鸡生蛋蛋生鸡问题了，把 Function.__proto__ 指向 Function.prototype 是为了保证原型链的完整，让 Function 可以获取定义在 Object.prototype 上的方法。

![image](http://resource.muyiy.cn/image/2019-07-24-060321.jpg)
### 2022-03-01 额外总结

1. 每个函数都有一个prototype属性（除了Function.prototype函数），这个属性指向原型对象，这个对象正是调用该构造函数创建的实例的原型；
2. 每一个js对象（null除外）在创建的时候都会与之关联另一个对象，这个对象就是我们所说的原型，每一个对象都会从原型“继承”属性；
3. 每一个实例对象都具有一个属性，```__proto__``` ,这个属性会指向该对象的原型。
4. 


