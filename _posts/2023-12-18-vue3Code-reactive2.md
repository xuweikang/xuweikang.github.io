---
layout: post
title: vue3源码 -- 2.  响应式【中】
tags: vue3 源码
categories: SourceCodeAnalysis
---

* TOC 
{:toc}

## 回顾Vue2 的响应式

![image.png](/static/img/vue3/reactivity/2.png)

上图来自官方文档，**初始化时对状态数据做了劫持，在执行组件的render函数时，会访问一些状态数据，就会触发这些状态数据的getter，然后render函数对应的render watcher就会被这个状态收集为依赖，当状态变更触发setter，setter中通知render watcher执行 update，然后render函数重新执行以更新组件。** 就这样完成了响应式的过程。

## Vue3 的响应式从 Reactive 开始

上篇文章，**从源码的最入口处开始，一步步寻找响应式的起源。**

最终，在下面代码里， `instance.data = reactive(data)` 找到对 data 数据的响应式化处理。

```ts
export function applyOptions(instance: ComponentInternalInstance) {
    // code...
    const {
      // state
      data: dataOptions
    } = options
     // code...
  if (dataOptions) {
    const data = dataOptions.call(publicThis, publicThis)
  
    if (!isObject(data)) {
      __DEV__ && warn(`data() should return an object.`)
    } else {
      instance.data = reactive(data) // 这里！！！
    }
  }
}
```

Vue3使用`Proxy`来实现数据的劫持，接下来我们进入源码（packages/reactivity/src/reactive.ts）

![image.png](/static/img/vue3/reactivity/3.png)

如果是只读，就这是返回 `target`，不做任何处理。

### createReactiveObject：

![image.png](/static/img/vue3/reactivity/4.png)

`createReactiveObject` 代码主要做了一些校验，然后最终 `new Proxy(...)` 并返回。

1. 如果 target 不是 Object 就直接返回不做响应式处理；
2. 如果目标对象已经是个 proxy 了就直接返回 target ；

主要逻辑在 new Proxy 的第二个参数里，也就是拦截方法里。如果tagetType 是集合类型（Map、Set、WeakMap、WeakSet这些），就用 collectionHandlers，否则常规的就是 baseHandlers。

**接下来按照常规 baseHandlers 继续阅读下去。**

### baseHandlers:

`baseHandlers` 是作为`createReactiveObject` 的第三个参数传入的，找到 `mutableHandlers` ，
mutableHandlers（packages/reactivity/src/baseHandlers.ts）：

![image.png](/static/img/vue3/reactivity/5.png)

继续找到 MutableReactiveHandler

![image.png](/static/img/vue3/reactivity/6.png)

这里面已经拦截了 ` set、deleteProperty、has、ownKeys`。

基类 `BaseReactiveHandler里拦截了 get`：

![image.png](/static/img/vue3/reactivity/7.png)

下面按照顺序分别细看下具体的代理拦截逻辑。

### Proxy.get

```ts
get(target: Target, key: string | symbol, receiver: object) {
    const isReadonly = this._isReadonly,
      shallow = this._shallow
    if (key === ReactiveFlags.IS_REACTIVE) {
      return !isReadonly
    } else if (key === ReactiveFlags.IS_READONLY) {
      return isReadonly
    } else if (key === ReactiveFlags.IS_SHALLOW) {
      return shallow
    } else if (
      key === ReactiveFlags.RAW &&
      receiver ===
        (isReadonly
          ? shallow
            ? shallowReadonlyMap
            : readonlyMap
          : shallow
          ? shallowReactiveMap
          : reactiveMap
        ).get(target)
    ) {
      return target
    }

    const targetIsArray = isArray(target)

    if (!isReadonly) {
      if (targetIsArray && hasOwn(arrayInstrumentations, key)) { // 是否是特定数组方法
        debugger
        return Reflect.get(arrayInstrumentations, key, receiver)
      }
      if (key === 'hasOwnProperty') {
        return hasOwnProperty
      }
    }

    const res = Reflect.get(target, key, receiver)

    if (isSymbol(key) ? builtInSymbols.has(key) : isNonTrackableKeys(key)) {
      return res
    }

    if (!isReadonly) {
      track(target, TrackOpTypes.GET, key) // 收集依赖
    }

    if (shallow) {
      // 如果是浅响应式，只收集第一层依赖，后面的直接返回了
      return res
    }

    if (isRef(res)) {
      // ref unwrapping - skip unwrap for Array + integer key.
      // 如果是数组并且key 是整数就直接返回 res，否则返回 res.value
      return targetIsArray && isIntegerKey(key) ? res : res.value
    }

    if (isObject(res)) {
      // Convert returned value into a proxy as well. we do the isObject check
      // here to avoid invalid value warning. Also need to lazy access readonly
      // and reactive here to avoid circular dependency.
      // 如果res是对象，那么就接着进行响应式处理，并返回代理对象递归处理，
      return isReadonly ? readonly(res) : reactive(res)
    }

    return res
  }
```
target 是需要代理的数据源，key 是访问target 的 key。 

