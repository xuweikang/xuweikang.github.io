---
layout: post
title: 函数记忆
tags: 函数 函数记忆
categories: Practice
---

* TOC 
{:toc}

### 何为函数记忆

函数记忆是指将上次的计算结果缓存起来，当下次调用时，如果遇到相同的参数，就直接返回缓存中的数据。
常用于，复杂且有重复的计算。
例如：斐波那契数列的计算

#### underscore中的实现

```js
function memoize(func, hasher) {
    var memoize = function(params1) {
        var cache = memoize.cache
        var key = '' + (!hasher ? params1 : hasher.apply(this, arguments))
        if (!cache[key]) {
            cache[key] = func.apply(this, arguments)
        }
        return cache[key]
    }
    memoize.cache = {}
    return memoize
}
```

#### 优化斐波那切的计算

```js
// 斐波那契实验
var count = 0
var fibonacci = function(n) {
  count ++
  return n < 2 ? 1 : fibonacci(n-1) + fibonacci(n-2)
}
// fib(10)
// console.log(count)  // 177

var fibonacci = memoize(fibonacci)
fibonacci(10)
console.log(count)  // 11
```



