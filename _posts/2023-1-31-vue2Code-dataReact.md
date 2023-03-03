---
layout: post
title: 1. Vue2框架源码分析-响应式原理
tags: Vue2 源码
categories: SourceCodeAnalysis
---

* TOC 
{:toc}

### 1. 使数据“可观测” （Observer）

```ts
class Observer {
  value: any
  constructor(value: any) {
    this.value = value
    if (Array.isArray(value)) {
      // 数组的响应式原理
    } else {
      this.walk(value)
    }
  }
  walk(data: any) {
    const keys = Object.keys(data)
    for (let i = 0; i < keys.length; i++) {
      this.defineReactive(data, keys[i]) // 给数据增加响应式绑定
    }
  }
  defineReactive(obj: any, key: string) {
    if(typeof key === 'object') {
      new Observer(key)
    }
    let val = obj[key]
    const dep = new Dep() // 实例化一个依赖管理器
    Object.defineProperty(obj, key, {
      enumerable: true,
      configurable: true,
      get() {
        // 触发依赖收集
        console.log('触发依赖收集')
        if (Dep.target) {
          dep.depend()
        }
        return val
      },
      set(newVal) {
        // 通知更新
        if (newVal === val || (newVal !== newVal && val !== val)) return
        console.log('数据更新了')
        dep.notify()
        val = newVal
      }
    })
  }
}
```

### 2. 依赖管理器 (Dep)

```ts
class Dep {
  subs: any[]
  static target: any
  constructor() {
    this.subs = []
  }
  addSub (sub: any) {
    this.subs.push(sub)
  }
  // 添加一个依赖
  depend() {
    if (Dep.target) {
      Dep.target.addDep(this)
    }
  }
  // 通知所有依赖进行更新
  notify() {
    const subs = this.subs.slice()
    for (let i = 0; i < subs.length; i ++) {
      subs[i].update()
    }
  }
}
Dep.target = null'
```

### 3. 依赖到底是谁？（Watcher）
```ts
class Watcher {
  value: any
  getter: Function
  constructor(exOrFn: Function) {
    this.getter = exOrFn
    this.value = this.get() // 手动触发get
  }
  get() {
    Dep.target = this // 设置 Dep.target，在触发依赖收集的时候能够分别
    let value = this.getter.call(this) // 执行exOrFn触发get（触发渲染函数）
    Dep.target = null // 重置 Dep.target，为下一个watcher实例做准备
    return value
  }
  addDep(dep: any) {
    dep.addSub(this)
  }
  update() {
    console.log('watcher update', )
  }
}
```


### 验证
```ts
const glob: any = global
// 模拟渲染render
function render() {
  console.log('开始render', glob.$data.name, glob.$data.age)
  setTimeout(() => {
    console.log('开始render2', glob.$data.name, glob.$data.age)
  },0)
}
const obj = new Observer({
  name: 'xwk',
  age: '30'
});
glob.$data = new Proxy(obj.value, {}) //代理到$data
new Watcher(render) // render里面进行了依赖收集

obj.value.name = 'xwk2' // 改变数据
obj.value.age = '31' // 改变数据
```
### 总结

1.  vue采用观察者模式实现的mvvm模式
2.  每个data数据都会实例化一个Dep依赖管理器
3.  每个vue模板文件在mounted生命周期的时候，会初始化一个渲染Watcher实例，并手动触发了一次本次渲染用到的所有data的get，进行依赖收集（dep.depend()，收集的其实就是watcher实例），如果数据改变触发了set，则会通知依赖管理器（dep.notify()），然后dep执行依赖们（watcher实例，可能不止一个）进行update更新。

![image.png](/static/img/vue2/demo1.png)