1. 首先处理了下特定的几个 key，ReactiveFlags.xxx
2. if (targetIsArray && hasOwn(arrayInstrumentations, key))  。如果是数组，并且 key是 arrayInstrumentations 的方法，就 return Reflect.get(arrayInstrumentations, key, receiver)。**arrayInstrumentations的方法下面会列出**。
3. 继续往下执行，`const res = Reflect.get(target, key, receiver)`，拿到 res 的值，如果 key 是一个 Symbol类型的值并且 在内置 `builtInSymbols` 里，或者 `isNonTrackableKeys(key) 为 true`，就直接返回 res，不需要进行接下来的依赖收集；
4. 如果不是只读， ` track(target, TrackOpTypes.GET, key)`，进行依赖收集。**这里的 track 方法是进行依赖收集的，后面会列出。**
5.  如果是浅响应式，只收集第一层依赖，后面的直接返回了；
6.  如果是数组并且key 是整数就直接返回 res，否则返回 res.value；
7. 如果res是对象，那么就接着进行响应式处理，并返回代理对象递归处理；
8. 最后返回 res。

至此，proxy.get 方法拦截结束，主要是根据 key 的类型进行特定返回，然后进行依赖收集，判断是否是shallow，是否是 ref ，是否是对象，最终返回 res 结果。

get 方法遗留有两个问题，**一个是 track 方法的定义，这个后面一起讲，这里做个标记**。

另一个就是 arrayInstrumentations。
```ts
const arrayInstrumentations = /*#__PURE__*/ createArrayInstrumentations()
function createArrayInstrumentations() {
  const instrumentations: Record<string, Function> = {}
  // instrument identity-sensitive Array methods to account for possible reactive
  // values
  ;(['includes', 'indexOf', 'lastIndexOf'] as const).forEach(key => {
    instrumentations[key] = function (this: unknown[], ...args: unknown[]) {
      debugger
      const arr = toRaw(this) as any

      // 为什么这里需要遍历下再 track？
      // 因为这3 个数组方法，第二个参数 key是可以指定开始遍历的位置从 key 开始。
      for (let i = 0, l = this.length; i < l; i++) {
        track(arr, TrackOpTypes.GET, i + '')
      }
      // we run the method using the original args first (which may be reactive)
      const res = arr[key](...args)

      // 排除参数也是个响应式变量，toRaw 后继续执行一遍
      if (res === -1 || res === false) {
        // if that didn't work, run it again using raw values.
        return arr[key](...args.map(toRaw))
      } else {
        return res
      }
    }
  })
  // instrument length-altering mutation methods to avoid length being tracked
  // which leads to infinite loops in some cases (#2137)
  ;(['push', 'pop', 'shift', 'unshift', 'splice'] as const).forEach(key => {
    instrumentations[key] = function (this: unknown[], ...args: unknown[]) {
      // 为什么需要先暂停依赖收集，等执行完后再恢复收集？
      // 解释例子：vue/examples/reactive/arrayGet.html
      pauseTracking()
      const res = (toRaw(this) as any)[key].apply(this, args)
      resetTracking()
      return res
    }
  })
  return instrumentations
}
```
createArrayInstrumentations方法，对 数组的一些方法进行了重写

1. 对'includes', 'indexOf', 'lastIndexOf' 这3 个方法的重写，首先对数组的所有项进行依赖收集，type 是 get，key 为 index；然后执行方法，返回 index，如果 找到，再将参数toRaw 后继续执行一遍，排除参数也是个响应式变量；
2. 对'push', 'pop', 'shift', 'unshift', 'splice'这 5 个方法的重写，主要就是先暂停依赖收集，等结果执行完毕后再恢复依赖收集，为什么如此操作？可以看下面的这个例子：

```ts
//  看一个例子
const arr = ['h','e','l','l']
const proxyArr = new Proxy(arr, {
  get(target,key) {
    console.log('get', key)
    return Reflect.get(arr, key)
  },
  set(target,key, value) {
    console.log('set', key)
    return Reflect.set(target, key, value)
  }
})
proxyArr.push('o')
// 结果如下：
// get push
// get length
// set 4
// set length

// 我们能够看到，调用了一次push，会触发一次length的get，一次length的set，而且刚好被重写的这些API都是会改变数组length的API。
// 这样的话，假如我们没有对这些API重写，在effect中使用这些API会怎么样：
// const obj2 = reactive(arr)
// effect(() => {
//   obj2.push('o')
// })
// effect(() => {
//   obj2.push('!')
// })

/**
  第1个effect：
  收集key='length'，触发track(target, ..., 'length')操作
  相当于proxy[5]=o，触发key='5' 以及 key='length' 的 trigger 操作

  第2个effect：
  收集key='length'，触发track(target, 'length')操作
  相当于proxy[6]=!，触发key='6'以及key='length'的trigger操作

  由于第1个effect收集了key='length'，因此会触发第1个effect重新执行，再次收集key='length'和触发key='7'以及key='length'的trigger操作；
  由于第2个effect收集了key='length'，因此会触发第2个effect重新执行，再次收集key='length'和触发key='7=8'以及key='length'的trigger操作；
  由于第1个effect收集了key='length'，因此会触发第1个effect重新执行，再次收集key='length'和触发key='9'以及key='length'的trigger操作；
  引起了死循环......
*/
```

