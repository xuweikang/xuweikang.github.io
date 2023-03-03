---
layout: post
title: 节流和防抖实现
tags: 节流 防抖 throttle debance
categories: Practice
---

* TOC 
{:toc}

## 节流
函数节流指的是某个函数在一定时间间隔内（例如 3 秒）只执行一次，在这 3 秒内 无视后来产生的函数调用请求，也不会延长时间间隔。3 秒间隔结束后第一次遇到新的函数调用会触发执行，然后在这新的 3 秒内依旧无视后来产生的函数调用请求，以此类推。

此时「管道中的水」就是我们频繁操作事件而不断涌入的回调任务，它需要接受「水龙头」安排；「水龙头」就是节流阀，控制水的流速，过滤无效的回调任务；「滴水」就是每隔一段时间执行一次函数，「3 秒」就是间隔时间，它是「水龙头」决定「滴水」的依据。

比如：提交按钮点击、下拉加载更多、onSrcoll、onresize等事件

#### 实现
我们通常有两种方法实现节流

方案1:
```js
function throttle(func, wait = 1000) {
  let previous = 0
  return function(...args) {
    const now = +new Date()
    if (now - previous > wait) {
      previous = now
      func.apply(this,args)
    }
  }
}
const throttleTask = throttle(() => console.log('节流函数运行了'),2000)
setInterval(throttleTask,500)
```
方案2:
```js
function throttle2(func, wait = 1000) {
  let timer = null
  return function(...args) {
    if (!timer) {
      timer = setTimeout(() => {
        func.apply(this,args)
        timer = null
      },wait)
    }
  }
}
const throttleTask = throttle2(() => console.log('节流函数运行了'),2000)
setInterval(throttleTask,500)
```

方案1的特点是：立刻执行事件，并且事件停止后不会继续执行了
方案2的特点是：延迟了执行事件，并且事件停止后还可以执行一次事件

#### underscore 节流的实现
上述代码实现了一个简单的节流函数，不过 underscore 实现了更高级的功能，即新增了两个功能，

1. 配置是否需要响应事件刚开始的那次回调（ leading 参数，false 时忽略）
2. 配置是否需要响应事件结束后的那次回调（ trailing 参数，false 时忽略）

所以需要搭配上述方案1和方案2共同实现。

```js
// underscore throttle 实现
function throttle_underscore(func, wait, options = {}) {
  let timer, context, previous = 0, args,result
  let later = function() {
    // 当设置 { leading: false } 时
    // 每次触发回调函数后设置 previous 为 0
    // 不然为当前时间
    previous = options.leading === false ? 0 : (+new Date())
    // 防止内存泄漏，置为 null 便于后面根据 !timeout 设置新的 timeout
    timer = null
    result = func.apply(context, args)
    if (!timeout) context = args = null
  }
  let throttled = function() {
    let now = +new Date()
    context = this
    args = arguments
    if (!previous && options.leading == false) previous = now
    let remaining = wait - (now - previous)
    if (remaining < 0 || remaining > wait) {
      if (timer) {
        clearTimeout(timer)
        timer = null
      }
      previous = now
      result = func.apply(context, args)
      if (!timer) {
        // 这应该只是为了垃圾回收
        context = args = null;
      }
    } else if (!timer && options.trailing) {
      timer = setTimeout(later, remaining)
    }
    return result
  }
  return throttled
}
```

##  防抖
防抖函数 debounce 指的是某个函数在某段时间内，无论触发了多少次回调，都只执行最后一次。假如我们设置了一个等待时间 3 秒的函数，在这 3 秒内如果遇到函数调用请求就重新计时 3 秒，直至新的 3 秒内没有函数调用请求，此时执行函数，不然就以此类推重新计时。

举一个小例子：假定在做公交车时，司机需等待最后一个人进入后再关门，每次新进一个人，司机就会把计时器清零并重新开始计时，重新等待 1 分钟再关门，如果后续 1 分钟内都没有乘客上车，司机会认为乘客都上来了，将关门发车。

此时「上车的乘客」就是我们频繁操作事件而不断涌入的回调任务；「1 分钟」就是计时器，它是司机决定「关门」的依据，如果有新的「乘客」上车，将清零并重新计时；「关门」就是最后需要执行的函数。

普通版实现如下：
```js
function debounce(func, wait = 1000, immediate){
  let timer
  return function(...args) {
    if (timer) {
      clearTimeout(timer)
    }
    if (immediate && !timer) {
      // 第一次立即执行
      func.apply(this,args)
    }
    timer = setTimeout(() => {
      func.apply(this,args)
    }, wait)
  }
}
const betterFn = debounce(() => console.log('fn 防抖执行了'), 1000, true)
// 第一次触发 scroll 执行一次 fn，后续只有在停止滑动 1 秒后才执行函数 fn
document.addEventListener('scroll', betterFn)
```
加强版的节流：
wait 时间内重复触发会使用防抖只触发最后一次。如果在wait时间后，必须触发一次。
这样可以解决用户，一直在时间内不停触发防抖，导致fn始终无法执行的情况。
```js
function throttleDebounce(fn, wait) {
  let previous = 0, timer = null
  return function(...args) {
    let now = +new Date()
    if (now - previous >= wait) {
      previous = now
      fn.apply(this, args)
    } else {
      if (timer) clearTimeout(timer)
      timer = setTimeout(() =>  {
        previous = now
        fn.apply(this, args)
      }, wait)
    }
  }
}
const betterFn2 = throttleDebounce(() => console.log('fn 节流执行了'), 1000)
// 第一次触发 scroll 执行一次 fn，每隔 1 秒后执行一次函数 fn，停止滑动 1 秒后再执行函数 fn
document.addEventListener('scroll', betterFn2)
```


