---
layout: post
title: 尾递归优化
tags: 最大公约数 尾递归 递归优化
categories: Practice
---

* TOC 
{:toc}

### 尾递归和普通递归有啥区别

尾调用，是指函数内部的最后一个动作是函数调用。该调用的返回值，直接返回给函数。

举个例子：
```js
// 尾调用
function f(x){
    return g(x);
}
// 非尾调用
function f(x){
    return g(x) + 1;
}
```
模拟下上述执行上下文栈：
尾调用：
```js
ECStack.push(<f> functionContext)
ECStack.pop()
ECStack.push(<g> functionContext)
ECStack.pop()
```
非尾调用：
```js
ECStack.push(<f> functionContext)
ECStack.push(<g> functionContext)
ECStack.pop()
ECStack.pop()
```
也就说尾调用函数执行时，虽然也调用了一个函数，但是因为原来的的函数执行完毕，执行上下文会被弹出，执行上下文栈中相当于只多压入了一个执行上下文。然而非尾调用函数，就会创建多个执行上下文压入执行上下文栈。

函数调用自身，称为递归。如果尾调用自身，就称为尾递归。

### 尾递归优化例子

阶乘优化：
```js
function factirial(n, total = 1) {
    if (n < 2) return total
    return factirial(n - 1, n * total)
}
factorial(5,1) // 5 * 4 * 3 * 2 * 1 = 120
```

累加求和：
```js
function sum(n, total = 0) {
    if (n < 1) return total
    return sum(n-1, n + total)
} 
sum(5,1) // 5 + 4 + 3 + 2 + 1 = 15
```

斐波那契：
```js
function fib(n, sum1 = 1, sum2 = 2) {
    if (n < 2) return sum2
    return fib(n-1, sum2, sum1 + sum2)
}
fib(100)
```