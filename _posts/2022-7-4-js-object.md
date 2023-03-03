---
layout: post
title: js创建对象的多种方式
tags: js 原型 对象
categories: jsBase
---

* TOC 
{:toc}

#### 1. 工厂模式

```javascript
function createPerson(name) {
  var o = new Object()
  0.name = name
  return o
}
var person1 = createPerson('xwk')
```

缺点：创造出来的对象无法识别，因为都指向同一个原型。

#### 2. 构造函数模式

```javascript
function Person(name) {
	this.name = name
  this.getName = function() {
    console.log(this.name)
  }
}
var person1 = new Person('xwk')
```

优点：实例可以识别为一个特定的类型

缺点：每次创建实例的时候，每个方法都要被创建一次

#### 3. 原型模式

```javascript
function Person(name) {}
Person.prototype.name = 'xwk'
Person.prototype.getName = function () {
  console.log(this.name)
}
var person1 = new Person()
```

优点：方法是共用了，不会重复创建

缺点：1.没办法初始化参数 2. 所有的属性和方法都共享了

#### 3.1 原型模式优化

```javascript
function Person(name) {}
Person.prototype = {
  name: 'xwk',
  getName: function () {
    console.log(this.name)
  },
  constructor: Person
}
var person1 = new Person()
```

优点：封装性好了点

缺点：原型模式该有的缺点还是有

#### 4. 组合模式

构造函数模式 + 原型模式

共享的写在原型里，独立的写在构造函数中。

```javascript
function Person(name) {
  this.name = name
}
Person.prototype = {
    constructor: Person,
    getName: function () {
        console.log(this.name);
    }
};

var person1 = new Person();
```

优点：这种方式使用更广泛

#### 4.1 动态原型模式

```javascript
function Person(name) {
    this.name = name;
    if (typeof this.getName != "function") {
        Person.prototype.getName = function () {
            console.log(this.name);
        }
    }
}

var person1 = new Person();
```

但是注意⚠️：使用动态原型模式的时候，不能对面量重写原型

解释下为什么：

```javascript
function Person(name) {
    this.name = name;
    if (typeof this.getName != "function") {
        Person.prototype = {
            constructor: Person,
            getName: function () {
                console.log(this.name);
            }
        }
    }
}

var person1 = new Person('kevin');
var person2 = new Person('daisy');

// 报错 并没有该方法
person1.getName();

// 注释掉上面的代码，这句是可以执行的。
person2.getName();

```

这是因为： var person1 = new Person('kevin') ，在执行这段代码的时候，new的过程分解如下，首先把 ```person1.__proto__``` 指向的是Person.prototype，我们可以认为是O1，其次再执行构造函数，发现没有getName，这时候把 Person.prototype 重新给复值给了一个新对象，我们认为是O2，但是此时person1的原型指向的是O1并不是O2，O1中并没有getName方法，person2的原型指向的是O2，所以可以正确执行。
可以修改代码：

 ```javascript
function Person(name) {
    this.name = name;
    if (typeof this.getName != "function") {
        Person.prototype = {
            constructor: Person,
            getName: function () {
                console.log(this.name);
            }
        }
        return new Person(name)  // 修改方案一：首先实例化一次
      	// 修改方案二：
				// 或者不要直接用字面量的方式赋值原型，修改之前的原型
				// Person.prototype.getName = function() {console.log(this.name)}
    }
}

var person1 = new Person('kevin');
var person2 = new Person('daisy');

person1.getName();

person2.getName();
 ```

#### 5.1 寄生构造函数模式

所谓的寄生构造函数模式就是比工厂模式在创建对象的时候，多使用了一个new，实际上两者的结果是一样的。

但是作者可能是希望能像使用普通 Array 一样使用 SpecialArray，虽然把 SpecialArray 当成函数也一样能用，但是这并不是作者的本意，也变得不优雅。

```javascript
function SpecialArray() {
    var values = new Array();

    for (var i = 0, len = arguments.length; i < len; i++) {
        values.push(arguments[i]);
    }

    values.toPipedString = function () {
        return this.join("|");
    };
    return values;
}

var colors = new SpecialArray('red', 'blue', 'green');
var colors2 = SpecialArray('red2', 'blue2', 'green2');


console.log(colors);
console.log(colors.toPipedString()); // red|blue|green

console.log(colors2);
```

#### 5.2 稳妥构造函数模式

```javascript
function person(name){
    var o = new Object();
    o.sayName = function(){
        console.log(name);
    };
    return o;
}

var person1 = person('kevin');

person1.sayName(); // kevin

person1.name = "daisy";

person1.sayName(); // kevin

console.log(person1.name); // daisy
```

所谓稳妥对象，指的是没有公共属性，而且其方法也不引用 this 的对象。

与寄生构造函数模式有两点不同：

1. 新创建的实例方法不引用 this
2. 不使用 new 操作符调用构造函数

稳妥对象最适合在一些安全的环境中。

稳妥构造函数模式也跟工厂模式一样，无法识别对象所属类型。
