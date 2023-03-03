---
layout: post
title: 什么是元编程？
tags: js 元编程 Symbol符号
categories: jsBase
---

* TOC 
{:toc}

维基百科：

> `元编程` （meta programming）是一种编程技术，编写出来的计算机程序能够将其他程序作为数据来处理。
>
> 意味着可以编写出这样的程序：它能够`读取、生成、分析或者转换`其它程序，甚至在`运行时修改程序自身`（反射）。

举个例子，如果想要查看对象a和对象b之间的关系，是否是通过`[[prototype]]`链接的。我们可以通过`a.isPrototype(b)`,

这就是一种元编程形式，称为**自省**。

用for..in 循环枚举对象的键，或者检查一个对象是否是某个“类构造器”的实例，也都是

常见的元编程例子。

元编程中 **元** 的概念，可以理解为 程序 本身，元编程关注以下的一点或几点：

1.  运行时修改语言结构，这种现象被称为 **反射**；
   - **自省**：代码检视自己；
   - **自我修改：**代码修改自己；
   - **调解：**代码修改 **默认的语言行为** 而使其他代码受到影响。
2. 生成代码。



### 1、自省

代码能够自我检查、访问内部属性，获得代码的底层信息!

```javascript
// 访问对象自身属性
var users = {
'Tom': 32,
'Bill': 50
};
Object.keys(users).forEach(name => {
	const age = users[name];
	console.log(`User ${name} is ${age} years old!`);
});
```

### 2、自我修改

代码可以修改自身属性或者其他底层信息！

```javascript
let a = 1;
if (a == 1 && a == 2 && a == 3) {
  console.log("元编程");
}
```

上述代码，正常情况下是没办法满足条件打印输出的。

但是，利用元编程，有办法可以办到。

```javascript
// 修改自身
let a = {
 [Symbol.toPrimitive]: ((i) => () => ++i)(0);
}
if (a == 1 && a == 2 && a == 3) {
console.log("元编程");
}
```

`Symbol.toPrimitive` 是一个内置的Symbol元属性，用在对象中作为函数的属性名存在，如果对象中声明了该属性，当该对象被转换为基本值的时候会被调用。mdn(https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Symbol/toPrimitive)。

在开发过程中**自我修改**应该要尽力避免，可以想象：正在使用一个数据的同时又在修改这个数据，后容易造成不可预期的错误！



### 3、 调解

代码`修改默认的语言行为`而使其他代码受影响，最明显的体现为改变其它对象的语义！

在元编程中，调解的概念类似于包装、捕获、拦截。

`Object.defineProperty()`就是典型的 调解 的运用。

还有ES6的 `Proxy` 和 `Reflect`。

### 4、 生成代码

js中使用元编程技术生成代码最常见的函数 `eval()`：函数会将传入的字符串当做 JavaScript 代码进行执行。

```javascript
let str = "function sayHello(){console.log('hello')}";
eval(str);
sayHello();
// 输出
hello
```

### 5、 总结

元编程是当你将程序的逻辑转向关注它自身（或者它的运行时环境）时进行的编程，要么为了调查它自己的结构，要么为了修改它。

元编程的主要价值是`扩展语言的普通机制来提供额外的能力`。





### 公开符号

ES6 新原生类型 symbol，除了我们可以通过symbol自定义一些符号外。js还预先定义了一些内置符号，称为 **公开符号**。

定义这些符号主要是为了提供专门的元属性，以便把这些元属性暴露给 JavaScript 程序以获取对 JavaScript 行为更多的控制。

#### 1. Symbol.iterator

Symbol.iterator 表示任意对象上的一个专门位置（属性），语言机制自动在这个位置上寻找一个方法，这个方法构造一个迭代器来消耗这个对象的值。很多对象定义有这个符号的默认值。

```javascript
var arr = [4,5,6,7,8,9]; 
for (var v of arr) { 
 console.log( v ); 
} 
// 4 5 6 7 8 9
// 定义一个只在奇数索引值产生值的迭代器
arr[Symbol.iterator] = function*() { 
  var idx = 1; 
  do { 
   yield this[idx]; 
   } while ((idx += 2) < this.length); 
};
for (var v of arr) { 
 console.log( v ); 
} 
// 5 7 9
```

#### 2. Symbol.toStringTag 与 Symbol.hasInstance

最常见的一个元编程任务，就是在一个值上进行自省来找出它是什么种类，这通常是

为了确定其上适合执行何种运算。对于对象来说，最常用的自省技术是 toString() 和

instanceof。

```javascript
function Foo() {} 
var a = new Foo(); 
a.toString(); // [object Object]  第二个Object是对象的类型
a instanceof Foo; // true

// 在 ES6 中，可以控制这些操作的行为特性：
function Foo(greeting) { 
	this.greeting = greeting; 
} 
Foo.prototype[Symbol.toStringTag] = "Foo"; // 1
Object.defineProperty( Foo, Symbol.hasInstance, { 
 value: function(inst) { 
	return inst.greeting == "hello"; 
 } 
} ); 
var a = new Foo( "hello" ), 
 b = new Foo( "world" ); 
b[Symbol.toStringTag] = "cool";  // 2
a.toString(); // [object Foo]  // 3
String( b ); // [object cool]  // 4
a instanceof Foo; // true 
b instanceof Foo; // false
```

