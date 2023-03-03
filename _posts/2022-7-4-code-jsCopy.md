---
layout: post
title: 深浅拷贝的实现
tags: 深拷贝 浅拷贝
categories: Practice
---

* TOC 
{:toc}

> MDN: 主要是将所有**可枚举属性**的值从一个或多个源对象复制到目标对象，同时**返回目标对象**。

```js
// 第一步
let a = {
    name: "advanced",
    age: 18
}
let b = {
    name: "muyiy",
    book: {
        title: "You Don't Know JS",
        price: "45"
    }
}
let c = Object.assign(a, b);
console.log(c);
// {
// 	name: "muyiy",
//  age: 18,
// 	book: {title: "You Don't Know JS", price: "45"}
// } 
console.log(a === c);
// true

// 第二步
b.name = "change";
b.book.price = "55";
console.log(b);
// {
// 	name: "change",
// 	book: {title: "You Don't Know JS", price: "55"}
// } 

// 第三步
console.log(a);
// {
// 	name: "muyiy",
//  age: 18,
// 	book: {title: "You Don't Know JS", price: "55"}
// } 
```

### `Object.assign` 模拟实现

实现一个 `Object.assign` 大致思路如下：

1. 判断原生 `Object` 是否支持该函数，如果不存在的话创建一个函数 `assign`，并使用 `Object.defineProperty` 将该函数绑定到 `Object` 上。

2. 判断参数是否正确（目标对象不能为空，我们可以直接设置{}传递进去,但必须设置值）。
3. 使用 `Object()` 转成对象，并保存为 to，最后返回这个对象 to。
4. 使用 `for..in` 循环遍历出所有可枚举的自有属性。并复制给新的目标对象（使用 `hasOwnProperty` 获取自有属性，即非原型链上的属性）。

```js
if (typeof Object.assign2 != 'function') {
  // Attention 1
  Object.defineProperty(Object, "assign2", {
    value: function (target) {
      'use strict';
      if (target == null) {  // Attention 2
        throw new TypeError('Cannot convert undefined or null to object');
      }
      // Attention 3
      var to = Object(target)
      for (var i = 1; i<arguments.length; index ++) {
        var nextSource = arguments[i]
        if (nextSource != null) {   // // Attention 2
          // Attention 4
          for (var nextKey in nextSource) {
            if (Object.prototype.hasOwnProperty.call(nextSource, nextKey)) {
              to[nextKey] = nextSource[nextKey]
            }
          }
        }
      }
      return to
    },
    writable: true,
    configurable: true,
    enumerable: false
  })
}
```

####  Attention 1 : 可枚举性

在模拟`assign`方法的时候，为什么不直接`Object`上添加方法呢。这是因为原生情况下挂载在 `Object` 上的属性是不可枚举的，但是直接在 `Object` 上挂载属性 `a` 之后是可枚举的，所以这里必须使用 `Object.defineProperty`，并设置 `enumerable: false` 以及 `writable: true, configurable: true`。

```js
// 下面两种方法，可以判断Object.assign方法是否可枚举

//方法1:
Object.getOwnPropertyDescriptor(Object, "assign");
// {
// 	value: ƒ, 
//  writable: true, 	// 可写
//  enumerable: false,  // 不可枚举，注意这里是 false
//  configurable: true	// 可配置
// }

// 方法2:
Object.propertyIsEnumerable("assign");
// false
```

所以要实现 `Object.assign` 必须使用 `Object.defineProperty`，并设置 `writable: true, enumerable: false, configurable: true`，当然默认情况下不设置就是 `false`。

#### Attention 2: 判断参数是否正确

 `Object.assign` 的第一个参数目标对象，是不允许是 `null` 和 `undefined`的，不然会报错。

后面的参数，可以是  `null` 和 `undefined`， 但是会过滤掉。

其中，`null == undefined` ，所以，只需要判断一个条件就行了。

#### Attention 3:  原始类型被包装为对象

```js
// 以下代码不能运行，会报错
var a = "abc";
var b = "def";
Object.assign(a, b); 
```

因为，基本类型的变量a作为目标值的时候，会被 `Object(a)` 转为 `Object('abc')`，

```js
// String {'abc'}
格式如下，
0: "a"
1: "b"
2: "c"
length: 3
```

 `Object("abc")` 时，其属性描述符为不可写，即 `writable: false`。所以他的0，1，2的属性的key值，是不可修改的。在将 'def' 转换成可枚举的key得到的也是0，1，2，所以在赋值的时候会报错。

```js
// 同样也会报错
var a = "abc";
var b = {
  0: "d"
};
Object.assign(a, b); 
```

>注意：在非严格模式下，对于不可写的属性值（writable: false）的修改静默失败，在严格模式下才会提示错误，所以代码中要加上 'use strict'。（Object.assign会报错的，所以我们在模拟的时候也要加入这个严格模式）

#### Attention 4:  存在性

如何在不访问属性值的情况下判断对象中是否存在某个属性呢，看下面的代码。

