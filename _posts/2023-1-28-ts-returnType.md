---
layout: post
title: ts实现精确的返回类型
tags: ts 精确的返回类型
categories: typescript
---

* TOC 
{:toc}


> **使用 ts 实现一个 zip 函数，对两个数组的元素按顺序两两合并，比如输入[1,2,3],[4,5,6]时，返回[[1,4],[2,5],[3,6]]。要求使用ts实现，并且要求用类型编程实现精确的类型提示，比如参数输入[1,2,3],[4,5,6]，那返回值的类型要提示出[[1,4],[2,5],[3,6]]**

![image.png](/static/img/ts/demo1.png)


上述题目如果要用js去解，或者ts去解但是没有后半部分的精确提示要求，其实是很简单的，我们可以很快写出对应代码。

### step1. ts简单实现前半部分要求

```ts
// 第一种：function函数声明
function zip(target: unknown[], source: unknown[]) {
  if (!target.length || !source.length) return []
  const [one, ...rest1] = target
  const [other, ...rest2] = source
  return [[one,other], ...zip(rest1, rest2)]
}

    
    
// 第二种：匿名函数加函数的构造签名
interface Zip {
  (target: unknown[], source: unknown[]): unknown[]
}
const zip: Zip = (target: unknown[], source: unknown[]) => {
  if (!target.length || !source.length) return []
  const [one, ...rest1] = target
  const [other, ...rest2] = source
  return [[one,other], ...zip(rest1, rest2)]
}
```

### step2. 修改默认返回类型使其为精准类型

这里要求返回值类型是精确的，我们就要根据参数的类型来动态生成返回值类型。

也就是这样：
```ts
function zip<Target extends unknown[], Source extends unknown[]>(target: Target, source: Source): Zip<Target, Source> {
  if (!target.length || !source.length) return []
  const [one, ...rest1] = target
  const [other, ...rest2] = source
  return [[one,other], ...zip(rest1, rest2)]
}

type Zip<Target, Source> = {
  // 待补充
}
```

Zip类型里面的具体逻辑，也就是我们需要的精确返回类型，需要对传入进来的泛型参数里面的元素进行提取，然后组合成新类型返回。


提取元素可以用模式匹配的方式：
![image.png](/static/img/ts/demo2.png)

所以Zip类型可以这样定义：
```ts
type Zip<One extends unknown[], Other extends unknown[]> = 
  One extends [infer OneFirst, ...infer Rest1]
    ? Other extends [infer OtherFirst, ...infer Rest2] 
      ? [[OneFirst, OtherFirst], ...Zip<Rest1, Rest2>]
      : [] 
    : []
;
```

到此，已经实现了大部分想要的功能，代码如下：
```ts
function zip<Target extends unknown[], Source extends unknown[]>(target: Target, source: Source): Zip<Target, Source> {
  if (!target.length || !source.length) return []
  const [one, ...rest1] = target
  const [other, ...rest2] = source
  return [[one,other], ...zip(rest1, rest2)]
}

type Zip<One extends unknown[], Other extends unknown[]> = 
  One extends [infer OneFirst, ...infer Rest1]
    ? Other extends [infer OtherFirst, ...infer Rest2] 
      ? [[OneFirst, OtherFirst], ...Zip<Rest1, Rest2>]
      : [] 
    : []
;
zip([1,2,3], [5,6,7])
```

但是发现，函数仍然有类型错误提示，
![image.png](/static/img/ts/demo3.png)

这是因为声明函数的时候都不知道参数是啥，自然计算不出 Zip<Target, Source> 的值，所以这里会类型不匹配：


### step3. 解决Zip<Target, Source>类型不匹配的问题

在ts里，可以使用函数重载的方法解决类似这种问题。

完善代码如下：

```ts
function zip(target: unknown[], source: unknown[]): unknown[]
function zip<Target extends unknown[], Source extends unknown[]>(target: Target, source: Source): Zip<Target, Source>
function zip(target: unknown[], source: unknown[]) {
  if (!target.length || !source.length) return []
  const [one, ...rest1] = target
  const [other, ...rest2] = source
  return [[one,other], ...zip(rest1, rest2)]
}

type Zip<One extends unknown[], Other extends unknown[]> = 
  One extends [infer OneFirst, ...infer Rest1]
    ? Other extends [infer OtherFirst, ...infer Rest2] 
      ? [[OneFirst, OtherFirst], ...Zip<Rest1, Rest2>]
      : [] 
    : []
;
const res = zip([1,2,3], [5,6,7])
```

代码完善后，确实没有报错了，但是返回的ts类型好像不是我们想要的。
![image.png](/static/img/ts/demo4.png)

其实，
这时候匹配的函数类型是对的，只不过推导出的不是字面量类型。
这时候可以加个 as const。

> as const 的作用：
TS 3.4中引入 as const，被称为const断言。
const断言告诉编译器为表达式推断出它能推断出的最窄或最特定的类型。如果不使用它，编译器将使用其默认类型推断行为，这可能会导致更广泛或更一般的类型。

看下面这个例子，

