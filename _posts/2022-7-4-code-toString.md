---
layout: post
title: Object.prototype.toString()方法
tags: Object原型方法 toString方法
categories: Practice
---

* TOC 
{:toc}

对于 `Object.prototype.toString()` 方法，会返回一个形如 `"[object XXX]"` 的字符串。

如果对象的 `toString` 方法没有被重写，就会返回如上面形式的字符串。

```javascript
({}).toString();  // => "[object Object]"
Math.toString();  // => "[object Math]"
```

但是，大部分对象的 `toString` 方法被重写了，这时，需要用 `call()` 或者 `Reflect.apply()` 等方法来调用。

比如下面这样，不是我们期望的返回"[object Array]"：
```javascript
[1,2,3].toString() // "1,2,3"
new Array(1,2,3).toString() // "1,2,3"
```
这是因为，Array构造函数重写了原型上的toString方法。

```javascript
// 验证是否是数组
function isArray(obj) {
  return Object.prototype.toString.call(obj) === '[object Array]'
}
isArray([1,2,3]) // true
isArray('1')  // false

```

### Object.prototype.toString 原理

对于 `Object.prototype.toString.call(arg)`，若参数为 `null` 或 `undefined`，直接返回结果。

```javascript
Object.prototype.toString.call(null);       // => "[object Null]"

Object.prototype.toString.call(undefined);  // => "[object Undefined]"
```

若参数不为 `null` 或 `undefined` ，则将参数转为对象，再做判断。

转为对象后，取得该对象的  `[Symbol.toStringTag]` 属性值（可能会遍历原型链）作为 `tag` ，如无该属性，或该属性不为基本类型，则按照下面规则 取 tag值，然后返回 [object,tag]类型的字符串。

> 注意: Symbol.toStringTag 的作用是可以指定 toString方法的tag值，具体在基础9：元编程里面有说明。

```js
// Boolean 类型，tag 为 "Boolean"
Object.prototype.toString.call(true);            // => "[object Boolean]"

// Number 类型，tag 为 "Number"
Object.prototype.toString.call(1);               // => "[object Number]"

// String 类型，tag 为 "String"
Object.prototype.toString.call("");              // => "[object String]"

// Array 类型，tag 为 "String"
Object.prototype.toString.call([]);              // => "[object Array]"

// Arguments 类型，tag 为 "Arguments"
Object.prototype.toString.call((function() {
  return arguments;
})());                                           // => "[object Arguments]"

// Function 类型， tag 为 "Function"
Object.prototype.toString.call(function(){});    // => "[object Function]"

// Error 类型（包含子类型），tag 为 "Error"
Object.prototype.toString.call(new Error());     // => "[object Error]"

// RegExp 类型，tag 为 "RegExp"
Object.prototype.toString.call(/\d+/);           // => "[object RegExp]"

// Date 类型，tag 为 "Date"
Object.prototype.toString.call(new Date());      // => "[object Date]"

// 其他类型，tag 为 "Object"
Object.prototype.toString.call(new class {});    // => "[object Object]"
```

下面为部署了 `Symbol.toStringTag` 的例子。可以看出，属性值期望是一个字符串，否则会被忽略。

```js
var o1 = { [Symbol.toStringTag]: "A" };
var o2 = { [Symbol.toStringTag]: null };

Object.prototype.toString.call(o1);      // => "[object A]"
Object.prototype.toString.call(o2);      // => "[object Object]"
```

`Symbol.toStringTag` 也可以部署在原型链上：

```js
class A {}
// A.prototype[Symbol.toStringTag] = "A";
a = new A()
a.toString();   // => "[object Object]"
// a.toString();   // => "[object A]"
```

### 部署了 Symbol.toStringTag 属性的内置对象

以下，是部署了此属性的内置对象。

#### 1. 三个容器对象

```js
JSON[Symbol.toStringTag];         // => "JSON"
Math[Symbol.toStringTag];         // => "Math"
Atomics[Symbol.toStringTag];      // => "Atomic"

JSON.toString();         // => "[object JSON]"
Math.toString();         // => "[object Math]"
Atomics.toString();      // => "[object Atomics]"
```

#### 2. BigInt 和 Symbol

```js
BigInt.prototype[Symbol.toStringTag];      // => "BigInt"
Symbol.prototype[Symbol.toStringTag];      // => "Symbol"
```

#### 3. **四个集合（Collection）对象**

```js
Set.prototype[Symbol.toStringTag];         // => "Set"

Map.prototype[Symbol.toStringTag];         // => "Map"

WeakSet.prototype[Symbol.toStringTag];     // => "WeakSet"

WeakMap.prototype[Symbol.toStringTag];     // => "WeakMap"
```

#### 4. **`ArrayBuffer` 及其视图对象**

```js
ArrayBuffer.prototype[Symbol.toStringTag];       // => "ArrayBuffer"

SharedArrayBuffer.prototype[Symbol.toStringTag]; // => "SharedArrayBuffer"

DataView.prototype[Symbol.toStringTag];          // => "DataView"
```

#### 5. **模块命名空间对象（Module Namespace Object）**