```js
var anotherObject = {
    a: 1
};

// 创建一个关联到 anotherObject 的对象
var myObject = Object.create( anotherObject );
myObject.b = 2;

("a" in myObject); // true
("b" in myObject); // true

myObject.hasOwnProperty( "a" ); // false
myObject.hasOwnProperty( "b" ); // true
```

1、`in` 操作符会检查属性是否在对象及其 `[[Prototype]]` 原型链中。

2、`for in` 的遍历方法，会遍历出 自身和原型对象上的所有可枚举属性。

3、`hasOwnProperty(..)` 只会检查属性是否在 `myObject` 对象中，不会检查 `[[Prototype]]` 原型链。

`Object.assign` 方法肯定不会拷贝原型链上的属性，所以模拟实现时需要用 `hasOwnProperty(..)` 判断处理下，但是直接使用 `myObject.hasOwnProperty(..)` 是有问题的，因为有的对象可能没有连接到 `Object.prototype` 上（比如通过 `Object.create(null)` 来创建），这种情况下，使用 `myObject.hasOwnProperty(..)` 就会失败。

```js
var myObject = Object.create( null );
myObject.b = 2;

("b" in myObject); 
// true

myObject.hasOwnProperty( "b" );
// TypeError: myObject.hasOwnProperty is not a function
```

使用 `call` 就可以解决了。

```js
Object.prototype.hasOwnProperty.call(obj,key)
```

### 深拷贝的完美实现

1. **简单实现**（浅拷贝+ 递归）

   ```js
   function cloneDeep1(source) {
     var target = {}
     for (let key in source) {
       if (Object.prototype.hasOwnProperty.call(source,key)) {
         if (typeof source[key] === 'object') {
           target[key] = cloneDeep1(source[key])
         }else{
           target[key] = source[key]
         }
       }
     }
     return target
   }
   ```

2. **拷贝数组**

```js
function isObject(obj) {
  return typeof obj === 'object' && obj != null
}
function cloneDeep2(source) {
  if (!isObject(source)) return source // 非对象返回本身
  var target = Array.isArray(source) ? [] : {}
  for (let key in source) {
    if (Object.prototype.hasOwnProperty.call(source,key)) {
      if (isObject(source[key])) {
        target[key] = cloneDeep2(source[key])
      }else{
        target[key] = source[key]
      }
    }
  }
  return target
}
```

3. **解决循环引用的问题**

   我们知道 `JSON` 无法深拷贝循环引用，遇到这种情况会抛出异常。

   那么我们如何解决这个问题呢。

   ```js
   var a = {
       book: {
           title: "You Don't Know JS",
           price: "45"
       },
       a1: undefined,
       a2: null,
       a3: 123
   }
   a.circleRef = a;
   
   JSON.parse(JSON.stringify(a));
   // TypeError: Converting circular structure to JSON
   ```

   使用map，存储已经拷贝过的对象和值，如果后面又用到了，可以直接从map中取出来。

   ```js
   function cloneDeep3(source,hash = new WeakMap()) {
     if (!isObject(source)) return source // 非对象返回本身
     if (hash.has(source)) return hash.get(source)
     var target = Array.isArray(source) ? [] : {}
     hash.set(source,target)
     for (let key in source) {
       if (Object.prototype.hasOwnProperty.call(source,key)) {
         if (isObject(source[key])) {
           target[key] = cloneDeep3(source[key],hash)
         }else{
           target[key] = source[key]
         }
       }
     }
     return target
   }
   ```
 
 
4. **支持Symbol**
   ```js
     function cloneDeep4(source, hash = new WeakMap()) {
      // 非对象返回自身
      if(typeof source != 'object' || source == null) return source
      if (hash.has(source)) return hash.get(source)  // 查哈希表
      var target = Array.isArray(source) ? [] : {}
      hash.set(source, target)
    
      Reflect.ownKeys(source).forEach(key => {
        if (Object.prototype.hasOwnProperty.call(source, key)) {
          if (typeof source[key] != 'object' || source[key] == null) {
            target[key] = source[key]
          } else {
            target[key] = cloneDeep3(source[key], hash)
          }
        }
      })
      return target
    }
   ```

5. **解决爆栈的问题**
   上面的实现，使用的都是递归方法，递归的缺点就是容易爆栈。
   那要如何解决呢，使用循环就可以了。
  ```js
    function cloneDeep5(x) {
      const root = {}
      const loopList = [{
        parent: root,
        key: undefined,
        data: x
      }]
      const hash = new WeakMap()
      while (loopList.length) {
        const node = loopList.pop()
        const { parent, key, data} = node
        let child = parent
        if (typeof key != 'undefined') {
          child = parent[key] = {}    
        }
    
        if (hash.has(data)) {
          parent[key] = hash.get()
          continue
        }
        hash.set(data,child)
        for (let k in data) {
          if (data.hasOwnProperty(k)) {
            if (typeof data[k] === 'object') {
              loopList.push({
                parent: child,
                key: k,
                data: data[k]
              })
            } else {
              child[k] = data[k]
            }
    
          }
        }
      }
      return root
    }
  ```