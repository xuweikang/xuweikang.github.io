---
layout: post
title: 常见Array原型方法实现
tags: Array原型方法 
categories: Practice
---

* TOC 
{:toc}

### Array.prototype.map

```js
Array.prototype.map2 = function(callbackfn, thisArg) {
  if (this == null) {
    throw new TypeError('Cannot read property "map" of null or undefined')
  }
  // 因为类数组对象也可以调用map方法，[].map.call(arguments,...)
  // 所以需要转出数组
  let O = Object(this)
  // 这行确保len是非负数，并且可以把非数字转换成0
  let len = O.length >>> 0
  if (typeof callbackfn !== 'function') {
    throw new TypeError(callbackfn+'is not a function')
  }
  let T = thisArg
  let A = new Array(len)
  let k = 0
  while (k < len) {
    if (k in O) {
      let mappedValue = callbackfn.call(T,O[k],k,O)
      A[k] = mappedValue
    }
    k ++
  }
  return A
}

let arrMap = [1,2,3]

console.log(arrMap.map2(item => item + 1))
```

### Array.prototype.filter
```js
Array.prototype.filter2 = function(callbackfn, thisArg) {
  if (this == null) {
    throw new TypeError('Cannot read property "map" of null or undefined')
  }
  // 因为类数组对象也可以调用filter方法，[].filter.call(arguments,...)
  // 所以需要转出数组
  let O = Object(this)
  // 这行确保len是非负数，并且可以把非数字转换成0
  let len = O.length >>> 0
  if (typeof callbackfn !== 'function') {
    throw new TypeError(callbackfn+'is not a function')
  }
  let T = thisArg
  let A = new Array(len)
  let k = 0
  let to = 0
  while (k < len) {
    if (k in O) {
      if (callbackfn.call(T,O[k],k,O)) {
        A[to++] = O[k]
      }
    }
    k ++
  }
  A.length = to
  return A
}

let arrFilter = [1,2,3]

console.log(arrFilter.filter2(item => item >= 2))
```

### Array.prototype.reduce

```js
Array.prototype.reduce2 = function(callbackfn, initValue) {
  if (this == null) {
    throw new TypeError("Cannot read property 'reduce' of null or undefined");
  }
  if (typeof callbackfn !== 'function') {
    throw new TypeError(callbackfn + ' is not a function')
  }
  let O = Object(this), accu = 0, len = O.length >>> 0, k = 0
  if (initValue) {
    accu = initValue
  } else {
    if (len == 0) {
      throw new TypeError('Reduce of empty array with no initial value');
    }
    let kFlag = false
    while (!kFlag && (k < len)) {
      if (k in O) {
        kFlag = true
        accu = O[k]
      }
      k ++
    }
  }
  while (k < len) {
    if (k in O) {
      accu = callbackfn.call(undefined,accu, O[k],k,O)
    }
    k ++
  }
  return accu

}


arr = [1,2,3]
console.log(arr.reduce2((p,c) => { return p + c }))
```


### Array.prototype.flat

```js
// reduce + concat + 递归
function flat2(arr, d = 1) {
  return d > 0 ? arr.reduce((pre,curr) => pre.concat(Array.isArray(curr) ? flat2(curr,d-1) : curr), []) : arr.slice()
}
// 使用堆栈实现
function flat3(arr) {
  let stack = [...arr], res = []
  while(stack.length) {
    const next = stack.pop()
    if (Array.isArray(next)) {
      stack.push(...next)
    } else {
      res.push(next)
    }
  }
  return res.reverse()
}
Array.prototype.flat2 = function(d = 1) {
  return flat2(this,d)
}
Array.prototype.flat3 = function() {
  return flat3(this)
}

var arr1 = [1,2,3,[1,2,3,4, [2,3,4,[5]]]];
console.log(arr1.flat2(Infinity))
console.log(arr1.flat3())
```