```js
import * as module from "./export.js";

module[Symbol.toStringTag];        // => "Moduel"
```

#### 6. 其他

浏览器中：

```js
Window.prototype[Symbol.toStringTag];       // => "Window"

HTMLElement.prototype[Symbol.toStringTag];  // => "HTMLElement"

Blob.prototype[Symbol.toStringTag];         // => "Blob"
```

node中：

```js
global[Symbol.toStringTag];                 // => "global"
```

### 什么时候会自动调用toString呢？
使用操作符的时候，如果其中一边为对象，则会先调用toSting方法，也就是隐式转换，然后再进行操作。

```js
console.log([1,2,3] < 2)  // false   
console.log([1,2,3] == '1,2,3')  // true       
console.log([1, 2, 3] > {'a':2})  // false       

```
### 什么情况重写toString方法，什么时候用Symbol.toStringTag？
如果两种写法都存在原型中，肯定优先调用toString方法。
如果像Array这种，需要有额外的方法运算的，采用重写toString；
如果像Math这种静态内部方法容器，只需要一个字符串就可以达到要求的话，或者只想标示一个类的实例化标识，采用Symbol.toStringTag。

### 检测数组的优化

优化前：

```js
// 验证是否是数组
function isArray(obj) {
  return Object.prototype.toString.call(obj) === '[object Array]'
}
isArray([1,2,3]) // true
isArray('1')  // false
```

优化后：

```js
var toStr = Function.prototype.call.bind(Object.prototype.toString)
function isArray(obj) {
  return toStr(obj) === '[object Array]'
}
isArray([1,2,3]) // true
isArray('1')  // false

toStr([1, 2, 3]); 	// "[object Array]"
toStr("123"); 		// "[object String]"
toStr(123); 		// "[object Number]"
toStr(Object(123)); // "[object Number]"
```

其实，`Function.prototype.call.bind(Object.prototype.toString)` 就相当于，`Object.prototype.toString.call()` ，其中 toStr 的方法的第一个参数为传入的参数。


### valueOf
具体功能与toString大同小异，同样具有以上的自动调用和重写方法。

```js
let c = [1, 2, 3]
let d = {a:2}

console.log(c.valueOf())    // [1, 2, 3]
console.log(d.valueOf())    // {a:2}
```

#### 两者区别
- 共同点：在输出对象时会自动调用。
- 不同点：默认返回值不同，且存在优先级关系。

二者并存的情况下，在数值运算中，优先调用了valueOf，字符串运算中，优先调用了toString。
```js
class A {
    valueOf() {
        return 2
    }
    toString() {
        return '哈哈哈'
    }
}
let a = new A()

console.log(String(a))  // '哈哈哈'  
console.log(Number(a))  // 2        
console.log(a + '22')   // '222'     
console.log(a == 2)     // true      
```
a + 22, 和我们预想的是不是不太一样，a难道不是转换成字符串后进行运算么，转换成字符串难道不是优先调用toString方法么。
其实不是这样的，'+' 作为二元操作符进行运算的时候，需要'+'两边的值都通过ToPrimitive规范转换成基本类型后，再进行运算。a通过ToPrimitive转换成基本类型，优先调用valueOf值，如果没有再看toString方法。

总结下：valueOf偏向于运算，toString偏向于显示。
1. 在进行对象转换时，将优先调用toString方法，如若没有重写 toString，将调用 valueOf 方法；如果两个方法都没有重写，则按Object的toString输出。
2. 在进行强转字符串类型时，将优先调用 toString 方法，强转为数字时优先调用 valueOf。
3. 使用运算操作符的情况下，valueOf的优先级高于toString。

### [Symbol.toPrimitive]
> MDN：Symbol.toPrimitive 是一个内置的 Symbol 值，它是作为对象的函数值属性存在的，当一个对象转换为对应的原始值时，会调用此函数。


- 作用：同valueOf()和toString()一样，但是优先级要高于这两者；
- 该函数被调用时，会被传递一个字符串参数hint，表示当前运算的模式，一共有三种模式：
- 1. string：字符串类型
- 2. number：数字类型
- 3. default：默认

```js
class A {
    constructor(count) {
        this.count = count
    }
    valueOf() {
        return 2
    }
    toString() {
        return '哈哈哈'
    }
    // 我在这里
    [Symbol.toPrimitive](hint) {
        if (hint == "number") {
            return 10;
        }
        if (hint == "string") {
            return "Hello Libai";
        }
        return true;
    }
}

const a = new A(10)

console.log(`${a}`)     // 'Hello Libai' => (hint == "string")
console.log(String(a))  // 'Hello Libai' => (hint == "string")
console.log(+a)         // 10            => (hint == "number")
console.log(a * 20)     // 200           => (hint == "number")
console.log(a / 20)     // 0.5           => (hint == "number")
console.log(Number(a))  // 10            => (hint == "number")
console.log(a + '22')   // 'true22'      => (hint == "default")
console.log(a == 10)     // false        => (hint == "default")
```
注意：
1. 该方法兼容性不太好，不兼容ie所有版本；
2. +如果做为二元运算符的时候[Symbol.toPrimitive]是default模式，== 同样。