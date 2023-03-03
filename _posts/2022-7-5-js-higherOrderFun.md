---
layout: post
title: 高阶函数
tags: js 函数 高阶函数 
categories: jsBase
---

* TOC 
{:toc}

## 高阶函数
高阶函数英文叫 Higher-order function，它的定义很简单，就是至少满足下列一个条件的函数：
- 接受一个或多个函数作为输入
- 输出一个函数
也就是说高阶函数是对其他函数进行操作的函数，可以将它们作为参数传递，或者是返回它们。 简单来说，高阶函数是一个接收函数作为参数传递或者将函数作为返回值输出的函数。


## 函数作为参数传递
js语言中内置了一些高阶函数，比如 Array.prototype.map，Array.prototype.filter 和 Array.prototype.reduce，它们接受一个函数作为参数，并应用这个函数到列表的每一个元素。


## 函数作为返回值输出
比较经典的应用，就是 函数柯里化。


## 练习题
已知如下数组，编写一个程序将数组扁平化去并除其中重复部分数据，最终得到一个升序且不重复的数组

> var arr = [ [1, 2, 2], [3, 4, 5, 5], [6, 7, 8, 9, [11, 12, [12, 13, [14] ] ] ], 10];


```js
arr.toString().split(',').sort((a,b) => a-b).filter((item,index,self) => self.indexOf(item) === index)
```


## 柯里化

### 定义
函数柯里化又叫部分求值，维基百科中对柯里化 (Currying) 的定义为：
> 在数学和计算机科学中，柯里化是一种将使用多个参数的函数转换成一系列使用一个参数的函数，并且返回接受余下的参数而且返回结果的新函数的技术。

用大白话来说就是只传递给函数一部分参数来调用它，让它返回一个新函数去处理剩下的参数。

### 实际应用
## 实现currying函数
我们可以理解所谓的柯里化函数，就是封装「一系列的处理步骤」，通过闭包将参数集中起来计算，最后再把需要处理的参数传进去。那如何实现 currying 函数呢？
实现原理就是「用闭包把传入参数保存起来，当传入参数的数量足够执行函数时，就开始执行函数」。上面延迟计算部分已经实现了一个简化版的 currying 函数。


```js
function currying(fn,length) {
    length = length || fn.length
    return function(...args) {
        return args.length >= length 
        ? fn.apply(this,args) 
        : currying(fn.bind(this,...args),length - args.length)
    }
}
```
或者用es6的写法，
```js
const currying = fn =>
    judge = (...args) =>
    args.length >= fn.length 
    ? fn(...args)
    : (...arg) => judge(...args,...arg)
```

Test
```js
// Test
const fn = currying(function(a, b, c) {
    console.log([a, b, c]);
});

fn("a", "b", "c") // ["a", "b", "c"]
fn("a", "b")("c") // ["a", "b", "c"]
fn("a")("b")("c") // ["a", "b", "c"]
fn("a")("b", "c") // ["a", "b", "c"]
```

#### 扩展：函数参数 length
函数 currying 的实现中，使用了 fn.length 来表示函数参数的个数，那 fn.length 表示函数的所有参数个数吗？并不是。

函数的 length 属性获取的是形参的个数，但是形参的数量不包括剩余参数个数，而且仅包括第一个具有默认值之前的参数个数，看下面的例子。

```js
((a, b, c) => {}).length; 
// 3

((a, b, c = 3) => {}).length; 
// 2 

((a, b = 2, c) => {}).length; 
// 1 

((a = 1, b, c) => {}).length; 
// 0 

((...args) => {}).length; 
```
所以在柯里化的场景中，不建议使用 ES6 的函数参数默认值。