### Proxy.set

```ts
set(
    target: object,
    key: string | symbol,
    value: unknown,
    receiver: object
  ): boolean {
    let oldValue = (target as any)[key]
    if (isReadonly(oldValue) && isRef(oldValue) && !isRef(value)) {
      return false
    }
    if (!this._shallow) {
      if (!isShallow(value) && !isReadonly(value)) {
        oldValue = toRaw(oldValue)
        value = toRaw(value)
      }
      if (!isArray(target) && isRef(oldValue) && !isRef(value)) {
        // 静态数据，直接更新
        oldValue.value = value
        return true
      }
    } else {
      // in shallow mode, objects are set as-is regardless of reactive or not
    }

    const hadKey =
      isArray(target) && isIntegerKey(key)
        ? Number(key) < target.length
        : hasOwn(target, key)
    const result = Reflect.set(target, key, value, receiver)
    // don't trigger if target is something up in the prototype chain of original
    // 如果target是原型链上的东西，不触发更新
    if (target === toRaw(receiver)) {
      if (!hadKey) {
        trigger(target, TriggerOpTypes.ADD, key, value) // 触发新增 更新
      } else if (hasChanged(value, oldValue)) {
        trigger(target, TriggerOpTypes.SET, key, value, oldValue)  // 触发直接设置 更新 
      }
    }
    return result
  }
```

1. 如果oldValue是只读的Ref，但是value（即将设置的值）不是Ref，直接return false。
2. `target === toRaw(receiver)` **表示target不是原型链上的东西，就是其本身。**
为什么会判断下上面这个？举个例子：
```ts
const obj = {a:1}
proxy = new Proxy(obj, {
    get(target,key, receiver){
        console.log(key, receiver === proxy)
    }
})
obj.a // 这时候的打印结果为：a true 因为第 3 个参数 receiver 是Proxy 或者继承 Proxy 的对象
const obj2 = Object.create(proxy)
obj2.a // 打印结果：a false。虽然通过原型链也能代理，但是这时候 receiver 已经指向obj2 了。
```
3. 如果没有 hadKey 是 false，说明是新增操作，trigger 触发更新，type 是 ADD
4. 如果 hadKey是 true，并且 oldValue 和 value 不相等，说明有修改，trigger 触发更新，type 是 SET

trigger 方法的定义，这个后面一起讲，**这里做个标记。是触发页面更新。**

### Proxy.deleteProperty
```ts
// 用于拦截从target删除某个属性
  deleteProperty(target: object, key: string | symbol): boolean {
    const hadKey = hasOwn(target, key)
    const oldValue = (target as any)[key]
    const result = Reflect.deleteProperty(target, key)
    if (result && hadKey) {
      // 触发更新
      trigger(target, TriggerOpTypes.DELETE, key, undefined, oldValue)
    }
    return result
  }
```
剩下的几个就比较简单了，上述是拦截从 target 上删除某个属性，并触发更新 trigger，type 为 DELETE

### Proxy.has

```ts

  has(target: object, key: string | symbol): boolean {
    const result = Reflect.has(target, key)
    if (!isSymbol(key) || !builtInSymbols.has(key)) {
      // 不能是Symbol类型 或者 不能是特定的几个 symbols 类型 ，如果操作 has 就触发依赖收集
      track(target, TrackOpTypes.HAS, key)
    }
    return result
  }
```
用于拦截判断某个key是否存在于target的操作（xxx in target），先调用Reflect.has拿到结果，然后判断如果这个key不是Symbol类型，则调用track收集依赖；如果是Symbol类型，那就判断这个key在不在builtInSymbols中，不在也调用track收集依赖，type 为 HAS。

### Proxy.ownKeys
```ts
  ownKeys(target: object): (string | symbol)[] {
    track(
      target,
      TrackOpTypes.ITERATE,
      isArray(target) ? 'length' : ITERATE_KEY
    )
    return Reflect.ownKeys(target)
  }
```
for in... 或者 for of... 拦截，track收集依赖，type 为 ITERATE or ITERATE_KEY

总结下，**到目前为止，Reactive 的模块结束了。 主要就是 new Proxy ，并做了一些拦截操作，在拦截里收集和触发依赖，收集依赖用 track 方法，依赖变化触发更新用 trigger 方法。**


## 再看看 ref 的实现

目前为止，我们还不知道依赖到底是什么。

带着这个问题，看下 ref 的源码实现（packages/reactivity/src/ref.ts）

![image.png](/static/img/vue3/reactivity/8.png)

![image.png](/static/img/vue3/reactivity/9.png)

Ref 真正的实现，在 RefImpl 这个类里面。从源码里也可以看出，ref 的实现并没有用 proxy，**而是采用 ES2015 类的方法进行实现，所以打包降级后还是通过 Object.defineProperty对value的访问进行拦截**

### RefImpl

