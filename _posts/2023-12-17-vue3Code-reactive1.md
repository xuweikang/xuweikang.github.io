---
layout: post
title: vue3源码 -- 2.  响应式【上】
tags: vue3 源码
categories: SourceCodeAnalysis
---

* TOC 
{:toc}

![image.png](/static/img/vue3/reactivity/1.png)

## 从一个简单例子开始

```ts
<div id="app">
  <button>{{name}}</button>
</div>

<script>

Vue.createApp({
  data() {
    return {
      name: 'hello'
    }
  },
  setup() {
    console.log(123)
  },
  methods: {
    
  }
}).mount('#app')
```

这个例子很简单，一个响应式变量 name，一个setup。但是也足以开始源码之旅。

## Vue.createApp

Vue3和Vue2对Vue这个变量的组织方式不一样。在vue2中，Vue是作为一个构造函数在源码里进行定义的，所以的运行时代码也是基于这个Vue构造函数开始展开的。

但是，Vue3源码并没有Vue构造函数的概念。那么这个Vue.createApp是怎么来的呢？

先看下执行npm run dev打包后的文件结构：

```ts
// vue.global.js
"use strict";
var Vue = (() => {
    // ...代码代码代码代码
})
```

看这个结构，能猜测出了，虽然源码里没有Vue这个定义，但是在打包的时候，是有意的把Vue作为整个全局全量，并且Vue的值为一个自执行函数。而这个逻辑肯定也是在打包脚本里。

scripts/dev.js里，是通过esbuild进行打包的，

```ts
esbuild
  .context({
    entryPoints: [resolve(__dirname, `../packages/${target}/src/index.ts`)],
    outfile,
    bundle: true,
    external,
    sourcemap: true,
    format: outputFormat,
    globalName: pkg.buildOptions?.name,
    platform: format === 'cjs' ? 'node' : 'browser',
    plugins,
    define: {
      __COMMIT__: `"dev"`,
      __VERSION__: `"${pkg.version}"`,
      __DEV__: `true`,
      __TEST__: `false`,
      __BROWSER__: String(
        format !== 'cjs' && !pkg.buildOptions?.enableNonBrowserBranches
      ),
      __GLOBAL__: String(format === 'global'),
      __ESM_BUNDLER__: String(format.includes('esm-bundler')),
      __ESM_BROWSER__: String(format.includes('esm-browser')),
      __NODE_JS__: String(format === 'cjs'),
      __SSR__: String(format === 'cjs' || format.includes('esm-bundler')),
      __COMPAT__: String(target === 'vue-compat'),
      __FEATURE_SUSPENSE__: `true`,
      __FEATURE_OPTIONS_API__: `true`,
      __FEATURE_PROD_DEVTOOLS__: `false`
    }
  })
```

其中globalName，就是Vue变量。

解决了Vue对象的问题，继续看下createApp方法的定义。从调用情况来看么，可以推测出：

1. createApp方法接受选项参数，这和vue2的options选项类似，data、methods、生命周期这些；
2. createApp方法会return出一个mount方法。

**从上面的打包脚本里，能看出入口文件为  packages/vue/src/index.ts，所以源码阅读的开始就是这个文件开始。**

```ts
// packages/vue/src/index.ts

// *** code
// *** code
function compileToFunction() {
  // *** code
  // 和vue2的compile函数类似，主要就是返回编译优化后的render方法字符串
  const { code } = compile(template, opts)
  // *** code
    const render = (
    __GLOBAL__ ? new Function(code)() : new Function('Vue', code)(runtimeDom)
  ) as RenderFunction

  // mark the function as runtime compiled
  ;(render as InternalRenderFunction)._rc = true

  return (compileCache[key] = render)
}
//将编译函数注册到运行时
registerRuntimeCompiler(compileToFunction)

export { compileToFunction as compile }
export * from '@vue/runtime-dom'

```

