---
layout: post
title: 设计模式-命令模式
tags: 设计模式 ts
categories: DesignPattern
---

* TOC 
{:toc}

### 定义

命令模式(Command Pattern)是一种行为型设计模式，是一个高内聚的模式，它将一个请求封装成一个对象，从而使你可以用不同的请求对客户端参数化。命令模式可以让请求者和接收者解耦，并支持撤销操作。

在命令模式中，通常会有四个角色：

1.命令(Command)：封装了一个请求及其相关的操作。

2.具体命令(Concrete Command)：实现了命令接口，负责执行具体的请求操作。

3.接收者(Receiver)：负责执行请求操作。

4.调用者(Invoker)：负责调用命令对象来执行请求操作，并可以支持撤销操作。

### 命令模式的优点

1. 可以将命令对象与调用者解耦，使得调用者不需要知道具体的请求操作是如何实现的。
2. 可以将请求对象与接收者解耦，使得接收者不需要知道具体的请求对象是如何创建和执行的。
3. 支持撤销操作，因为命令对象通常会存储执行操作所需的状态信息。
4. 可以方便地增加新的命令对象，因为每个命令对象都实现了命令接口。

### 命令模式的缺点

1. 可能会导致系统中出现大量的具体命令类。
2. 增加了系统的复杂度和开销，因为需要多个对象来实现一个请求操作。
3. 可能会影响系统的性能，因为需要多次调用对象的方法。

### 举个例子

如果想实现一个购物车功能，有添加产品、删除产品、更改产品等功能。就比较适用于命令模式了。

#### 1. 命令四角色 ---命令接口：

```typescript
interface ICommand {
  execute(): void;
  undo(): void;
}
```

#### 2. 命令四角色 --- 实现命令接口：(ConcreteCommand)：

```typescript
class AddProductCommand implements ICommand {
  constructor(private product: Product, private shoppingCart: ShoppingCart){}

  execute() {
    this.shoppingCart.addProduct(this.product)
  }
  undo() {
    this.shoppingCart.removeProduct(this.product)
  }
}
class RemoveProductCommand implements ICommand {
  constructor(private product: Product, private shoppingCart: ShoppingCart){}
  execute() {
    this.shoppingCart.removeProduct(this.product) 
  }
  undo() {
    this.shoppingCart.addProduct(this.product)
  }
}
class ChangeProductQuantityCommand implements ICommand {
  private readonly oldQuantity: number;
  constructor(private product: Product, private shoppingCart: ShoppingCart, private newQuantity: number){
    this.oldQuantity = this.product.quantity
  } 
  execute() {
    this.product.quantity = this.newQuantity
  }
  undo() {
    this.product.quantity = this.oldQuantity
  }
}
```

#### 3. 命令四角色 --- 接收者(Receiver)：

```typescript
class Product {
  constructor(public readonly id: number, public readonly name: string, public readonly price: number, public  quantity: number) {}
  get total() {
    return this.price * this.quantity
  }
}
class ShoppingCart {
  private readonly products: Product[] = []
  addProduct(product: Product) {
    const existingProduct = this.products.find(item => item.id === product.id)
    if (existingProduct) {
      existingProduct.quantity ++
    } else {
      this.products.push(product)
    }
  }
  removeProduct(product: Product) {
    const index = this.products.findIndex(p => p.id === product.id);
    if (index >= 0) {
      const finded = this.products[index]
      if (finded.quantity === 1) {
        this.products.splice(index, 1)
      } else {
        finded.quantity -- 
      }
    }
  }
  getTotalPrice() {
    return this.products.reduce((total, product) => total + product.total , 0)
  }
  print() {
    this.products.forEach(item => {
      console.log(`${item.name}: 单价是${item.price}，数量是${item.quantity}， 共${item.total}钱`)
    })
  }
}
```

#### 3. 命令四角色 --- 调用者(Invoker)：

```typescript
class Invoker {
  private readonly commands:ICommand[] = []
  clear() {
    this.commands.length = 0
  }
  addCommand(command: ICommand) {
    this.commands.push(command)
    return this
  }
  executeCommands() {
    this.commands.forEach(command => command.execute());
  }
  undoCommands() {
    this.commands.slice().reverse().forEach(command => command.undo());
  }
}
```

最后调用代码如下，

```typescript

const shoppingCart = new ShoppingCart()
const product1 = new Product(1, 'apple', 30, 2) 
const product2 = new Product(2, 'banana', 10, 3) 
const product3 = new Product(3, 'pie', 50, 1) 

// 添加商品
const addProductCommand1 = new AddProductCommand(product1, shoppingCart)
const addProductCommand2 = new AddProductCommand(product2, shoppingCart)
const addProductCommand3 = new AddProductCommand(product3, shoppingCart)
// 删除商品
const removeProductCommand = new RemoveProductCommand(product2, shoppingCart)
// 修改数量
const changeProductQuantityCommand = new ChangeProductQuantityCommand(product3, shoppingCart, 20)


const invoke = new Invoker()
invoke.addCommand(addProductCommand1).addCommand(addProductCommand2).addCommand(addProductCommand3)
invoke.executeCommands()
shoppingCart.print()

invoke.clear()
invoke.addCommand(removeProductCommand)
invoke.executeCommands()
shoppingCart.print()

invoke.clear()
invoke.addCommand(changeProductQuantityCommand)
invoke.executeCommands()
shoppingCart.print()

invoke.clear()
invoke.addCommand(changeProductQuantityCommand)
invoke.undoCommands()
shoppingCart.print()
```


### 总结

上面那个购物车例子，定义一个公用的命令接口类型，每个命令都有一个execute和undo方法，然后基于这个类型去实现不同的命令。
命令的接收者就是ShoppingCart类，接收者负责命令的具体实现逻辑。
如何去调用命令呢，就是Invoker类，起到一个调用命令的作用。