```ts
class RefImpl<T> {
  private _value: T // 存 value
  private _rawValue: T // 存 原始 value

  public dep?: Dep = undefined // 存依赖
  public readonly __v_isRef = true // 是否是 Ref 对象

  constructor(
    value: T,
    public readonly __v_isShallow: boolean
  ) {
    this._rawValue = __v_isShallow ? value : toRaw(value)
    this._value = __v_isShallow ? value : toReactive(value)
  }

  get value() {
    trackRefValue(this) // 收集依赖
    return this._value
  }

  set value(newVal) {
    const useDirectValue =
      this.__v_isShallow || isShallow(newVal) || isReadonly(newVal)
    newVal = useDirectValue ? newVal : toRaw(newVal)
    if (hasChanged(newVal, this._rawValue)) {
      this._rawValue = newVal
      this._value = useDirectValue ? newVal : toReactive(newVal)
      triggerRefValue(this, newVal)
    }
  }
}
```

RefImpl 类有几个构造属性，_value 和 _rawValue 是私有属性，分别存放 新的 value 值和之前的 value 值。

dep 存放依赖。劫持了 value 属性的获得和设置，并且同样也做了 get 依赖收集，set 触发更新。但是和 reactive 不同，**不是用的 track/trigger 方法，而是 trackRefValue/triggerRefValue。**

### trackRefValue

```ts
export function trackRefValue(ref: RefBase<any>) {
  if (shouldTrack && activeEffect) { // activeEffect？？
    ref = toRaw(ref)
    if (__DEV__) {
      trackEffects(ref.dep || (ref.dep = createDep()), {
        target: ref,
        type: TrackOpTypes.GET,
        key: 'value'
      })
    } else {
      trackEffects(ref.dep || (ref.dep = createDep())) // 创建一个空的依赖存放地，并触发收集
    }
  }
}
```

**这里面有两个变量shouldTrack和activeEffect，记录下，后面应该会讲到。**

`trackEffects`方法，应该也是收集依赖的一个方法。可以也记录下。

`ref.dep = createDep()` 创建一个空的依赖存放地，并保存到 ref.dep 里。

### triggerRefValue
```ts
export function triggerRefValue(ref: RefBase<any>, newVal?: any) {
  ref = toRaw(ref)
  const dep = ref.dep
  if (dep) {
    if (__DEV__) {
      triggerEffects(dep, {
        target: ref,
        type: TriggerOpTypes.SET,
        key: 'value',
        newValue: newVal
      })
    } else {
      triggerEffects(dep)
    }
  }
}
```

拿到ref.dep，并且 触发更新，triggerEffects方法，参数是 dep 依赖。

上面两个方法，都用到了 dep。下面看下 dep 究竟是什么，从 `createDep` 方法开始看起。

### createDep
```ts
export const createDep = (effects?: ReactiveEffect[]): Dep => {
  const dep = new Set<ReactiveEffect>(effects) as Dep
  dep.w = 0 // wasTracked
  dep.n = 0 // newTracked
  return dep
}
```

这方法很简单，如果不传参数，就是创建一个空的 Set 作为 dep，并且设置 dep.w和 dep.n，这两个属性，是为了做性能优化用的。**做个标记，现在只需要知道dep 有这两个属性就行。**

到目前为止，ref 的实现也没有了，遗留几个问题：

1. shouldTrack和activeEffect是什么
2. trackEffects和triggerEffects方法的定义
3. dep.w 和 dep.n 是干什么用的

## Effect.ts

下面从哪条分支开始看呢？还是继续createDep方法看能不能看出点啥。

目前为止，虽然我们知道 dep 是个 Set 集合，但也仅仅而已，还是不清楚究竟 dep 的类型长啥样。
ts 的优势体现出来了。

```ts
  const dep = new Set<ReactiveEffect>(effects) as Dep
```

可以看到，这个 Set 类型是 `ReactiveEffect` 的集合。

从这个类型入手，可以认为`ReactiveEffect`就是封装依赖的对象。

### ReactiveEffect

![image.png](/static/img/vue3/reactivity/10.png)

`ReactiveEffect` 是个类，这个类有个 run 实例方法。联想到 vue2 的 watch 类，同样也有个 run 方法。

在 `Vue2` 中，有一个 `Watcher` 类，其种类包括 `RenderWatcher`(渲染Watcer)`、UserWatcher`(用户定义的watch选项)`、ComputedWatcher`(用户定义的computed选项)。在对数据进行响应式处理时，数据会持有 `Dep` 实例，并且将这些 `Watcher` 实例添加到 `subs` 中；当依赖的数据变化，通过 `dep.notify()` 通知 `subs` 中的每个 `Watcher` `实例，Watcher` 实例收到通知后再去做相应的处理。由此可见 `Watcher` 类的作用就是封装各种 Watcher 的处理逻辑。可以说在 Vue2 中 `Watcher实例` 就是依赖。

同样的 `ReactiveEffect` 的作用与 `Watcher` 差不多。

全局搜下这个，有哪些地方 `new` 了这个类。

![image.png](/static/img/vue3/reactivity/11.png)

感觉和 vue2 的 `new Watcher`的使用地方也一致。

具体看下实现：

`ReactiveEffect` 类的构造函数有几个比较重要的属性，

1. fn，这是第一个参数，后面执行 run 方法的时候会直接调用它
2. scheduler，这是一个调度器，vue3 源码里面有大量的这种调度器，作为第二个构造参数。**这里做个标记，后面会补充到。**

