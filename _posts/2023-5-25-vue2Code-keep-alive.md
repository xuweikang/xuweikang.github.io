---
layout: post
title: Vue2框架源码分析-keep-alive
tags: Vue2 源码
categories: SourceCodeAnalysis
---

* TOC 
{:toc}

## 介绍和应用

### keep-alive是什么
> keep-alive是一个抽象组件：它自身不会渲染一个 DOM 元素，也不会出现在父组件链中；使用keep-alive包裹动态组件时，会缓存不活动的组件实例，而不是销毁它们。

### 它的使用场景

1. 动态组件切换：当应用需要在不同的状态之间频繁切换时，Keep-Alive可以帮助我们缓存已经创建的组件实例，避免重复的创建和销毁过程，提高组件切换的性能。

```vue
<keep-alive :include="whiteList" :exclude="blackList" :max="max">
  <component :is="currentComponent"></component>
</keep-alive>
```

2. 路由导航：在使用Vue Router进行路由导航时，经常需要切换页面组件。如果每次路由切换都销毁之前的组件并重新创建新的组件，会导致性能下降。通过将需要缓存的页面组件包裹在Keep-Alive中，可以在路由切换时保留组件状态，提高导航性能。

```vue
<keep-alive :include="whiteList" :exclude="blackList" :max="max">
  <router-view></router-view>
</keep-alive>
```

3. 频繁加载的组件：对于某些需要频繁加载的组件，例如列表项或者弹窗组件，使用Keep-Alive可以避免每次重新渲染和重新挂载组件，从而提高性能。

4. 表单数据保持：在一些表单场景中，当用户切换表单页签或者步骤时，如果不对组件进行缓存，会导致用户已经输入的数据丢失。通过使用Keep-Alive包裹表单组件，可以保持表单数据的持久性，提供更好的用户体验。

`keep-alive` 组件可以接受3个props，分别是：`include`: 缓存白名单，keep-alive会缓存命中的组件 `exclude`:  缓存黑名单，这些组件将不会被缓存 `max`：定义缓存组件上限，超出上限将使用 **LRU** 的策略置换缓存数据。


## 源码解析

`keep-alive` 组件的定义位置在 `src/core/components/keep-alive.js`,核心代码如下：

```js
// src/core/components/keep-alive.js
export default {
  name: 'keep-alive',
  abstract: true,  // 判断当前组件虚拟dom是否渲染成真是dom的关键

  props: {
    include: patternTypes,
    exclude: patternTypes,
    max: [String, Number]
  },

  created () {
    this.cache = Object.create(null)
    this.keys = []
  },

  destroyed () {
    for (const key in this.cache) { // 删除所有的缓存
      pruneCacheEntry(this.cache, key, this.keys)
    }
  },

  mounted () {
    // 实时监听黑白名单的变动
    this.$watch('include', val => {
      pruneCache(this, name => matches(val, name))
    })
    this.$watch('exclude', val => {
      pruneCache(this, name => !matches(val, name))
    })
  },

  render () {
    ...
  }
}
```

keep-alive定义了三个钩子函数：

- **created:**
初始化两个对象分别缓存VNode（虚拟DOM）和VNode对应的键集合（为了实现LRU）

- **destroyed:**
删除this.cache中缓存的VNode实例。遍历调用pruneCacheEntry函数删除,目的是为了触发组件实例的destory钩子函数。

```js
function pruneCacheEntry (
  cache: VNodeCache,
  key: string,
  keys: Array<string>,
  current?: VNode
) {
  const cached = cache[key]
  if (cached && (!current || cached.tag !== current.tag)) {
    cached.componentInstance.$destroy()
  }
  cache[key] = null
  remove(keys, key)
}
```
- **mounted:**
在mounted这个钩子中对include和exclude参数进行监听，然后实时地更新（删除）this.cache对象数据。pruneCache函数的核心也是去调用pruneCacheEntry，只不过多了一层对includes和excludes的校验。

```js
function pruneCache (keepAliveInstance: any, filter: Function) {
  const { cache, keys, _vnode } = keepAliveInstance
  for (const key in cache) {
    const cachedNode: ?VNode = cache[key]
    if (cachedNode) {
      const name: ?string = getComponentName(cachedNode.componentOptions)
      if (name && !filter(name)) {
        pruneCacheEntry(cache, key, keys, _vnode)
      }
    }
  }
}
```

- **render:**