入口文件内容不多，大致的内容就是，定义了一个`compileToFunction`方法，这个方法和vue2的`compile`方法类似，然后将这个编译函数注册到运行时里，通过`registerRuntimeCompiler`。

最后将这个compileToFunction导出，然后导出 整个 '`@vue/runtime-dom`' 文件。

我们现在的目的是想找到`createApp`的方法定义，但是`compileToFunction` 方法里并没有追踪到相关代码逻辑。所以 唯一可能的，**只能继续找到`@vue/runtime-dom` 里**。

### packages/runtime-core/src/index.ts

在这里找到`createApp`的定义了。

```ts
export const createApp = ((...args) => {
  const app = ensureRenderer().createApp(...args)
  const { mount } = app
  // 拓展mount（和不同平台的mount功能不一致有关，这种写法不会污染公共mount方法定义，避免在mount方法里出现很多平台判断）
  app.mount = (containerOrSelector: Element | ShadowRoot | string): any => {
    // code...
    if (!container) return
    // code...
    container.innerHTML = ''
    const proxy = mount(container, false, container instanceof SVGElement)
    if (container instanceof Element) {
      container.removeAttribute('v-cloak')
      container.setAttribute('data-v-app', '')
    }
    return proxy
  }

  return app
}) as CreateAppFunction<Element>
```

这部分代码能看出，该方法返回了app，并且重新定义了mount方法挂载到app对象上。

到这里能解决前面的问题， `Vue.createApp` 和`mount`方法的逻辑都能找到位置了。

app对象的具体逻辑，都在`ensureRenderer()`里。

## ensureRenderer
```ts
function ensureRenderer() {
  return (
    renderer ||
    (renderer = createRenderer<Node, Element | ShadowRoot>(rendererOptions))
  )
}
export function createRenderer<
  HostNode = RendererNode,
  HostElement = RendererElement
>(options: RendererOptions<HostNode, HostElement>) {
  return baseCreateRenderer<HostNode, HostElement>(options)
}
```

真正的逻辑，在baseCreateRenderer里。

```ts
function baseCreateRenderer(
  options: RendererOptions,
  createHydrationFns?: typeof createHydrationFunctions
): any { 
  // code...
  const render: RootRenderFunction = (vnode, container, isSVG) => {
    if (vnode == null) {
      if (container._vnode) {
        // 之前的dom渲染
        unmount(container._vnode, null, null, true)
      }
    } else {
      patch(container._vnode || null, vnode, container, null, null, null, isSVG)
    }
    flushPreFlushCbs()
    flushPostFlushCbs()
    container._vnode = vnode
  }
  return {
    render,
    hydrate,
    createApp: createAppAPI(render, hydrate)
  }  

}
```

`baseCreateRenderer`里返回了`createApp`，调用的是`createAppAPI`，传入参数 `render`。继续往下看`createAppAPI`，

```ts
export function createAppAPI<HostElement>(
  render: RootRenderFunction<HostElement>,
  hydrate?: RootHydrateFunction
): CreateAppFunction<HostElement> {
  return function createApp(rootComponent, rootProps = null) {
    // code...
    const app: App = (context.app = {
      _uid: uid++,
      _component: rootComponent as ConcreteComponent,
      _props: rootProps,
      _container: null,
      _context: context,
      _instance: null,

      version,

      get config() {
        return context.config
      },

      set config(v) {
       // code...
      },

      use(plugin: Plugin, ...options: any[]) {
        // code...
      },

      mixin(mixin: ComponentOptions) {
        // code...
        return app
      },

      component(name: string, component?: Component): any {
        // code...
        return app
      },

      directive(name: string, directive?: Directive) {
        // code...
        return app
      },

      //挂载
      mount(
        rootContainer: HostElement,
        isHydrate?: boolean,
        isSVG?: boolean
      ): any {
        if (!isMounted) {
        // code...
        // 这里和vue2的render作用不同
        // 1. vue2的render返回一个vNode
        // 2. vue3的render函数是做分发工作的，相当于是一个路由器，两条线路，unmount和patch，无返回结果。
        render(vnode, rootContainer, isSVG)
        return getExposeProxy(vnode.component!) || vnode.component!.proxy
      },

      unmount() {
        // code...
      }
    })
      
 }
}
```

