---
layout: post
title: 闭包的定义
tags: js 闭包 closure 内存泄漏
categories: jsBase
---

* TOC 
{:toc}

MDN 对闭包的定义：

> 闭包是指那些能够访问 **自由变量** 的 **函数**。
>
> 两个点，首先闭包必须是 **函数**，其次必须要能访问 **自由变量**

自由变量指哪些？

> 自由变量是指在函数中使用的，但既不是函数参数也不是函数内局部变量的变量。

由此，我们可以看出闭包由两部分构成：

> 闭包 = 函数 + 函数中使用的自由变量

举个例子：

```javascript
var a = 1
function foo() {
  console.log(a)
}
foo()
```

foo函数可以访问变量a，但是a既不是函数内局部变量也不是参数变量，所以a是一个自由变量。所以，函数foo + 变量a 就构成了一个闭包。

> 《javaScript权威指南》：从技术的角度来说，所有的javascript函数都是闭包。

这是理论上的闭包，还有一种实践上的闭包：

ECMAScript中，闭包指的是：

1.  从理论角度：所有的函数。因为它们在创建的时候都把上层的上下文都保存起来了。哪怕是简单的全局变量都如此，因为函数中访问全局变量就相当于在访问自由变量。
2.  从实践角度：以下函数才算是闭包：
    - 即使创建它的上下文已经销毁，它仍然存在（比如，内部函数从父函数中返回）。
    - 在代码中引用了自由变量。

### 分析

``` javascript
var data = []
for (var i = 0; i < 3; i++) {
  data[i] = function () {
    console.log(i);
  };
}

data[0]();
data[1]();
data[2]();
```

很显然，答案都是3，让我们分析一下原因：

当执行到 ``` data[0]() ``` 的时候，

此时全局上下文的VO为，

``` javascript
globalContext.VO = {
  data: [...],
  i: 3 
}
```

函数 ``` data[0] ``` 的AO和作用域如下，

```javascript
data[0]Context = {
  AO: {
    arguments: {
      length: 0
    }
  },
  Scope: [AO, globalContext.VO	]
}
```

我们可以看到，data[0]Context的AO中并没有i值，往上层作用域查找，找到globalContext.VO中i为3，所以打印的结果是3。data[1] 和 data[2] 是一样的道理。

如果我们想正确打印出想要的结果，可以使用闭包改动一下，

```javascript
var data = []
for (var i = 0; i < 3; i++) {
  data[i] = (function(i) {
    return function () {
      console.log(i);
    }
  })(i);
}

data[0]();
data[1]();
data[2]();
```

同样，分析一下上面的代码，

当执行到 ``` data[0]() ``` 的时候，此时全局上下文的VO没变，还是

```javascript
globalContext.VO = {
  data: [...],
  i: 3 
}
```

但是当函数执行的时候，data[0]的作用域发生了改变，为

```javascript
data[0]Context = {
  scope: [ AO, 匿名函数Context.AO, globalContext.VO ]
}
```

匿名函数Context.AO为

```javascript
匿名函数Context.AO = {
  arguments: {
    0: 0,
    length: 1	
  },
  i: 0
}
```

所以，当查找i值的时候，在当前AO里面没找到，会沿着作用域链往上查找，找到匿名函数Context上下文的AO，找到了i值，然后输出。





### BUT ，闭包真的就是上述这样么？

当然不是，由于目前大多数网络上存在的关于闭包的博客或者文章，都是之前的老知识点或者直接搬了别人的老文章。

所以，大多数认为就是 **MDN** 上描述的那样了。闭包是指那些能够访问 **自由变量** 的 **函数**。

首先，看个例子：

```javascript
function fun(){
  let arr = Array(10000000)
  function fun1(){// arr 加入 Closure
    console.log(arr)
  }
  return function fun2(){
  }
}
window.f = fun()// 长久持有fun2的引用
```

上述代码会形成闭包，然后发生了内存泄漏，如何解释？



#### 回忆作用域链`[[Scopes]]`

作用域链是执行上下文的3个重要属性之一（其他2个分别是**变量对象**和**this**），

**VO**和**AO**都是在 **预编译** 的时候创建的，然后在 代码执行 的时候去修改。

> 注意⚠️：
>
> 1. 这里的VO和AO，是es3中存在的概念，在现在的es5中已经不存在这些概念了。取而代之的是 词法环境（Lexical environment）；
>
> 2. 这里的预编译指的是函数开始执行前的那一刻所做的事情，将一些声明放到当前的词法环境中，在代码执行阶段进行赋值。

