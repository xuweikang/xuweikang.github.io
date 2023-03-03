---
layout: post
title: 分析js代码的运行顺序
tags: js this 作用域 执行上下文 AO VO 作用域链
categories: jsBase
---

* TOC 
{:toc}

####  1. 下面的问题你能准确回答出来么？

- 词法作用域是什么？动态作用域是什么？
- js代码是一行一行运行的么？
- 什么是执行上下文？执行上下文栈？
- 活动对象（AO）？变量对象（VO）？作用域链？



#### 2. 静态作用域和动态作用域

```javascript
var value = 1;

function foo() {
    console.log(value);
}

function bar() {
    var value = 2;
    foo();
}

bar();
// 结果是 ???
```



**作用域**：源代码中定义变量的区域，决定了如何查找变量和对变量的访问权限。

**静态作用域（词法作用域）**：函数的作用域在定义的时候就已经确定了的，对于JS而言，其实就是函数定义的时候会给函数绑定一个[[scope]]内部属性，这个属性指向父作用域（所有的父级变量对象）。

**动态作用域**：函数的作用域在函数执行的时候确定的。

上述代码的执行结果是1，很显然，js这门语言采用的是词法作用域。函数foo在声明的时候就已经决定了他的作用域，执行foo函数，先从foo内部查找有没有value变量，如果没有，就根据书写的位置查找上一级的代码，找到value 的定义。

那有什么语言是采用动态作用域的呢？当然有，bash就是动态作用域。

```bash
value=1
function foo () {
    echo $value;
}
function bar () {
    local value=2;
    foo;
}
bar  // 2
```



#### 3.  执行上下文栈

##### 顺序执行？

```javascript
function foo() {
    console.log('foo1');
}
foo();  // foo2
function foo() {
    console.log('foo2');
}
foo(); // foo2
```

也许你可能会知道，这不就是js的函数提升么，js引擎会默认将变量或者函数声明的逻辑置前，这就是所谓的变量提升和函数提升，所以上述代码的结果就是打印的foo2，因为第一个foo声明会后面的给覆盖掉了。

那变量提升和函数提升的原因到底是什么呢？后文会解释原因。

至少从上述代码能看出，JS代码并非是一行一行执行的，而是按照某种顺序和规则进行划分的。

##### 可执行代码

在JavaScript中的可执行代码有哪些哪些呢？

其实就三种，全局代码、函数代码、eval代码。

其实就是在执行函数的时候，做的准备工作，叫做“执行上下文”。

##### 执行上下文栈

那如果“执行上下文”有很多，就必须有一个地方去管理这些上下文的顺序。

这个地方就是**执行上下文栈**（Execution context stack，ECS），

为了模拟栈行为，用一个数组来定义，

当JavaScript开始执行代码的时候，最先遇到的就是全局执行代码，所以栈底肯定是全局执行上下文，用globalContext表示。只有代码全部执行完毕后，ECStack才被清空。

```javascript
ECStack = [
    globalContext
];
```

当执行一个函数的时候，js就会创建一个上下文，并把这个上下文推到执行上下文栈，当函数执行完毕的时候，就将此上下文弹出，下述代码：

```javascript
function fun1(){
  console.log('f1')
}
function fun2() {
  fun1();
}
fun2();
```

js会这样处理执行上下文栈：

```javascript
// 伪代码
ECStack.push(<fun2>functionContext)
// [globalContext, <fun2>functionContext]
ECStack.push(<fun1>functionContext)
// [globalContext,<fun2>functionContext,<fun1>functionContext]
ECStack.pop() // [globalContext,<fun2>functionContext]
ECStack.pop() // [globalContext]

// 执行剩余代码
```

再看个例子：

```javascript
var scope = "global scope";
function checkscope(){
    var scope = "local scope";
    function f(){
        return scope;
    }
    return f;
}
checkscope()();
```

```
ECStack.push(<checkscope> functionContext);
ECStack.pop();
ECStack.push(<f> functionContext);
ECStack.pop();
```

#### 4. 变量对象