这个方法里，定义了mount的最原始方法。

**到目前为止。已经可以完全解释 Vue.createApp().mount()了。但是目前还没发现响应式的相关代码。**

## 执行render方法

继续往下执行 mount 方法，会执行下方代码：

```ts
render(vnode, rootContainer, isSVG)
```

render方法是调用createAppAPI传入进来的参数，具体的定义在baseCreateRenderer里，

```ts
function baseCreateRenderer(
  options: RendererOptions,
  createHydrationFns?: typeof createHydrationFunctions
): any { 
  // code...
  const render: RootRenderFunction = (vnode, container, isSVG) => {
    if (vnode == null) {
      if (container._vnode) {
        // 之前的dom渲染
        unmount(container._vnode, null, null, true)
      }
    } else {
      patch(container._vnode || null, vnode, container, null, null, null, isSVG)
    }
    flushPreFlushCbs()
    flushPostFlushCbs()
    container._vnode = vnode
  }
  return {
    render,
    hydrate,
    createApp: createAppAPI(render, hydrate)
  }  

}
```

vue3源码的render方法和vue2的render方法不一样，vue2是返回vnode，vue3里是起到分发工作的，相当于是一个路由器，两条线路，unmount和patch，无返回结果。

初次渲染，走到patch方法里。patch方法的定义也在这个文件里。

```ts
const patch: PatchFn = (
    n1,
    n2,
    container,
    anchor = null,
    parentComponent = null,
    parentSuspense = null,
    isSVG = false,
    slotScopeIds = null,
    optimized = __DEV__ && isHmrUpdating ? false : !!n2.dynamicChildren
  ) => {
    if (n1 === n2) {
      return
    }

    // patching & not same type, unmount old tree
    if (n1 && !isSameVNodeType(n1, n2)) {
      anchor = getNextHostNode(n1)
      unmount(n1, parentComponent, parentSuspense, true)
      n1 = null
    }

    if (n2.patchFlag === PatchFlags.BAIL) {
      optimized = false
      n2.dynamicChildren = null
    }

    const { type, ref, shapeFlag } = n2
    switch (type) {
      case Text:
        processText(n1, n2, container, anchor)
        break
      case Comment:
        processCommentNode(n1, n2, container, anchor)
        break
      case Static:
        if (n1 == null) {
          mountStaticNode(n2, container, anchor, isSVG)
        } else if (__DEV__) {
          patchStaticNode(n1, n2, container, isSVG)
        }
        break
      case Fragment:
        processFragment(
          n1,
          n2,
          container,
          anchor,
          parentComponent,
          parentSuspense,
          isSVG,
          slotScopeIds,
          optimized
        )
        break
      default:
        // 首次渲染会走到这里
        debugger
        // 用二进制存储来优化类型查找
        if (shapeFlag & ShapeFlags.ELEMENT) {
          processElement(
            n1,
            n2,
            container,
            anchor,
            parentComponent,
            parentSuspense,
            isSVG,
            slotScopeIds,
            optimized
          )
        } else if (shapeFlag & ShapeFlags.COMPONENT) {
          processComponent(
            n1,
            n2,
            container,
            anchor,
            parentComponent,
            parentSuspense,
            isSVG,
            slotScopeIds,
            optimized
          )
        } else if (shapeFlag & ShapeFlags.TELEPORT) {
          ;(type as typeof TeleportImpl).process(
            n1 as TeleportVNode,
            n2 as TeleportVNode,
            container,
            anchor,
            parentComponent,
            parentSuspense,
            isSVG,
            slotScopeIds,
            optimized,
            internals
          )
        } else if (__FEATURE_SUSPENSE__ && shapeFlag & ShapeFlags.SUSPENSE) {
          ;(type as typeof SuspenseImpl).process(
            n1,
            n2,
            container,
            anchor,
            parentComponent,
            parentSuspense,
            isSVG,
            slotScopeIds,
            optimized,
            internals
          )
        } else if (__DEV__) {
          warn('Invalid VNode type:', type, `(${typeof type})`)
        }
    }

    // set ref
    if (ref != null && parentComponent) {
      setRef(ref, n1 && n1.ref, parentSuspense, n2 || n1, !n2)
    }
  }
```