每一个函数都有一个 **[[Scopes]]** 属性，其存储的是函数运行时的作用域链。

> 更正下⚠️：
>
> 之前分享过的，**[Scopes]]** 属性是会把自己的AO放到最前面，父级的在后面按照词法依次列出。
>
> 应该是，函数预编译时，会先把当前的词法环境LE放到最前面，作用域链的其他部分会在其父函数的预编译时添加到 **[Scopes]]**  的后面。

```javascript
// 1: 全局词法环境global.LE = {t,fun}
let t = 111
function fun(){
    // 3: fun.LE = {a,b,fun1}
    let a = 1
    let b = 2
    function fun1() {
        // 5: fun1.LE = {c}
        let c = 3
    }
    // 4: fun1.[[Scopes]] = [fun.LE, global.LE]
}
// 2: fun.[[Scopes]] = [global.LE]
fun()
```

`[[Scopes]]` 就像一个数组一样，每一个函数的 `[[Scopes]]` 中都存在当前函数的 LE 和上级函数的 `[[Scopes]]`。在函数运行时会优先取距离当前函数 LE 近的变量值，这就是作用域的就近原则。

**但是，对于[[Scopes]]的解释 又不完全是这样的。**

其实，

每一个 **词法作用域** 都会有一个 **outer** 属性指向其上级 **词法作用域** ，根据 **outer** 属性完全可以构造一个作用域链了，但是 这时候还有一个属性 **Closure**， 这个闭包属性是用来干什么的呢？

这就是涉及到闭包和内存泄漏问题，如果单纯的通过 **outer** 链路来实现作用域链，那么存在一个闭包时，就会导致整个作用域链中的所有 **词法环境** 都无法回收，但是此时如果我们只使用了父级词法环境中的一个变量，而 V8 为了让我们能使用这个一个变量付出如此大的内存代价，很显然是不值得的。而 `[[Scopes]]` + `Closure` 就是他们的解决方案。

所以现在的 V8 中已经发生了改变（Chrome 中已经可以看到这些变化），在为一个函数绑定**词法作用域**时，并不会粗暴的直接把父函数的 LE 放入其 `[[Scopes]]` 中，而是会分析这个函数中会使用父函数的 LE 中的哪些变量，而这些可能会被使用到的变量会被存储在一个叫做 `Closure` 的对象中，**每一个函数都有且只有一个 `Closure` 对象** ，最终这个 `Closure `将会代替父函数的 LE 出现在子函数的 `[[Scopes]]` 中。

> 所以，V8引擎中的 作用域链，并不是单纯存LE构成链条了，而是会把 Closure 也会存在其中。



#### 闭包对象 Closure

在V8中每一个函数执行前都会进行预编译，预编译阶段都会执行3个重要的步骤，

1. CreateFunctionContext 创建函数执行上下文
2. PushContext 上下文入栈
3. **CreateClosure 创建函数的闭包对象**

也就说，每个函数创建的时候，都会创建一个该函数的闭包对象，无论这个闭包是否被使用。

`Closure` 跟 `[[Scopes]]` 一样会在函数预编译时被确定，区别是当前函数的 `[[Scopes]]` 是在其父函数预编译时确定， 而 `Closure` 是在当前函数预编译时确定（在当前函数执行上下文创建完成入栈后就开始创建闭包对象了）。

```javascript
function f1() {
  let a = 1
  function f2() {
    console.log(a)
  }
}
f1()
// f2的[[Scopes]]是在函数f1被预编译的时候就已经被同时创建了，
// 但是f2的 Closure ，是在自身预编译开始的时候才被创建，
// 时机不同。
```

当 V8 预编译一个函数时，如果遇到内部函数的定义不会选择跳过，而是会快速的扫描这个内部函数中使用到的本函数 LE 中的变量，然后将这些变量的引用加入 `Closure` 对象。再来为这个内部函数函数绑定 `[[Scopes]]` ，并且使用当前函数的 `Closure` 作为内部函数 `[[Scopes]]` 的一部分。

> 注意：每一次遇到内部声明的函数/方法时都会这么做，无论其内部函数/方法的声明嵌套有多深，并且他们使用的都是同一个 `Closure` 对象。并且这个过程 **是在预编译时进行的而不是在函数运行时**。