run方法：

```ts
run() {
    if (!this.active) {
      // 如果是暂停状态，就只执行 fn 后直接返回，不收集依赖
      return this.fn()
    }
    let parent: ReactiveEffect | undefined = activeEffect
    let lastShouldTrack = shouldTrack
    while (parent) {
      if (parent === this) {
        return
      }
      parent = parent.parent
    }
    try {
      this.parent = activeEffect
      activeEffect = this
      shouldTrack = true

      trackOpBit = 1 << ++effectTrackDepth // effectTrackDepth表示当前 run 执行深度

      // 至于maxMarkerBits为什么是 30？
      // 是因为trackOpBit的取值是 1 左移 effectTrackDepth位，js 采用的计数是 32 位的有符号二进制表示，第 1 位是符号位，
      // 所以最多只能左移30 位。

      // 该 优化方案是 3.2 以后的优化方案，3.2 版本之前的，统一走cleanupEffect方法
      if (effectTrackDepth <= maxMarkerBits) {
        initDepMarkers(this) // deps[i].w |= trackOpBit 给dep.w 做标记。 标记优化
      } else {
        cleanupEffect(this) // 删除所有依赖，清空deps。然后执行 fn 重新开始收集。这样重新收集所有依赖对性能要求较高
      }
      return this.fn() // 执行 fn，如果 fn 中有使用到响应式变量的 get，就会触发收集函数 track
    } finally {
      if (effectTrackDepth <= maxMarkerBits) {
        finalizeDepMarkers(this) // 进一步处理和优化 debs 数组
      }

      // 还原操作
      trackOpBit = 1 << --effectTrackDepth

      activeEffect = this.parent
      shouldTrack = lastShouldTrack
      this.parent = undefined

      if (this.deferStop) {
        this.stop()
      }
    }
  }
```

这个`run`的作用就是用来执行`副作用函数fn`，并且在执行过程中进行依赖收集：

**在 renderder.ts 中，有这样一段代码（组件初始化的时候会执行到此）：**

![image.png](/static/img/vue3/reactivity/12.png)

组件初始化的时候，会手动执行 `update` 方法一次，目的是为了收集依赖。

而这个 `update` 方法执行的是 `effect.run`方法，而 `effect` 又是 `new ReactiveEffect` 的。所以，再来看
`ReactiveEffect`代码。

假设我们的组件结构是这样的。

```html
<parent>
    <child>
    </child>
</parent>
```

1. 如果当前effect实例是不工作状态，就仅仅执行一下fn，不需要收集依赖。

2. 由于在一个effect.run的过程中可能会触发另外的effect.run, 暂存上一次的activeEffect、shouldTrack，目的是为了本次执行完以后把activeEffect、shouldTrack恢复回去。

3. 找到最顶层 parent。

4. 设置activeEffect shouldTrack，回答上面了一个标记（activeEffect是什么），activeEffect其实就是ReactiveEffect中的 this，也就是依赖本身。shouldTrack是一个全局标记，表示是否需要开始进行依赖收集。这里表示需要进行依赖收集。

5. trackOpBit = 1 << ++effectTrackDepth，effectTrackDepth是个全局变量，表示当前 调用 effect.run 的深度，trackOpBit 更新为 1 << effectTrackDepth。effectTrackDepth和trackOpBit的关系如下：

    | effectTrackDepth（effect.run 嵌套调用深度）  | trackOpBit |
    | 1 | 0001<<1=0010=2 |
    | 2 | 0001<<2=0100=4 |
    | 3 | 0001<<3=1000=8 |


6.  effectTrackDepth 是否小于全局的 maxMarkerBits = 30，如果是就调用 initDepMarkers 给 dep.w 做标记（优化方案），否则就把依赖都删除（简单方案）。

7. 执行 fn()

8. finally中主要是做一些善后的工作了：移除多余依赖、恢复activeEffect、shouldTrack、调用--effectTrackDepth & trackOpBit更新。其中 finalizeDepMarkers方法也是优化方案的一部分，后面讲优化会讲到。


### 非优化方案cleanupEffect

```ts
function cleanupEffect(effect: ReactiveEffect) {
  const { deps } = effect
  if (deps.length) {
    for (let i = 0; i < deps.length; i++) {
      deps[i].delete(effect)
    }
    deps.length = 0
  }
}
```

如果 effectTrackDepth > maxMarkerBits，采用上述方案，每次采集依赖的时候，都会删除所有依赖，清空deps。然后执行 fn 重新开始收集。这样重新收集所有依赖很有可能有重复采集依赖，即之前采集过。**3.2 版本之前的，统一走cleanupEffect方法。**

**为什么maxMarkerBits 是 30？**

这是因为 js 采用的计数是 32 位的有符号二进制表示，第 1 位永远是符号位，层级加 1 左移1 位，最后一位不算。如果用二进制表示层级数的话，最多只能表示 30 位。

### 优化方案

#### initDepMarkers

```ts
export const initDepMarkers = ({ deps }: ReactiveEffect) => {
  if (deps.length) {
    for (let i = 0; i < deps.length; i++) {
      deps[i].w |= trackOpBit // set was tracked 按位或运算 dep.w |= trackOpBit
    }
  }
}
```