```js
render () {
    const slot = this.$slots.default
    const vnode: VNode = getFirstComponentChild(slot)
    const componentOptions: ?VNodeComponentOptions = vnode && vnode.componentOptions
    if (componentOptions) {
      // check pattern
      const name: ?string = getComponentName(componentOptions)
      const { include, exclude } = this
      if (
        // not included
        (include && (!name || !matches(include, name))) ||
        // excluded
        (exclude && name && matches(exclude, name))
      ) {
        return vnode
      }

      const { cache, keys } = this
      const key: ?string = vnode.key == null
        // same constructor may get registered as different local components
        // so cid alone is not enough (#3269)
        ? componentOptions.Ctor.cid + (componentOptions.tag ? `::${componentOptions.tag}` : '')
        : vnode.key
      if (cache[key]) {
        vnode.componentInstance = cache[key].componentInstance
        // make current key freshest
        remove(keys, key)
        keys.push(key)
      } else {
        cache[key] = vnode
        keys.push(key)
        // prune oldest entry
        if (this.max && keys.length > parseInt(this.max)) {
          pruneCacheEntry(cache, keys[0], keys, this._vnode)
        }
      }

      vnode.data.keepAlive = true
    }
    return vnode || (slot && slot[0])
  }
```

上述render方法的实现代码，会先取出第一个子组件对象及其组件名，判断是否缓存过，如果缓存过就取缓存的vnode，并且将该缓存对应的key放到keys数组的最后面（更新key的位置是实现LRU置换策略的关键），如果没有缓存过，就添加进缓存，并且push key到keys。判断是否超出最大max，如果超出就删除keys数组的第一项缓存，也就是进行LRU置换。


### keep-alive不会生成真正的DOM节点，这是怎么做到的？

```js
// src/core/instance/lifecycle.js
export function initLifecycle (vm: Component) {
  const options = vm.$options

  // locate first non-abstract parent
  // 找到第一个非abstract的父组件实例
  let parent = options.parent
  if (parent && !options.abstract) {
    while (parent.$options.abstract && parent.$parent) {
      parent = parent.$parent
    }
    parent.$children.push(vm)
  }
  ...
}
```

Vue在初始化生命周期的时候，为组件实例建立父子关系会根据abstract属性决定是否忽略某个组件。在keep-alive中，设置了abstract: true，那Vue就会跳过该组件实例。

### keep-alive包裹的组件是如何使用缓存的？

在patch阶段，会执行createComponent函数：

```js
// src/core/vdom/patch.js
function createComponent (vnode, insertedVnodeQueue, parentElm, refElm) {
    let i = vnode.data
    if (isDef(i)) {
      const isReactivated = isDef(vnode.componentInstance) && i.keepAlive // keepAlive是在keep-alive 组件的创建时候添加到VNOde的。
      if (isDef(i = i.hook) && isDef(i = i.init)) {
        i(vnode, false /* hydrating */)
      }
      if (isDef(vnode.componentInstance)) {
        // keep-alive 第一次执行的时候不会走到这。
        // 第二次的时候就会发现有缓存
        initComponent(vnode, insertedVnodeQueue)
        insert(parentElm, vnode.elm, refElm)
        if (isTrue(isReactivated)) {
          reactivateComponent(vnode, insertedVnodeQueue, parentElm, refElm)
        }
        return true
      }
    }
  }
```
`keep-alive` 页面渲染的过程，首先执行其提供的render函数拿到虚拟DOM，然后进行patch替换和生成真实DOM，因为是个组件类型的虚拟DOM，patch的时候执行`createComponent`方法，

这里第一次执行到`createComponen`方法时候，因为`vnode.componentInstance`还没有赋值，所以只能执行到`i(vnode, false /* hydrating */)` 这里，这里的`i`对应的是`vnode.data.hook.init`，执行该`init`方法。该方法的定义，在下面：

```js
// src/core/vdom/create-component.js
const componentVNodeHooks = {
  init (vnode: VNodeWithData, hydrating: boolean): ?boolean {
    if (
      vnode.componentInstance &&
      !vnode.componentInstance._isDestroyed &&
      vnode.data.keepAlive
    ) {
      // kept-alive components, treat as a patch
      // keep-alive 组件
      const mountedNode: any = vnode // work around flow
      componentVNodeHooks.prepatch(mountedNode, mountedNode)
    } else {
      // 正常component
      const child = vnode.componentInstance = createComponentInstanceForVnode(
        vnode,
        activeInstance
      )
      // 执行$mount，进行模版解析并且转化经过优化成虚拟dom，生成render函数，并且处罚created、mounted声明周期
      child.$mount(hydrating ? vnode.elm : undefined, hydrating)
    }
  },

  prepatch (oldVnode: MountedComponentVNode, vnode: MountedComponentVNode) {
    const options = vnode.componentOptions
    const child = vnode.componentInstance = oldVnode.componentInstance
    updateChildComponent(
      child,
      options.propsData, // updated props
      options.listeners, // updated listeners
      vnode, // new parent vnode
      options.children // new children
    )
  },

  insert (vnode: MountedComponentVNode) {
    
  },

  destroy (vnode: MountedComponentVNode) {
 
  }
}
```
第一次进入的时候，因为`vnode.componentInstance`还没有赋值，走到else的逻辑，执行`$mount`方法。走完else逻辑，注意此时`vnode.componentInstance`已经被赋值了。