初次渲染，会走到`processComponent`这个条件分支里。

```ts
const processComponent = (
    n1: VNode | null,
    n2: VNode,
    container: RendererElement,
    anchor: RendererNode | null,
    parentComponent: ComponentInternalInstance | null,
    parentSuspense: SuspenseBoundary | null,
    isSVG: boolean,
    slotScopeIds: string[] | null,
    optimized: boolean
  ) => {
    n2.slotScopeIds = slotScopeIds
    if (n1 == null) {
      if (n2.shapeFlag & ShapeFlags.COMPONENT_KEPT_ALIVE) {
        ;(parentComponent!.ctx as KeepAliveContext).activate(
          n2,
          container,
          anchor,
          isSVG,
          optimized
        )
      } else {
        mountComponent(
          n2,
          container,
          anchor,
          parentComponent,
          parentSuspense,
          isSVG,
          optimized
        )
      }
    } else {
      updateComponent(n1, n2, optimized)
    }
  }
```

```ts
const mountComponent: MountComponentFn = (
    initialVNode,
    container,
    anchor,
    parentComponent,
    parentSuspense,
    isSVG,
    optimized
  ) => {
  
    // code...
    setupComponent(instance) // 这里面会进行响应式初始化
    // code...
    setupRenderEffect(
      instance,
      initialVNode,
      container,
      anchor,
      parentSuspense,
      isSVG,
      optimized
    )
  }
```

setupComponent里会进行响应式的初始化

```ts
export function setupComponent(
  instance: ComponentInternalInstance,
  isSSR = false
) {
  isInSSRComponentSetup = isSSR

  const { props, children } = instance.vnode
  const isStateful = isStatefulComponent(instance)
  debugger
  initProps(instance, props, isStateful, isSSR)
  initSlots(instance, children)

  const setupResult = isStateful
    ? setupStatefulComponent(instance, isSSR)
    : undefined
  isInSSRComponentSetup = false
  return setupResult
}
```
```ts
function setupStatefulComponent(
  instance: ComponentInternalInstance,
  isSSR: boolean
) {
  const Component = instance.type as ComponentOptions
  // code...
  // 0. create render proxy property access cache
  instance.accessCache = Object.create(null)
  // 1. create public instance / render proxy
  // also mark it raw so it's never observed
  instance.proxy = markRaw(new Proxy(instance.ctx, PublicInstanceProxyHandlers))
  // 2. call setup()
  const { setup } = Component
  // 是否定义setup
  if (setup) {
    const setupContext = (instance.setupContext =
      setup.length > 1 ? createSetupContext(instance) : null)

    setCurrentInstance(instance)
    pauseTracking()
    const setupResult = callWithErrorHandling(
      setup,
      instance,
      ErrorCodes.SETUP_FUNCTION,
      [__DEV__ ? shallowReadonly(instance.props) : instance.props, setupContext]
    )
    // code...
    finishComponentSetup(instance, isSSR)
}
```
这段代码主要是执行setup函数，然后执行finishComponentSetup方法，finishComponentSetup里会执行
`applyOptions(instance)` 将data响应式化。

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

至此，我们知道了Vue3是如何将data进行响应式化的，以及从入口文件开始的项目阅读路径。
下一章，会说明响应式的具体原理。