遍历 deps，更新 deps[i].w ，与 trackOpBit 做 或 运算，其实相当于给每一层标记下当前 dep 有没有被收集过。

比如 effect.run 到第3层，trackOpBit = 4 ，用二进制表示就是 0100，首次初始化的时候第3层deps[i].w = 0000，执行 完fn 并且收集了依赖，此时 deps[i].n = 0100， deps[i].w = 0000，

#### finalizeDepMarkers

然后执行到 finally，执行finalizeDepMarkers 方法

![image.png](/static/img/vue3/reactivity/14.png)

```ts
export const finalizeDepMarkers = (effect: ReactiveEffect) => {
  const { deps } = effect
  if (deps.length) {
    let ptr = 0
    for (let i = 0; i < deps.length; i++) {
      const dep = deps[i]
      if (wasTracked(dep) && !newTracked(dep)) {
        // 第二次添加依赖的时候，并没有被标记未新依赖，所以需要删除他
        dep.delete(effect)
      } else {
        deps[ptr++] = dep // 这里有个技巧。一次循环，做到删除数组和重新赋值操作。比如原数组是 ['new1', 'discard1', 'discard2','new2']，执行一次循环后， 最后执行deps.length = ptr，得到['new1','new2']
      }
      // clear bits
      // 还原标记
      dep.w &= ~trackOpBit
      dep.n &= ~trackOpBit
    }
    deps.length = ptr
  }
}
```

finalizeDepMarkers 方法其实就是做优化了，根据 dep 的 n 和 w 属性，来删除前一次收集到的冗余依赖。

`wasTracked(dep) && !newTracked(dep)`

```ts
export const wasTracked = (dep: Dep): boolean => (dep.w & trackOpBit) > 0 // dep.w >= trackOpBit

export const newTracked = (dep: Dep): boolean => (dep.n & trackOpBit) > 0 // dep.n >= trackOpBit
```

这两个方法的定义，比较简单，就是拿 w 和 n 和trackOpBit做与运算，得到的结果如果 > 0，就表示在当前深度是有被标记位 1 的。

比如：

```ts
const switch = ref(true)
const foo = ref('foo')
effect( () = {
  if(switch.value){
    console.log(foo.value)
  }else{
    console.log('else condition')
  }
})
switch.value = false
```

当 switch 为 true 时，triggerEffect，双向删除后，执行副作用函数，switch、foo 会重新收集到依赖 effect

当 switch 变成 false 后，triggerEffect，双向删除后，执行副作用函数，仅有 switch 能重新收集到依赖 effect

![image.png](/static/img/vue3/reactivity/15.png)

#### 优化方案总结

1. 组件首次初始化的时候会执行effect.run方法，如果有子组件的话，同样也会执行阶段执行子组件的 effect.run 方法，所以就会形成嵌套，用 effectTrackDepth表示当前 run 方法执行的深度，trackOpBit 表示用二进制来存储当前的深度，比如 effectTrackDepth为 2的时候，trackOpBit二进制表示 0100；

2. 如果执行深度 <= 30 的话，就采用优化方案，否则采用暴力方案（3.2 版本之前的 vue 统一采用的后者方案）；

3. 暴力方法，就是每次 run 的时候都把之前的依赖全部清除掉，然后重新执行副作用函数 fn 进行依赖收集，缺点就是会产生很多无谓运算开销；

4. 优化方案，每次初始化 dep 依赖的时候，都会有一个 w 和 n 属性，初始化都是 0。每次当前依赖收集进来的时候，就在当前深度上标识 1，同样也用二进制表示，比如当前trackOpBit为 0100，那么标记后的n 就是 0100，w 目前还是 0000。等到下次依赖发生更新需要重新采集的时候（发生在同一个 activeEffec里），就会根据 w 和 n 进行依赖的筛选和删除，然后重新赋值 w 属性，n 清空。

**effect.ts 里面还有几个特别重要的方法定义，也是上面做了标记但是还没讲到的 track、trigger 方法**

#### track

回顾下，前面在讲 Reactive的定义里，new Proxy()的 get 拦截就使用到这个方法进行依赖收集。

```ts
export function track(target: object, type: TrackOpTypes, key: unknown) {
  if (shouldTrack && activeEffect) {
    // targetMap 用于保存和当前响应式对象相关的依赖内容，本身是一个 WeakMap类型
    // key是响应式对象 value是
    let depsMap = targetMap.get(target)
    if (!depsMap) {
      // value是 depsMap
      targetMap.set(target, (depsMap = new Map()))
    }
    let dep = depsMap.get(key)
    if (!dep) {
      depsMap.set(key, (dep = createDep())) // dep是个 set 集合
    }

    const eventInfo = __DEV__
      ? { effect: activeEffect, target, type, key }
      : undefined

    trackEffects(dep, eventInfo)
  }
}
```

这个函数，有3 个参数：

1. target 就是数据对象。
2. TrackOpTypes track操作类型，有以下三种。
    ```ts
      export const enum TrackOpTypes {
        GET = 'get', // target.key
        HAS = 'has', // key in target
        ITERATE = 'iterate' // 遍历
      }
      ```