前面讲到，当执行一段可执行代码的时候，会创建一个对应的执行上下文，那这个“执行上下文”究竟有哪些属性呢？

对于每个执行上下文，都有三个重要属性：

- 变量对象（Variable object，VO）
- 作用域链（Scope chain）
- this

不同执行上下文下的变量对象也并不相同。

#### 全局上下文

全局对象是什么，w3c的介绍如下：

> 全局对象是预定义的对象，作为 JavaScript 的全局函数和全局属性的占位符。通过使用全局对象，可以访问所有其他所有预定义的对象、函数和属性。

> 在顶层 JavaScript 代码中，可以用关键字 this 引用全局对象。因为全局对象是作用域链的头，这意味着所有非限定性的变量和函数名都会作为该对象的属性来查询。

> 例如，当JavaScript 代码引用 parseInt() 函数时，它引用的是全局对象的 parseInt 属性。全局对象是作用域链的头，还意味着在顶层 JavaScript 代码中声明的所有变量都将成为全局对象的属性。

简单点来说，全局对象在浏览器环境中其实可以认为就是window对象，在node环境中就是globalThis。

全局上下文的变量对象，就是全局对象。

#### **函数上下文**

在函数上下文中，变量对象稍微复杂点。用 **活动对象**（activation object, AO） 来表示**变量对象**（VO）。

**变量对象VO **是引擎实现的，不可在js环境中被访问，只有当到达一个可执行上下文时，这个变量对象才会被激活成 **AO** ，只有被激活的变量对象也就是活动对象，上面的各种属性才能够被访问。

活动对象是在进入函数上下文被创建的，通过**arguments**属性初始化。

#### 执行过程

执行上下文的代码分为两个阶段进行处理：

1. 进入执行上下文
2. 代码执行

##### a. 进入执行上下文

当进入执行上下文时，这时候还没有执行代码，

变量对象包括：

1.  函数的所有形参和值（如果是函数上下文）
2.  函数声明
3.  变量声明

举个例子：

```javascript
function foo(a) {
  var b = 2;
  function c() {}
  b = 3;
}
foo(1);
```

当执行foo，函数进入执行上下文，函数的变量对象被激活成活动对象，此时的AO表示是：

```javascript
AO = {
  arguments: {
    0: 1,
    length: 1
  },
  a: 1,
  b: undefined,
  c: function c(){}
}
```

##### b. 代码执行

在代码执行阶段，会从上往下顺序执行代码，然后去修改变量对象的值。

上面的代码在第二阶段代码执行后，AO如下：

``` javascript
AO = {
  arguments: {
    0: 1,
    length: 1
  },
  a: 1,
  b: 3,
  c: function c(){}
}
```

关于变量对象的概念，总结如下：

1. 全局上下文的变量对象是全局对象；
2. 函数上下文的变量对象初始值只有Arguments对象，在进入上下文后，会添加形参、函数声明、变量声明等属性，代码执行阶段会修改属性值。

这里其实也解释了前面的“函数提升”“变量提升”的问题，

进入到上下文的代码，首先第一步会将变量对象的属性进行声明和初始化，第二步才是执行并修改。

下面再看个例子：

```javascript
console.log(foo);  // ???

function foo(){
    console.log("foo");
}

var foo = 1;

/**  
  结果显然打印1，
  从执行上下文的角度分析这题，发现并没那么简单。
  var foo = 1;
  console.log(foo);  // ???
  function foo(){
      console.log("foo");
  }
*/


```

第一段代码，打印出来的结果是函数foo，原因如下：

如果在进入到执行上下文时，出现了变量和函数同名的话，以函数为主，和顺序无关。因为函数是js的一等公民！

如果 “ var foo = 1;  ” 在 “ console.log(foo); ” 的上面呢，打印结果将变化。

##### c.  AO和VO之间到底有什么联系？

网上有的文章认为，AO其实就是VO，只不过是不同时期的不同表现而已。

