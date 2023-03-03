---
layout: post
title: 惰函数
tags: js 函数 惰函数
categories: jsBase
---

* TOC 
{:toc}

有两种实现惰性载入的方式，第一种事函数在第一次调用时，对函数本身进行二次处理，该函数会被覆盖为符合分支条件的函数，这样对原函数的调用就不用再经过执行的分支了，我们可以用下面的方式使用惰性载入重写addEvent()。

```js
function addEvent (type, element, fun) {
    if (element.addEventListener) {
        addEvent = function (type, element, fun) {
            element.addEventListener(type, fun, false);
        }
    }    else if(element.attachEvent){
        addEvent = function (type, element, fun) {
            element.attachEvent('on' + type, fun);
        }
    }    else{
        addEvent = function (type, element, fun) {
            element['on' + type] = fun;
        }
    }   
     return addEvent(type, element, fun);
    }

```
第二种实现惰性载入的方式是在声明函数时就指定适当的函数。这样在第一次调用函数时就不会损失性能了，只在代码加载时会损失一点性能。一下就是按照这一思路重写的addEvent()。

```js
var addEvent = (function () {  
    if (document.addEventListener) {  
        return function (type, element, fun) {
            element.addEventListener(type, fun, false);
        }
    }  
   else if (document.attachEvent) {  
          return function (type, element, fun) {
            element.attachEvent('on' + type, fun);
        }
    }  
      else {        
         return function (type, element, fun) {
            element['on' + type] = fun;
        }
    }
})();

```


[惰性函数介绍](https://juejin.cn/post/6844903552825950222)