通过上述代码，我们可以知道，通过修改原型对象或者实例本身的Symbol.toStringTag属性值，可以替换  [object  A]  字符串化时使用的A的值（1和2）。

@@hasInstance 符号是在构造器函数上的一个方法，接受实例对象值，通过返回 true 或false 来指示这个值是否可以被认为是一个实例。

#### 3. Symbol.species

不懂。

#### 4. Symbol.toPrimitive

这个在前面也用到了，在一个对象，被强制转换为一个基本类型的时候，所使用到哪个基本值。

```javascript
var arr = [1,2,3,4,5]; 
arr + 10; // 1,2,3,4,510 
arr[Symbol.toPrimitive] = function(hint) { 
 if (hint == "default" || hint == "number") { 
 // 求所有数字之和
	return this.reduce( function(acc,curr){ 
		return acc + curr; 
 	}, 0 ); 
 } 
}; 
arr + 10; // 25
```

#### 5. Symbol.isConcatSpreadable

符号 @@isConcatSpreadable 可以被定义为任意对象（比如数组或其他可迭代对象）的布尔型属性（Symbol.isConcatSpreadable），用来指示如果把它传给一个数组的 concat(..) 是否应该将其展开。

```javascript
var a = [1,2,3], 
 b = [4,5,6]; 
b[Symbol.isConcatSpreadable] = false; 
[].concat( a, b ); // [1,2,3,[4,5,6]]
```

#### 6. Symbol.unscopables

符号 @@unscopables 可以被定义为任意对象的对象属性（Symbol.unscopables），用来指示使用 with 语句时哪些属性可以或不可以暴露为词法变量。

```javascript
var o = { a:1, b:2, c:3 }, 
 a = 10, b = 20, c = 30; 
o[Symbol.unscopables] = { 
 a: false, 
 b: true, 
 c: false
}; 
with (o) { 
 console.log( a, b, c ); // 1 20 3 
}
```



#### 7. Proxy  和 Reflect

##### 7.1 代理局限性

可以在对象上执行的很广泛的一组基本操作都可以通过这些元编程处理函数 trap。但有一些操作是无法（至少现在）拦截的。

比如，下面这些操作都不会 trap 并从代理 pobj 转发到目标 obj：

```javascript
var obj = { a:1, b:2 }, 
 handlers = { .. }, 
 pobj = new Proxy( obj, handlers ); 

typeof obj; 
String( obj );
obj + ""; 
obj == pobj; 
obj === pobj
```

##### 7.2 可取消代理

普通代理总是陷入到目标对象，并且在创建之后不能修改——只要还保持着对这个代理的引用，代理的机制就将维持下去。但是，可能会存在这样的情况，比如你想要创建一个在你想要停止它作为代理时便可以被停用的代理。解决方案是创建可取消代理（revocable proxy）：

```javascript
 handlers = { 
 	get(target,key,context) { 
 		console.log( "accessing: ", key ); 
		return target[key]; 
 	} 
 }, 
 { proxy: pobj, revoke: prevoke } = 
 Proxy.revocable( obj, handlers ); 
pobj.a; 
// accessing: a 
// 1 
// 然后：
prevoke(); 
pobj.a; 
// TypeError
```

#### 7.3 Reflect

Reflect 对象是一个平凡对象（就像 Math），不像其他内置原生值一样是函数 / 构造器。它持有对应于各种可控的元编程任务的静态函数。这些函数一对一对应着代理可以定义的处理函数方法（trap）。

`Reflect` 內部封裝了一系列对目标的最底层实际操作。

js的反射现象，其实就是代码自己运行过程中去检视或者修改自己。通俗点，其实就是一个对象被遍历，我们都可以认为是反射。

js 的`for in` ，还有`Object.keys` 这些其实都是反射。

那么为什么还需要 `Reflect` 这个东西呢？

提供Reflect对象将这些能够实现反射机制的方法都归结于一个地方并且做了简化，保持JS的简单。于是我们再也不需要调用`Object`对象，然后写上很多的代码。

比如：

```javascript
var myObject = Object.create(null) // 此时myObject并没有继承Object这个原型的任何方法,因此有：

myObject.hasOwnProperty === undefined // 此时myObject是没有hasOwnProperty这个方法，那么我们要如何使用呢？如下：

Object.prototype.hasOwnProperty.call(myObject, 'foo') // 是不是很恐怖，写这么一大串的代码！！！！
```

而使用Reflect以后：

```javascript
var myObject = Object.create(null)
Reflect.ownKeys(myObject)
```