3. key 对应数据对象的属性名，如果是数组，那就是索引。

track方法里面定义的变量有点多，主要就是确定 dep，然后执行 trackEffects。而 ref 函数的定义，也是最终执行的是trackEffects 方法。

targetMap 是个全局的 WeakMap，key 是 target，value 是 depsMap，depsMap的 key 是 get 的 key，value 是 dep，如果dep 是空就创建一个 dep，dep 是一个 Set 集合，dep.w = 0; dep.n = 0。

![image.png](/static/img/vue3/reactivity/16.png)


#### trackEffects

```ts
export function trackEffects(
  dep: Dep,
  debuggerEventExtraInfo?: DebuggerEventExtraInfo
) {
  debugger
  let shouldTrack = false
  if (effectTrackDepth <= maxMarkerBits) {
    if (!newTracked(dep)) { // newTracked = (dep: Dep): boolean => (dep.n & trackOpBit) > 0
      dep.n |= trackOpBit // set newly tracked dep.n = dep.n | trackOpBit
      shouldTrack = !wasTracked(dep) // / 如果前一次收集的 dep 已经被收集过了，就可以跳过本次收集
    }
  } else {
    // Full cleanup mode.
    shouldTrack = !dep.has(activeEffect!)
  }

  if (shouldTrack) {
    dep.add(activeEffect!)
    activeEffect!.deps.push(dep)
    if (__DEV__ && activeEffect!.onTrack) {
      activeEffect!.onTrack(
        extend(
          {
            effect: activeEffect!
          },
          debuggerEventExtraInfo!
        )
      )
    }
  }
}
```

1. 优化方案给 dep 加 n属性，；
2. 把当前的 activeEffect 加入到 dep 集合中；
3. 然后 将 dep 加入到 activeEffect 的 deps中；

#### trigger

```ts
export function trigger(
  target: object,
  type: TriggerOpTypes,
  key?: unknown,
  newValue?: unknown,
  oldValue?: unknown,
  oldTarget?: Map<unknown, unknown> | Set<unknown>
) {
  const depsMap = targetMap.get(target)
  if (!depsMap) {
    // never been tracked
    return
  }

  let deps: (Dep | undefined)[] = []
  if (type === TriggerOpTypes.CLEAR) {
    // collection being cleared
    // trigger all effects for target
    deps = [...depsMap.values()]
  } else if (key === 'length' && isArray(target)) {
    const newLength = Number(newValue)
    depsMap.forEach((dep, key) => {
      if (key === 'length' || key >= newLength) {
        deps.push(dep)
      }
    })
  } else {
    // schedule runs for SET | ADD | DELETE
    if (key !== void 0) {
      // void 0 === undefined
      deps.push(depsMap.get(key))
    }

    // also run for iteration key on ADD | DELETE | Map.SET
    switch (type) {
      case TriggerOpTypes.ADD:
        if (!isArray(target)) {
          deps.push(depsMap.get(ITERATE_KEY))
          if (isMap(target)) {
            deps.push(depsMap.get(MAP_KEY_ITERATE_KEY))
          }
        } else if (isIntegerKey(key)) {
          // new index added to array -> length changes
          deps.push(depsMap.get('length'))
        }
        break
      case TriggerOpTypes.DELETE:
        if (!isArray(target)) {
          deps.push(depsMap.get(ITERATE_KEY))
          if (isMap(target)) {
            deps.push(depsMap.get(MAP_KEY_ITERATE_KEY))
          }
        }
        break
      case TriggerOpTypes.SET:
        if (isMap(target)) {
          deps.push(depsMap.get(ITERATE_KEY))
        }
        break
    }
  }

  if (deps.length === 1) {
    if (deps[0]) {
     triggerEffects(deps[0])
    }
  } else {
    const effects: ReactiveEffect[] = []
    for (const dep of deps) {
      if (dep) {
        effects.push(...dep)
      }
    }
     triggerEffects(createDep(effects))
  }
}
```
上述方法，主要是根据不同的类型，去筛选中 dep 来更新。

1. 从targetMap获取depsMap，如果获取不到，代表从来没有track过，直接return。

2. 如果是Map|Set的clear操作，depsMap中的所有dep都需要去trigger，这个很好理解，对象被清空了，所以任何用到对象中任何一项的dep都需要被trigger。

3. 如果key是length，并且target是Array，只需要 trigger length对应的dep 和 索引大于等于 newlength对应的dep。这个也很好理解，索引大于等于newlength的被删了，length变了。

4. 最后一个分支是处理ADD|SET|DELETE。先 deps.push(depsMap.get(key))，对于ADD操作来说，这个肯定是undefined，不过SET|DELETE则可能有对应的dep，然后针对ADD|SET|DELETE分别处理。

5. 处理ADD操作时，如果target是数组并且key是正整数，直接trigger length 的 dep，如果是 非数组的情况，先将 key 为 ITERATE_KEY 的 dep 放进去（不管有没有），因为最后会把 undefined 的过滤掉；如果target 是 Map，还要将 key 为 MAP_KEY_ITERATE_KEY 的 dep 放进去。