继续回到`createComponent`方法，`isDef(vnode.componentInstance)`为`true`，所以继续下面的逻辑，`initComponent`方法里面赋值 `vnode.elm`。紧接着，将转换为真实DOM的节点，插入到页面中，` insert(parentElm, vnode.elm, refElm)`。此时 `isReactivated` 还是为 `false`，第一次渲染`keep-alive`执行结束。

总结下第一次渲染：
除了以下不同点，其他和渲染一个普通的组件，没有任何区别：
- 如果是`keep-alive`组件，会在内存里缓存下`vNode`。

当我们从别的组件第二次再进入该缓存组件的时候，就会发生一些变化。

我们知道，当数据发生变化，在 `patch` 的过程中会执行 `patchVnode` 的逻辑，它会对比新旧 vnode 节点，甚至对比它们的子节点去做更新逻辑，但是对于组件 `vnode` 而言，是没有 `children` 的，那么对于 `<keep-alive>` 组件而言，如何更新它包裹的内容呢？

```js
// src/core/vdom/patch.js
function patchVnode () {
  // ...
  if (isDef(data) && isDef(i = data.hook) && isDef(i = i.prepatch)) {
    i(oldVnode, vnode)
  }
  // ...
}
```
这里面会再次执行`componentVNodeHooks.prepatch`方法，

```js
// src/core/vdom/create-component.js
prepatch (oldVnode: MountedComponentVNode, vnode: MountedComponentVNode) {
  const options = vnode.componentOptions
  const child = vnode.componentInstance = oldVnode.componentInstance
  updateChildComponent(
    child,
    options.propsData, // updated props
    options.listeners, // updated listeners
    vnode, // new parent vnode
    options.children // new children
  )
},
```
```js
// src/core/instance/lifecycle.js
export function updateChildComponent (
  vm: Component,
  propsData: ?Object,
  listeners: ?Object,
  parentVnode: MountedComponentVNode,
  renderChildren: ?Array<VNode>
) {
  // Any static slot children from the parent may have changed during parent's
  // update. Dynamic scoped slots may also have changed. In such cases, a forced
  // update is necessary to ensure correctness.
  const needsForceUpdate = !!(
    renderChildren ||               // has new static slots
    vm.$options._renderChildren ||  // has old static slots
    hasDynamicScopedSlot
  )

  // ...
  if (needsForceUpdate) {
    vm.$slots = resolveSlots(renderChildren, parentVnode.context)
    vm.$forceUpdate()
  }
}
```
`updateChildComponent` 方法主要是去更新组件实例的一些属性，这里我们重点关注一下 slot 部分，由于 `<keep-alive>` 组件本质上支持了 slot，所以它执行 `prepatch` 的时候，需要对自己的 `children`，也就是这些 `slots` 做重新解析，并触发 `<keep-alive>` 组件实例 `$forceUpdate` 逻辑，也就是重新执行 `<keep-alive>` 的 `render` 方法，这个时候如果它包裹的第一个组件 `vnode` 命中缓存，则直接返回缓存中的 `vnode.componentInstance`，接着又会执行 `patch` 过程，再次执行到 `createComponent` 方法，

这时候 `isReactivated` 为 `true` 了，
接着执行 `componentVNodeHooks.init` 方法，这时候会走到`if`里，执行`prepatch`方法（是的，又会执行一次`updateChildComponent`，源码调试也确实会执行两次这个方法，具体用途不知）。**注意**，
第二次渲染，就不会触发组件的 `created`、`mounted` 等钩子了。因为并没有执行`$mount`。
```js
// src/core/vdom/create-component.js
const componentVNodeHooks = {
  init (vnode: VNodeWithData, hydrating: boolean): ?boolean {
    if (
      vnode.componentInstance &&
      !vnode.componentInstance._isDestroyed &&
      vnode.data.keepAlive
    ) {
      // kept-alive components, treat as a patch
      // keep-alive 组件
      const mountedNode: any = vnode // work around flow
      componentVNodeHooks.prepatch(mountedNode, mountedNode)
    } else {
      // 正常component
      const child = vnode.componentInstance = createComponentInstanceForVnode(
        vnode,
        activeInstance
      )
      // 执行$mount，进行模版解析并且转化经过优化成虚拟dom，生成render函数，并且处罚created、mounted声明周期
      child.$mount(hydrating ? vnode.elm : undefined, hydrating)
    }
  }
}
```
`createComponent`方法继续往下执行，走到`reactivateComponent`。