但是查找资料后发现，其实这种说法并不准确，准确的来说，AO包含VO，因为AO中除了VO以外，还有parameters（形参），arguments对象。AO在VO进行到执行阶段的时候被激活，但是激活的除了VO以外，还有parameters和arguments。

**AO = VO + function parameters + arguments**



#### 5. 作用域链

一个函数执行，是如何通过AO，还有执行上下文栈这些来准确查找到变量的呢？

或者说，js代码执行里面的变量查找，究竟是通过什么来查找的？

仅仅通过执行执行上下文栈，变量对象，作用域，这些知识还是无法解释。

前面讲过，作用域其实就是定义了一个变量定义和查找的地方，那如果当前作用域没有该变量，那他该以什么来继续查找，这个其实就是 **作用域链**。

**定义：**js代码当在查找变量的时候，先会从当前上下文的变量对象中去找，如果找不到，就会从父级（词法层面的父级）的变量对象中去查找，一直找到全局对象，这样由多个执行上下文变量对象构成的链表，就是 **作用域链**。

#### a. 函数创建

​	函数有一个内部属性[[scope]]，当函数创建的时候，这个属性就被赋值为所有父亲变量对象的层级链。 

``` javascript
function foo() {
    function bar() {
        ...
    }
}
```

函数创建时候，各自的[[scope]]属性如下：

```javascript
// 伪代码
foo.[[scope]] = [ globalContext.VO ]
bar.[[scope]] = [ fooContext.VO, globalContext.VO ]
```

#### b. 函数激活

​	当函数激活时，进入函数上下文，创建VO后，就会将活动对象添加到作用域链的前端。

``` javascript
Scope = [AO].concat([[Scope]]);
```



可以看下之前的例子，

``` javascript
var scope = "global scope";
function checkscope(){
    var scope2 = 'local scope';
    return scope2;
}
checkscope();
```

执行过程如下：

1. checkscope函数被创建，绑定[[scope]]内部属性，globalContext.VO被保存到其中

   ``` javascript
   checkscope.[[scope]] = [globalContext.VO]
   ```

2. 执行checkscope函数，创建函数执行上下文

   ```javascript
   ECStack = [ checkscopeContext, globalContext ]
   ```

3. checkscope函数不会立即执行，先进入执行上下文，复制[[scope]]属性

   ``` javascript
   checkscopeContext = {
       Scope: checkscope.[[scope]],
   }
   ```

4. 初始化AO

   ```
   checkscopeContext = {
     AO: {
       arguements: { length: 0 },
       scope2: undefined
     },
     Scope: checkscope.[[scope]]
   }
   ```

5. 将活动对象AO压入checkscope作用域链顶端

   ```javascript
   checkscopeContext = {
     AO: {
       arguements: { length: 0 },
       scope2: undefined
     },
     Scope: [AO,[[Scope]]]
   }
   ```

6. 开始执行函数，修改AO属性值

   ```javascript
   checkscopeContext = {
       AO: {
           arguments: {
               length: 0
           },
           scope2: 'local scope'
       },
       Scope: [AO, [[Scope]]]
   }
   ```

7. 查找到scope2的值，返回后函数执行完毕，将函数上下文从执行上下文栈中弹出

   ```javascript
   ECStack = [
       globalContext
   ];
   ```







###  总结：

1.  js代码并非从上往下一句一句执行的，而是“一段一段”执行的，这里的段，我理解为不同的可执行记录栈中的栈元素。比如全局上下文可执行代码和函数上下文执行代码。
2.  代码执行到可执行代码时候，会形成一个执行上下文，执行上下文中的VO/AO中记录了当前上下文的变量和函数声明。
3.  js在执行一个函数时，并非直接执行，而是先进入到当前函数上下文，做些准备的声明工作以后，再开始执行函数内代码。
4.  由多个不同层级执行上下文的变量对象，所串起来的链表，就叫做作用域链。
5.  在查找变量的时候，会优先从当前的活动对象中，或者说作用域链的最前端中去查找，如果找不到，会沿着作用域链往上层的变量对象中去查找，一直找到全局对象。