6. 最后一段逻辑，区分了下 deps 数组长度是 1 还是非 1 的情况，我没弄明白是什么意思。本着 vue3 严谨、极致优化的代码风格。我能想到的唯一的作用就是，如果deps 数组只有一项，就直接去执行它进行重渲染。如果非 1，需要把 debs 的每一项合并到一起，目的是为了用 set 做个去重

#### triggerEffects

```ts
export function triggerEffects(
  dep: Dep | ReactiveEffect[],
  debuggerEventExtraInfo?: DebuggerEventExtraInfo
) {
  // spread into array for stabilization
  const effects = isArray(dep) ? dep : [...dep]

  // 先 trigger computed的effect，先执行computed的副作用函数
  // 因为其他的副作用函数可能依赖computed的value
  for (const effect of effects) {
    if (effect.computed) {
      triggerEffect(effect, debuggerEventExtraInfo)
    }
  }
  // 再trigger别的effect
  for (const effect of effects) {
    if (!effect.computed) {
      triggerEffect(effect, debuggerEventExtraInfo)
    }
  }
}
```

先trigger computed的effect，再trigger别的effect，为了防止死循环，因为别的副作用函数可能依赖 computed 的value。然后执行 triggerEffect

#### triggerEffect

```ts
function triggerEffect(
  effect: ReactiveEffect,
  debuggerEventExtraInfo?: DebuggerEventExtraInfo
) {
  if (effect !== activeEffect || effect.allowRecurse) {
    if (effect.scheduler) {
      effect.scheduler()
    } else {
      effect.run()
    }
  }
}
```
如果 effect.scheduler 存在，就执行它。如果没有，就执行 run方法。

这两个方法是啥呢？如果 vue2 的 watch 的 update，肯定就是更新逻辑了。

而 effect，就是 ReactiveEffect 的实例，scheduler是作为ReactiveEffect的第二个构造参数传进来的，run 方法内部执行的也是ReactiveEffect的第一个构造参数。

所以，这两个方法的执行逻辑，秘密就在哪里实例ReactiveEffect类。

#### renderer.ts

```ts
const setupRenderEffect: SetupRenderEffectFn = (
    instance,
    initialVNode,
    container,
    anchor,
    parentSuspense,
    isSVG,
    optimized
  ) => {
     const componentUpdateFn = () => {
         // ... 省略代码
     }

     const effect = (instance.effect = new ReactiveEffect(
      componentUpdateFn,
      () => queueJob(update),
      instance.scope // track it in component's effect scope
    ))
    const update: SchedulerJob = (instance.update = () => effect.run())
    update.id = instance.uid
    update()
  }
```

这个方法，在mountComponent方法里面会调用它。也就是组件初始化的时候。

按照前面 trigger 源码的解释，对应的 run 方法就是第一个参数 componentUpdateFn，这个方法是更新页面渲染逻辑，打个标记后面会重点讲到这个方法，

而scheduler是个调度方法，对应的是 `() => queueJob(update)`。update 方法的定义是 `(instance.update = () => effect.run())`，执行的是` effect.run` 方法。

#### queueJob

```ts
export function queueJob(job: SchedulerJob) {
  // the dedupe search uses the startIndex argument of Array.includes()
  // by default the search index includes the current job that is being run
  // so it cannot recursively trigger itself again.
  // if the job is a watch() callback, the search will start with a +1 index to
  // allow it recursively trigger itself - it is the user's responsibility to
  // ensure it doesn't end up in an infinite loop.
  if (
    !queue.length ||
    !queue.includes(
      job,
      isFlushing && job.allowRecurse ? flushIndex + 1 : flushIndex
    )
  ) {
    if (job.id == null) {
      queue.push(job)
    } else {
      // 插队
      queue.splice(findInsertionIndex(job.id), 0, job)
    }
    queueFlush()
  }
}

function queueFlush() {
  if (!isFlushing && !isFlushPending) {
    isFlushPending = true
    currentFlushPromise = resolvedPromise.then(flushJobs)
  }
}
```

这个方法，vue3 源码里很多地方都用到了。主要就是调度的作用，按照不同类型的队列，来决定怎么执行这个队列里面的方法。上面的例子就是 update 方法。还有个类型队列，可以插队。

## 总结

至此，响应式的整个流程差不多都走了一遍。

1. Vue 的 data 的响应式逻辑，是通过 reactive 实现的；

2. Reactive 通过 new Proxy 进行拦截的；

3. Ref 方法的实现是通过 Class 拦截 value 的 set 和 get 的；

4. Vue 的依赖收集和触发依赖更新的逻辑都在 effect.ts里，trackEffects 和 triggerEffect；

5. Vue 的依赖Dep 其实就是 ReactiveEffect 的实例，ReactiveEffect类有个 deps 属性，这里面放置的是 dep。组件会在初始化的时候，实例化一个ReactiveEffect实例，watch 和 computed 会分别 实例化一个新的ReactiveEffect。

6. 每个 dep 依赖，如果发生改变，会triggerEffect触发更新，执行 update 方法去重新渲染页面，剩下的更新逻辑交给 renderder。