```javascript
// 1: global.LE = {t,fun}
var t = 111
// 2: fun.[[Scopes]] = [global.LE]
function fun(){
    // 3: fun.LE = {a,b,c,fun1,obj}，并创建一个空的闭包对象fun.Closure = {}
    let a = 1,b = 2,c = 3
    // 4: 遇到函数，解析到函数会使用a，所以 fun.Closure={a:1} (实际没这么简单)
    // 5: fun1.[[Scopes]] = [global.LE, fun.Closure]
    function fun1() {
        debugger
        console.log(a)
    }
    fun1()
    let obj = {
        // 6: 遇到函数，解析到函数会使用b，所以 fun.Closure={a:1,b:2}
        // 7: method.[[Scopes]] = [global.LE, fun.Closure]
        method(){
            console.log(b)
        }
    }
}

// 执行到这里时，预编译 fun
fun()
```

结论：每一个函数都会产生闭包，无论 **闭包中是否存在内部函数** 或者 **内部函数中是否访问了当前函数变量** 又或者 **是否返回了内部函数**，因为闭包在当前函数预编译阶段就已经创建了。

####  内存泄漏

我们知道，闭包使用不当的缺点就是容易造成 **内存泄漏**。

所谓闭包产生的内存泄漏就是因为闭包对象 `Closure` 无法被释放回收，那么什么情况下 `Closure` 才会被回收呢？

这当然是在没有任何地方引用 `Closure` 的时候，因为 `Closure` 会被所有的子函数的作用域链 `[[Scopes]]` 引用，所以想要 `Closure` 不被引用就需要所有子函数都被销毁，从而导致所有子函数的 `[[Scopes]]` 被销毁，然后 `Closure` 才会被销毁。

最开始的例子：

```javascript
function fun(){
    let arr = Array(10000000)
    let a = 2
    function fun1(){// arr 加入 Closure
        console.log(arr)
    }
    return function fun2(){}
}
window.f = fun()// 长久持有fun2的引用
```

因为 `fun1` 让 `arr` 加入了 `fun.Closure`，`fun2` 又被 `window.f` 持有引用无法释放，因为 `fun2` 的作用域链同样包含 `fun.Closure`，所以 `fun.Closure `也无法释放，最终导致 `arr` 无法释放产生内存泄漏。



再看一个例子：

```javascript
let theThing = null;
let replaceThing = function () {
    let leak = theThing;
    function unused () { 
        if (leak){}
    };

    theThing = {  
        longStr: new Array(1000000),
        someMethod: function () {  
                                   
        }
    };
  	// leak = null 
};

let index = 100;
while(index--){
    replaceThing()
}
```

上述代码，会造成内存泄漏。

因为 unused函数使用了leak变量，所以，导致replaceThing的闭包对象中包含了leak，而leak指向theThing，replaceThing函数最后把一个大对象赋值给theThing，其中someMethod方法的[[Scopes]]同样包含了replaceThing的Closure未被销毁，theThing一直没被销毁，这样就导致了闭包对象随着循环越变越大，导致内存泄漏。





### 总结

1. 任何函数在执行之前都会进行预编译，预编译会创建一个空的闭包对象；

2. 每当这个函数预编译时遇到其内部的函数声明时，会快速的扫描内部函数使用了当前函数中的哪些变量，将可能使用到的变量加入到闭包对象中，最终这个闭包对象将作为这些内部函数作用域链中的一员；

3. 只有所有内部函数的作用域链都被释放才会释放当前函数的闭包对象，所谓的闭包内存泄漏也就是因为闭包对象无法释放产生的；

4. 为什么就算创建闭包的函数的上下文被销毁掉了，闭包函数还能正常使用，因为这个闭包函数保存了作用域scopes的关系链，这个链条还会保存在内存中不被销毁。可以参考下述代码：

   ``` javascript
   function foo () {
     var str = 123
     function bar(){
       console.log(str)
     }
     return bar
   }
   foo()()  // 123
   
   
   // 按照执行上下文栈的逻辑：
   // 执行到foo函数的时候，执行代码记录栈里面已经有全局执行上下文，和foo函数上下文了
   // 这时候return bar后，foo函数执行完毕，会把当前foo的执行上下文弹出记录栈
   // 开始执行bar函数，将bar函数推进执行上下文推进栈的前面
   // 执行结束，栈弹出
   
   // 按照这个逻辑foo函数执行上下文已经被销毁了，但是在bar中依然能访问foo的变量对象，这说明作用域链条不会
   // 随着执行上下文的销毁而销毁，而是会保存在内存中
   ```

   













https://juejin.cn/post/7079995358624874509