```ts
const args = [8, 5];
// const args: number[]
const angle = Math.atan2(...args); // error! Expected 2 arguments, but got 0 or more.
console.log(angle);
```
以上代码在js环境中执行没有任何问题，但是在ts环境中，执行Math.atan2这块会报参数错误。
编译器看到const args = [8,5]；推断出args的类型是number[]，这是一个由0个或更多number类型的元素组成的可变数组。但是，Math.atan2方法，只需要两个数值参数，但是编译器只知道args是一个数字数组，所以在调用Math.atan2方法时编译器会报出一个错误。
这时候，如果用到as const，就会避免上述问题。
```ts
const args = [8, 5] as const;
// const args: readonly [8, 5]
const angle = Math.atan2(...args); 
console.log(angle);
```
使用const断言，将args断言成 readonly[8,5]类型的元组了，后面在执行Math.atan2这段代码的时候，编译器就能精确的推断出参数解构后是两个值的参数了。

再看一个例子，
```ts
let m = 100;
let n="abc";
let array = [m,n] ;
let f = array[0];
f = 2000;
f="aaa";
```
f可以被赋值为number和string任意类型都不会报错，但是其实我们想要的是f对应的是m的类型，也就是number类型，当他赋值成非number类型的时候应当会报错。
先解释下为啥不会报错，这是因为虽然m是number类型n是string类型，但是加到数组里后，
array的类型就变成了更为宽泛的 (number|string)[]了，所以后面数组元素重新赋值只要是string或者number类型的都会通过。
```ts
let m = 100;
let n="abc";
let array = [m,n] as const;
let f = array[0];
f = 2000;
f="aaa";
```
使用as const后就可以解决上述问题，因为会将array类型变成readonly的元数组类型，而元数组类型是能记录数组每个位置的类型的。


回到最开始的代码，要想使返回的ts类型不是更为宽泛的数组类型而是我们想要的具体类型，需要使用到as const。
```ts
function zip(target: unknown[], source: unknown[]): unknown[]
function zip<Target extends unknown[], Source extends unknown[]>(target: Target, source: Source): Zip<Target, Source>
function zip(target: unknown[], source: unknown[]) {
  if (!target.length || !source.length) return []
  const [one, ...rest1] = target
  const [other, ...rest2] = source
  return [[one,other], ...zip(rest1, rest2)]
}

type Zip<One extends unknown[], Other extends unknown[]> = 
  One extends [infer OneFirst, ...infer Rest1]
    ? Other extends [infer OtherFirst, ...infer Rest2] 
      ? [[OneFirst, OtherFirst], ...Zip<Rest1, Rest2>]
      : [] 
    : []
;
const res = zip([1,2,3] as const, [5,6,7] as const)
```

这时候又出现新的问题了，由于as const会将数组变成只读的数组类型，这样zip函数声明定义的参数类型就不匹配了。
修改代码如下，
```ts
function zip(target: unknown[], source: unknown[]): unknown[]
function zip<Target extends readonly unknown[], Source extends readonly unknown[]>(target: Target, source: Source): Zip<Target, Source>
function zip(target: unknown[], source: unknown[]) {
  if (!target.length || !source.length) return []
  const [one, ...rest1] = target
  const [other, ...rest2] = source
  return [[one,other], ...zip(rest1, rest2)]
}

type Zip<One extends unknown[], Other extends unknown[]> = 
  One extends [infer OneFirst, ...infer Rest1]
    ? Other extends [infer OtherFirst, ...infer Rest2] 
      ? [[OneFirst, OtherFirst], ...Zip<Rest1, Rest2>]
      : [] 
    : []
;
const res = zip([1,2,3] as const, [5,6,7] as const)
```
但这样 Zip 函数的类型又不匹配了。
难道要把所有用到这个类型的地方都加上 readonly 么？
不用，我们 readonly 的修饰去掉就行了。

### step4. 实现去除 readonly 的高级类型

Typescript 有内置的高级类型 Readonly：
```ts
type Readonly<T> = { readonly [P in keyof T]: T[p]; }
```
可以把索引类型的每个索引都加上 readonly 修饰：
![image.png](/static/img/ts/demo5.png)

但没有提供去掉 Readonly 修饰的高级类型，我们可以自己实现一下：
```ts
type Mutable<Obj> = {
    -readonly [k in keyof Obj]: Obj[k]
}
```

### step5. 最终

最终代码如下：
```ts
function zip(target: unknown[], source: unknown[]): unknown[]
function zip<Target extends readonly unknown[], Source extends readonly unknown[]>(target: Target, source: Source): Zip<Mutable<Target>, Mutable<Source>>
function zip(target: unknown[], source: unknown[]) {
  if (!target.length || !source.length) return []
  const [one, ...rest1] = target
  const [other, ...rest2] = source
  return [[one,other], ...zip(rest1, rest2)]
}

type Zip<One extends unknown[], Other extends unknown[]> = 
  One extends [infer OneFirst, ...infer Rest1]
    ? Other extends [infer OtherFirst, ...infer Rest2] 
      ? [[OneFirst, OtherFirst], ...Zip<Rest1, Rest2>]
      : [] 
    : []
;
type Mutable<Obj> = {
  -readonly [k in keyof Obj]: Obj[k]
}
const res = zip([1,2,3] as const, [5,6,7] as const)



```
