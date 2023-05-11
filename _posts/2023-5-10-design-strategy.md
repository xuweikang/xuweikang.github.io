---
layout: post
title: 设计模式-策略模式
tags: 设计模式 ts
categories: DesignPattern
---

* TOC 
{:toc}

### 定义

当我们需要在运行时根据不同的情境选择不同的算法或者策略时，策略模式就派上用场了。它通过定义一组算法族，将每个算法分别封装起来，并且可以相互替换使用，从而使得算法的变化不会影响到使用算法的客户端。

策略模式的核心思想是将算法的实现与客户端代码分离开来，通过策略接口来定义算法的公共接口，从而实现更加灵活和可扩展的算法组合方式。策略模式包含三个核心角色：策略接口、具体策略类和上下文类。其中，策略接口定义了所有具体策略类需要实现的公共接口，具体策略类实现策略接口，封装了具体的算法实现，上下文类包含一个指向策略接口的引用，客户端可以通过上下文类来使用具体的策略算法，同时也可以在运行时动态地切换算法策略。

### 策略模式的优点

1. 算法可以相互替换，使得代码更加灵活和可扩展。
2. 算法的实现与客户端代码分离，使得代码更加清晰和易于维护。
3. 可以通过继承或组合的方式来扩展策略模式，实现更加复杂的算法组合方式。
4. 可以在运行时动态地切换算法策略，使得系统更加灵活和可配置。

### 策略模式的缺点

1. 需要定义许多具体的策略类，增加了代码的复杂度和维护成本。
2. 客户端需要了解所有的具体策略类，增加了代码的复杂度和耦合度。

### 举个例子

如果想实现一个多方式登陆的方法，我们可以这样做：
```typescript
function login(mode) {
  if (mode === 'account') {
    loginWithAccount()
  } else if (mode === 'phone') {
    loginWithPhone()
  } else if (mode === 'email') {
    loginWithEmail()
  } 
}
```
但是如果后期还需要添加weixin登陆，git登陆呢，我们只能不断的修改 login 方法，这样login内的代码就会变得越来越臃肿。而且，也不利于维护。

这种情况，就特别适用于策略模式了。

#### 1. 策略三角色 --- 策略接口()：

```typescript
interface IStrategy {
  login(args: any[]): boolean
}
```

#### 2. 策略三角色 --- 实现策略类(ConcreteStategy)：

```typescript
class AccountStategy implements IStrategy{
  login(args: any[]) {
    console.log(args)
    const [userName, password] = args
    if (userName != 'xwk' || password != '000') {
      console.log('no permission to login')
      return false
    }
    console.log('AccountStategy successful')
    return true
  }
}
class PhoneStategy implements IStrategy {
  login(args: any[]) {
    console.log(args)
    const [phone, smsCode] = args
    if (phone != '12345' || smsCode != '000') {
      console.log('no permission to login')
      return false
    }
    console.log('PhoneStategy successful')
    return true
  }
}
class EmailStategy implements IStrategy {
  login(args: any[]) {
    const [email, code] = args
    if (email != '123.qq.com' || code != '000') {
      console.log('no permission to login')
      return false
    }
    console.log('EmailStategy successful')
    return true
  }
}
```

#### 3. 策略三角色 --- 引用策略类(Context)：

```typescript
class Authenticator {
  strategies: Record<string, IStrategy> = {}
  use(name: string, strategy: IStrategy) {
    this.strategies[name] = strategy
  }
  authenticate(name: string, ...args: any) {
    console.log(args)
    return this.strategies[name].login.apply(null, args)
  }
}
```

最后客户端调用代码如下，

```typescript
const auth = new Authenticator()
auth.use('account', new AccountStategy())
auth.use('phone', new PhoneStategy())
auth.use('email', new EmailStategy())

auth.authenticate('account', ['xwk','000'])
auth.authenticate('phone', ['12345','000'])
```