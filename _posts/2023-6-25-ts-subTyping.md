---
layout: post
title: ts类型系统-逆变、协变、双变和不变
tags: ts 类型
categories: typescript
---

* TOC 
{:toc}

## 定义
**「Subtyping」** 是一个面向对象编程中非常重要的概念，用于描述一个类型和另一个类型之间的关系。

一个类型可以被看作是另一个类型的字类型，即该类型可以替代父类型的使用。

具体来说，如果类型A是类型B的字类型，那么可以将A类型的值赋给B类型的变量。

```ts
interface Animal {
  name: string
}
interface Dog extends Animal {
  bark: boolean
}
let animal: Animal = { name: 'A' }
let dog: Dog = {name: 'D', bark: true}

animal = dog // ok
dog = animal // error
```

TypeScript 给 JavaScript 添加了一套静态类型系统，是为了保证类型安全的，也就是保证变量只能赋同类型的值，对象只能访问它有的属性、方法。

但是这种类型安全的限制也不能太死板，有的时候需要一些变通，比如子类型是可以赋值给父类型的变量的，可以完全当成父类型来使用，也就是“型变”（类型改变）。


## 协变（Covariant）
在类型系统中，如果某个类型构造器是`T`，满足 `A <: B`，并且 `T<A> <: T<B>`的话，则说明T类型系统是协变。

例如：
```typescript
type List<T> = T[]
interface Person {
  name: string
}
interface Women extends Person {
  shopping: boolean
}
type t1 = Women extends Person ? true : false
type t2 = List<Women> extends List<Person> ? true : false
type t3 = List<Person> extends List<Women> ? true : false

```
像类似这种的协变构造器，还有 `Record<T,U>`、`Promise<T>`等等。

## 逆变（Contravariant）
在类型系统中，如果某个类型构造器是`T`，满足 `A <: B`，并且 `T<B> <: T<A>`的话，则说明T类型系统是逆变。

```typescript 
type PrintFeature<T> = (arg: T) => void
interface Person {
  name: string
}
interface Women extends Person {
  shopping: boolean
}

let printPerson: PrintFeature<Person> = (person: Person) => {
    console.log(person.name)
}
let printWomen: PrintFeature<Women> = (women: Women) => {
    console.log(`${women.name} ${women.shopping ? 'like shopping' : ''}`)
}

// Women <: Person，但是 PrintFeature<Person> <: PrintFeature<Women>，说明PrintFeature类型方法是属于逆变。
printPerson = printWomen // 报错了

printWomen = printPerson
```

由此可以得出结论，**如果类型构造器被用于在函数参数，那么将会产生逆变。**

那么 **如果类型构造器被用于函数的返回值呢？**，答案是 **会产生协变。**，如下例子：

```ts
type ReturnSelf<T> = () => T
interface Person {
  name: string
}
interface Women extends Person {
  shopping: boolean
}

type t1 = ReturnSelf<Women> extends ReturnSelf<Person> ? true : false // true
```

## 双变（Bivariant）
在类型系统中，如果某个类型构造器是`T`，让类型之间的关系既可以发生协变，又可以产生逆变，就可以被称为双变。

在 typescript 2.6 版本，新增了“strictFunctionTypes” 编译器配置项。如果没有开启，上述作为参数的逆变代码将不会报错。

ts默认是开启双变的，至于原因官方有声明，[查看说明](https://github.com/microsoft/TypeScript/wiki/FAQ#why-are-function-parameters-bivariant)，总结就是：TS 的解释为，我们在声明某个方法时可能会使用更广泛的类型，但是实际使用的时候则会传入更详细的子类型，比如：

```ts
interface MyEvent{
    type: string
}

interface MyMouseEvent extends MyEvent{
    x: number,
    y: number
}

declare function on(eventName: string, callBack: (event: MyEvent) => void): void
on('mouse', (event: MyMouseEvent) => console.info(`${event.x} : ${event.y}`))
```
上述代码，在执行 `on` 方法的时候，会将参数进行赋值操作，即
 ```ts
 callBack = (event: MyMouseEvent) => console.info(`${event.x} : ${event.y}`)
 ```
 而 `MyMouseEvent <: MyEvent`，这种情况产生协变规则才能类型匹配，但是构造器类型作用于函数参数，此时产生的是逆变。

又例如，array 的 forEach 方法本身定义为:
```ts
forEach((element, index, array) => { /* ... */ })
```
当然也可以传递只有一个参数的函数如:
```ts
forEach(x => console.info(x))
```

## 不变（Invariant）
所谓的不变是指，某个类型构造器T，让类型之间的关系既不产生协变又不产生逆变。

例如：
```ts
type PrintFeature<T> = (arg: T) => T
interface Person {
  name: string
}
interface Women extends Person {
  shopping: boolean
}

let printPerson: PrintFeature<Person> = (person: Person) => {
    return person
}
let printWomen: PrintFeature<Women> = (women: Women) => {
    return women
}

printPerson = printWomen // 报错了，参数类型逆变检查不通过

printWomen = printPerson // 报错了，返回值类型协变检查不通过
```

## 总结
对于函数类型的构造器来说，类型参数的位置决定了产生何种变化。在类型系统中，某个类型构造器T，如果它保留了类型之间的关系，则是协变。如果它使得类型之间的关系产生了逆转就是逆变。

## 逆变协变对 infer 的影响
简单一句话来说就是 infer 处于逆变位置推断类型为交叉类型，处于协变位置推断出类型为联合类型:

```ts
interface Foo{
    bar1(name: string): string,
    bar2(name: number): number
}

type BAR<T> = T extends {
    bar1: (name: infer R) => infer Y,
    bar2: (name: infer R) => infer Y,
} ? [R, Y] : never

type BBB = BAR<Foo> // [never, string|number]. 第一个 never 是因为 string & number = never
```


## 练习

1. 函数参数逆变 配合 extends、infer 的将联合类型转化成交叉类型([https://github.com/type-challenges/type-challenges/issues/122](https://github.com/type-challenges/type-challenges/issues/122))

```ts
type UnionType = "AA" | "BB" |  "CC";
// 目标是将上面类型转换成 "AA" & "BB" &  "CC"
type UnionToIntersection<U> = 
    (U extends unknown ? (arg: U) => unknown : never) extends ((arg: infer P) => unknown) ? P : never;
  
type IntersectionType = UnionToIntersection<UnionType>
```