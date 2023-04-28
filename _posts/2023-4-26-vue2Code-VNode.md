---
layout: post
title: Vue2框架源码分析-虚拟dom
tags: Vue2 源码
categories: SourceCodeAnalysis
---

* TOC 
{:toc}

### 定义

浏览器中，操作真实DOM的开销其实特别大的。频繁的删除，修改，和添加都会增加特别大的开销。如何避免频繁的操作DOM就是 **虚拟DOM** 的作用。
虚拟DOM就是一个保存着真实DOM的结构和属性的一个对象，vue2 中是以组件为粒度的，也就是每一个组件都会创建生成一个虚拟DOM来进行diff更新。

### 可能的疑问？

#### 1. 虚拟DOM为何能提高页面更新性能？

举个例子，
old Dom:

    <ul>
       <li> A <li>
       <li> B <li>
       <li> C <li>
       <li> D <li>
    </ul>

new Dom:

    <ul>
       <li> A <li>
       <li> BB <li>
       <li> D <li>
       <li> C </li>
    </ul>

上述页面上dom的变化，有节点被删除(remove)，有节点被添加(append)，也有移动。如果给真实DOM去操作，会触发3次删除操作、3次添加操作。但是如果先转换成虚拟DOM，再进行DOM操作，只需要1次添加，1次删除和1次移动就行了。

#### 2. 虚拟DOM的节点信息怎么和真实DOM联系起来的

比如我们在对组件的dom树进行比对的时候，发现了一个子节点比如A节点发生了改变，需要删除它，直接操作真实DOM删除。那么vue，是如何准确找到具体是哪个真实节点的呢？
答：根据\_el属性，初始化创建虚拟DOM的时候，就已经将真实节点挂载在了\_el里了。

### VNnode

VNode 是vue源码里的一个类，这个类用于初始化和创建不同的真实DOM元素。vnode有很多类型，

*   注释节点。isComment为true。
*   文本节点。只有一个text属性。
*   克隆节点
*   元素节点。tag、data、children、context
*   组件节点。componentOptions、componentInstance
*   函数式节点。和组件节点类似。

### patch

> 对比两个vnode之间的差异只是patch的一部分，这是手段，不是目的。

patch的目的修改真实DOM，也就是渲染真实DOM。patch并不是暴力替换，而是在现有DOM结构里最大程度的合并一些修改操作来达到更新渲染视图。
对现有DOM的修改处理无非只有三种方式：

*   创建新增的节点

*   删除废弃的节点

*   更新已有的节点

![image.png](/static/img/vue2/patch/patch1.png)

### 1. 新增节点
新增节点，
1. 如果是元素节点(有tag属性), 先用document.createElement()生成元素，然后将这个元素appendChild到真实dom。如果有children属性，就递归生成元素。
2. 如果是注释节点(isComment)，调用createComment;
3. 如果是文本节点 调用document.createTextNode（）创建元素。

![image.png](/static/img/vue2/patch/patch2.png)

### 2. 删除节点

```js
// 删除vnodes数组，startIdx到endIdx的所有内容
function removeVnodes (vnodes, startIdx, endIdx) {
    for (; startIdx <= endIdx; ++startIdx) {
      const ch = vnodes[startIdx]
      if (isDef(ch)) {
        if (isDef(ch.tag)) {
          removeAndInvokeRemoveHook(ch)
          invokeDestroyHook(ch)
        } else { // Text node
          removeNode(ch.elm)
        }
      }
    }
  }
```

```js
// 删除单个节点
function removeNode (el) {
    const parent = nodeOps.parentNode(el)
    // element may have already been removed due to v-html / v-text
    if (isDef(parent)) {
      nodeOps.removeChild(parent, el)
    }
  }
```

### 3. 更新节点
如果当新旧节点是相同节点的时候，需要对两个节点进行细致的对比，然后对老节点在页面上真实的节点位置进行更新。
更新节点比较麻烦，有一大堆的条件判断，如下：

#### 1. 新旧都是静态节点

```js
if (isTrue(vnode.isStatic) && isTrue(oldVnode.isStatic) ) {
  ...
}
```
比如下面的节点，永远不会随状态变化而发生变化，从一开始创建就已经确定，isStatic是被模板编译optimize后添加的属性。
```html
<h1>我是一个静态节点</h1>
```
如果新旧都是静态节点，直接跳过。

#### 2. 新是文本节点

如果新的虚拟节点有text属性，就不用关心旧虚拟节点的类型了，直接调用setTextContent（浏览器环境就是node.textContent）设置节点为新的text，当然得判断下旧节点是否也是文本节点并且文本内容和新的一样，就跳过了。
```js
if (isDef(vnode.text)) {
  if (oldVnode.text !== vnode.text) {
    nodeOps.setTextContent(elm, vnode.text)
  }
}
```

#### 3. 新没有text属性

如果新没有text属性，那么它只可能是元素节点了。

#### 3.1 新有children

如果old没有children，说明old要么有一个文本属性，要么是一个空标签。如果是文本属性，就先清掉文本变成空标签，然后将新的children挨个创建真实DOM，最后插入到这个空标签下。
如果old也有children，那么就需要 **diff算法**了。

#### 3.2 新无children

无text，无children，只能是空标签了。
这种情况，就清空old的子内容，使其也成为空标签。

```js
// 新没有text属性的情况
if (isUndef(vnode.text)) {
  if (isDef(oldCh) && isDef(ch)) {
    if (oldCh !== ch) updateChildren(elm, oldCh, ch, insertedVnodeQueue, removeOnly)
  } else if (isDef(ch)) {
    if (process.env.NODE_ENV !== 'production') {
      checkDuplicateKeys(ch)
    }
    if (isDef(oldVnode.text)) nodeOps.setTextContent(elm, '')
    addVnodes(elm, null, ch, 0, ch.length - 1, insertedVnodeQueue)
  } else if (isDef(oldCh)) {
    removeVnodes(oldCh, 0, oldCh.length - 1)
  } else if (isDef(oldVnode.text)) {
    nodeOps.setTextContent(elm, '')
  }
}
```
下图是源码中的更新节点流程，

![image.png](/static/img/vue2/patch/patch3.png)


### 4. 更新子节点(diff)
只有当新老是同一个节点，并且都有children属性并且不相同的时候才会存在diff过程。

更新真实DOM的子节点可以分为4种操作，新增节点、更新节点、删除节点、移动节点。
对比两个children，是用两层循环实现的。
伪代码如下：
```
for 新children:
  var node = 新节点;
  for 旧children:
     if (oldNode == node) {
       if (所处位置相同) {
        就做更新处理
       } else {
        移动节点
       }
       break
     }
  if (找不到旧节点) {
    新增node追加到页面DOM
  }
```

#### 4.1  新增节点
如果遍历旧children都找不到该节点，则这个节点就应该是新增的，需要将该节点插入到 **所有未处理节点的最前面**。

![image.png](/static/img/vue2/patch/patch4.png)

为什么是插入到 **所有未处理节点的最前面** ，因为我们是按照newChildren的顺序遍历处理的，所以在页面的真实dom里，也应该是按照这个顺序进行处理，那为什么不是 **放在已处理节点** 的后面？
乍一看，如果按照上面的那个图示例子，貌似这种说法也没有什么问题。

看下面这个例子，

![image.png](/static/img/vue2/patch/patch5.png)

newChildren的最后两个节点，都是新增的节点，所以如果是按照  **放在已处理节点** ，那么将会像上图那样展示出来，第三和第四节点的位置反了。

因为虚拟dom diff，是和oldChildren的对比，而不是真实节点对比，也就是上方伪代码的第二个for是对oldChildren的循环，不是真实节点。

所以oldChildren的已处理节点，只有前两个是已处理的， 即使newChildren的第三个已经追加到后面了，第四个追加的时候如果还是在最后一个已处理的后面，那就会有顺序问题。

#### 4.2  更新子节点
如果两个节点是同一个节点，并且位置相同，就直接按照上述规则更新节点就行了。
![image.png](/static/img/vue2/patch/patch6.png)

那如果是节点相同，但是位置不同，就需要移动节点，以达到组件复用。

#### 4.3  移动子节点
![image.png](/static/img/vue2/patch/patch7.png)

如果移动子节点，只需要将老节点移动到所有未处理节点的最前面就行了，原理和新增节点一样。

#### 4.4  删除节点

上述在新增节点的时候，会一直将新节点追加到真实dom里，这样的话真实DOM就会变的越来越长，所以还有一个删除操作。
代码中实现删除节点就是，遍历完所有新节点以后，剩下的如果还有未处理的节点，那这些节点就是要被删除的。

### 5. 更新优化

通常情况下，一个列表中并不是每个节点位置都是发生改变需要被移动的，更多的情况下是有一些位置是没有发生变化的，如果我们都是统一用嵌套循环来处理，会导致很多计算浪费。
vue使用了一套快速查找的方法：
- newStart 和 oldStart
- newEnd 和 oldEnd
- newEnd 和 oldStart
- newStart 和 oldEnd

newStart和newEnd，分别表示newChildren所有未处理节点的第一个和最后一个

oldStart和oldEnd，分别表示oldChildren所有未处理节点的第一个和最后一个

#### 5.1 newStart 和 oldStart
先对比第一个节点，如果新老第一个节点正好是同一个节点，直接更新即可，否则，继续走下面的判断。

#### 5.2 newEnd 和 oldEnd
如果新老最后一个节点正好是同一个节点，直接更新即可，否则，继续走下面的判断。

![image.png](/static/img/vue2/patch/patch8.png)

#### 5.3 newEnd 和 oldStart
如果这两个节点是同一个节点的话，就先按照更新规则进行更新，然后将这个节点移动到 **真实DOM的未处理节点的最后面** 。否则，继续走下面的判断。

![image.png](/static/img/vue2/patch/patch9.png)

#### 5.4 newStart 和 oldEnd
如果这两个节点是同一个节点的话，就先按照更新规则进行更新，然后将这个节点移动到 **真实DOM的未处理节点的最前面** 。否则，就只能走循环找一遍了。

![image.png](/static/img/vue2/patch/patch10.png)



### 6. 从源码角度分析patch

Vue.prototype.$mount方法实现的时候，有一段比较重要的代码，就从这里开始承上启下吧。

```js
function mountComponent() {
  // 省略代码...
  callHook(vm, 'beforeMount')
  // 省略代码...
  let updateComponent = () => {
    vm._update(vm._render(), hydrating) // vm._render（）拿到VNode，vm_update虚拟DOM对比，并更新到真实DOM。
  }
  new Watcher(vm, updateComponent, noop, {
    before () {
      if (vm._isMounted && !vm._isDestroyed) {
        callHook(vm, 'beforeUpdate')
      }
    }
  }, true /* isRenderWatcher */)
  // 省略代码...
  callHook(vm, 'mounted')
}
```

上述代码，实例化了一个Watcher（被订阅者），第二个参数传人一个updateComponent方法，在Watcher被new的时候，会执行这个方法，并且这时候也开始进行依赖收集，如果依赖更新会通知到这个Watcher，执行update方法进行更新。
updateComponent方法究竟发生了什么呢？

#### 6.1 _render

下面一步步追溯 `_redner` 方法的定义，

找到原型方法的最初定义位置，core/insatance/render.js
```js
Vue.prototype._render = function (): VNode { 
  const { render, _parentVnode } = vm.$options
  // 省略代码...
  vnode = render.call(vm._renderProxy, vm.$createElement)
  // 省略代码...
  rerurt vnode
}
```
**_render方法返回了一个 vnode， vnode是通过 vm.$options.render方法生成的。**

继续找vm.$options.render的定义，
在 entry-runtime-with-compiler.js，

```js
  // compileToFunctions函数比较绕，最终返回render函数，和staticRenderFns数组
  const { render, staticRenderFns } = compileToFunctions(template, {
    outputSourceRange: process.env.NODE_ENV !== 'production',
    shouldDecodeNewlines,
    shouldDecodeNewlinesForHref,
    delimiters: options.delimiters,
    comments: options.comments
  }, this)
  options.render = render
```
找到vm.$options.render的定义，在 **Vue2框架源码分析-模板编译原理** 里有具体分析。

#### 6.2 _update
下面一步步追溯 `_update` 方法的定义，

找到原型方法最初定义，core/insatance/lifecycle.js

```js
Vue.prototype._update = function() {
  // 省略代码...
  if (!prevVnode) {
    // 首次渲染
    vm.$el = vm.__patch__(vm.$el, vnode, hydrating, false /* removeOnly */)
  } else {
    // 更新dom
    vm.$el = vm.__patch__(prevVnode, vnode)
  }
  // 省略代码...
}
```
_update原型方法里，通过调用实例方法`__patch__`，然后挂载到$el属性上。

继续找`__patch__`方法的定义，在 web/runtime/index.js

```js
Vue.prototype.__patch__ = inBrowser ? patch : noop
```
patch方法的定义，在  web/runtime/patch.js

```js
export const patch: Function = createPatchFunction({ nodeOps, modules })
```
所以核心逻辑，都是在 `createPatchFunction` 这里面， 该方法所处在文件， core/vdom/patch.js

该方法内部定义了很多函数，最后 return 了一个`patch`方法。

这个patch方法里面的源码所描述的，就是上面的流程。

```js
return function patch (oldVnode, vnode, hydrating, removeOnly) {
    if (isUndef(vnode)) {
      if (isDef(oldVnode)) invokeDestroyHook(oldVnode)
      return
    }

    let isInitialPatch = false
    const insertedVnodeQueue = []

    if (isUndef(oldVnode)) {
      // 如果没有老节点，直接创建新节点并插入
      isInitialPatch = true
      createElm(vnode, insertedVnodeQueue)
    } else {
      const isRealElement = isDef(oldVnode.nodeType)
      if (!isRealElement && sameVnode(oldVnode, vnode)) {
        // 如果是同一个节点，并且不是真实dom，需要走 patchVnode 逻辑
        patchVnode(oldVnode, vnode, insertedVnodeQueue, null, null, removeOnly)
      } else {
        if (isRealElement) {
          // mounting to a real element
          // check if this is server-rendered content and if we can perform
          // a successful hydration.
          if (oldVnode.nodeType === 1 && oldVnode.hasAttribute(SSR_ATTR)) {
            oldVnode.removeAttribute(SSR_ATTR)
            hydrating = true
          }
          if (isTrue(hydrating)) {
            // ssr相关逻辑
            if (hydrate(oldVnode, vnode, insertedVnodeQueue)) {
              invokeInsertHook(vnode, insertedVnodeQueue, true)
              return oldVnode
            } else if (process.env.NODE_ENV !== 'production') {
              warn(
                'The client-side rendered virtual DOM tree is not matching ' +
                'server-rendered content. This is likely caused by incorrect ' +
                'HTML markup, for example nesting block-level elements inside ' +
                '<p>, or missing <tbody>. Bailing hydration and performing ' +
                'full client-side render.'
              )
            }
          }
          // either not server-rendered, or hydration failed.
          // create an empty node and replace it
          oldVnode = emptyNodeAt(oldVnode) // 把RealElement创建一个 VNode，包在一个空节点下面
        }

      
        // replacing existing element
        const oldElm = oldVnode.elm
        const parentElm = nodeOps.parentNode(oldElm)

        // 1. 创建新组件，并插入到dom中
        createElm(
          vnode,
          insertedVnodeQueue,
          // extremely rare edge case: do not insert if old element is in a
          // leaving transition. Only happens when combining transition +
          // keep-alive + HOCs. (#4590)
          oldElm._leaveCb ? null : parentElm,
          nodeOps.nextSibling(oldElm)
        )

        // update parent placeholder node element, recursively
        if (isDef(vnode.parent)) {
          let ancestor = vnode.parent
          const patchable = isPatchable(vnode) // 是否是可挂载节点（是否有tag）
          while (ancestor) {
            for (let i = 0; i < cbs.destroy.length; ++i) {
              cbs.destroy[i](ancestor)
            }
            ancestor.elm = vnode.elm
            if (patchable) {
              for (let i = 0; i < cbs.create.length; ++i) {
                cbs.create[i](emptyNode, ancestor)
              }
              // #6513
              // invoke insert hooks that may have been merged by create hooks.
              // e.g. for directives that uses the "inserted" hook.
              const insert = ancestor.data.hook.insert
              if (insert.merged) {
                // start at index 1 to avoid re-invoking component mounted hook
                for (let i = 1; i < insert.fns.length; i++) {
                  insert.fns[i]()
                }
              }
            } else {
              registerRef(ancestor)
            }
            ancestor = ancestor.parent
          }
        }

        // destroy old node
        if (isDef(parentElm)) {
          removeVnodes([oldVnode], 0, 0) // 删除老节点
        } else if (isDef(oldVnode.tag)) {
          invokeDestroyHook(oldVnode)
        }
      }
    }

    invokeInsertHook(vnode, insertedVnodeQueue, isInitialPatch)
    return vnode.elm
  }
}
```

patchVnode

```js
function patchVnode (
    oldVnode,
    vnode,
    insertedVnodeQueue,
    ownerArray,
    index,
    removeOnly
  ) {
    if (oldVnode === vnode) {
      // 如果是同一个vnode
      return
    }

    if (isDef(vnode.elm) && isDef(ownerArray)) {
      // 克隆节点相关的逻辑，不懂
      vnode = ownerArray[index] = cloneVNode(vnode)
    }

    const elm = vnode.elm = oldVnode.elm

    if (isTrue(oldVnode.isAsyncPlaceholder)) {
      if (isDef(vnode.asyncFactory.resolved)) {
        hydrate(oldVnode.elm, vnode, insertedVnodeQueue)
      } else {
        vnode.isAsyncPlaceholder = true
      }
      return
    }

    // 如果是静态节点，直接return
    if (isTrue(vnode.isStatic) &&
      isTrue(oldVnode.isStatic) &&
      vnode.key === oldVnode.key &&
      (isTrue(vnode.isCloned) || isTrue(vnode.isOnce))
    ) {
      vnode.componentInstance = oldVnode.componentInstance
      return
    }

    let i
    const data = vnode.data
    // 执行componentVNodeHooks.prepatch 方法，这个方法就是拿到新的 vnode 的组件配置以及组件实例
    if (isDef(data) && isDef(i = data.hook) && isDef(i = i.prepatch)) {
      i(oldVnode, vnode)
    }

    const oldCh = oldVnode.children
    const ch = vnode.children
    if (isDef(data) && isPatchable(vnode)) {
      // 执行update钩子函数
      for (i = 0; i < cbs.update.length; ++i) cbs.update[i](oldVnode, vnode) // 系统的update钩子
      if (isDef(i = data.hook) && isDef(i = i.update)) i(oldVnode, vnode) // 用户自定义的update钩子
    }
    if (isUndef(vnode.text)) {
      // 不是文本节点
      if (isDef(oldCh) && isDef(ch)) {
        // 都有children
        if (oldCh !== ch) updateChildren(elm, oldCh, ch, insertedVnodeQueue, removeOnly) 
      } else if (isDef(ch)) {
        // old没有children，new 有children
        if (process.env.NODE_ENV !== 'production') {
          checkDuplicateKeys(ch) // check 是否设置了重名的key值
        }
        if (isDef(oldVnode.text)) nodeOps.setTextContent(elm, '') // old如果有text属性的话，先清空
        addVnodes(elm, null, ch, 0, ch.length - 1, insertedVnodeQueue) // 全插入
      } else if (isDef(oldCh)) {
        // old有children，new没有children
        removeVnodes(oldCh, 0, oldCh.length - 1) // 需要递归全部删除children
      } else if (isDef(oldVnode.text)) {
        nodeOps.setTextContent(elm, '')
      }
    } else if (oldVnode.text !== vnode.text) {
      // 文本节点，并且文本不相同，直接setTextContent新内容即可
      nodeOps.setTextContent(elm, vnode.text)
    }
    if (isDef(data)) {
      if (isDef(i = data.hook) && isDef(i = i.postpatch)) i(oldVnode, vnode)
    }
  }
```

updateChildren
```js
// 新老节点不相同的
  function updateChildren (parentElm, oldCh, newCh, insertedVnodeQueue, removeOnly) {
    let oldStartIdx = 0
    let newStartIdx = 0
    let oldEndIdx = oldCh.length - 1
    let oldStartVnode = oldCh[0]
    let oldEndVnode = oldCh[oldEndIdx]
    let newEndIdx = newCh.length - 1
    let newStartVnode = newCh[0]
    let newEndVnode = newCh[newEndIdx]
    let oldKeyToIdx, idxInOld, vnodeToMove, refElm

    // removeOnly is a special flag used only by <transition-group>
    // to ensure removed elements stay in correct relative positions
    // during leaving transitions
    const canMove = !removeOnly // 如果初始化渲染，canMove就是false

    if (process.env.NODE_ENV !== 'production') {
      // check 是否设置了重名的key值
      checkDuplicateKeys(newCh)
    }

    while (oldStartIdx <= oldEndIdx && newStartIdx <= newEndIdx) {
      // old节点和new节点同时首尾双指针遍历
      if (isUndef(oldStartVnode)) {
        // old节点移动的过程中，移动前的位置会被赋为undfined
        oldStartVnode = oldCh[++oldStartIdx] 
      } else if (isUndef(oldEndVnode)) {
        // 同上
        oldEndVnode = oldCh[--oldEndIdx]
      } else if (sameVnode(oldStartVnode, newStartVnode)) {
        patchVnode(oldStartVnode, newStartVnode, insertedVnodeQueue, newCh, newStartIdx)
        oldStartVnode = oldCh[++oldStartIdx]
        newStartVnode = newCh[++newStartIdx]
      } else if (sameVnode(oldEndVnode, newEndVnode)) {
        patchVnode(oldEndVnode, newEndVnode, insertedVnodeQueue, newCh, newEndIdx)
        oldEndVnode = oldCh[--oldEndIdx]
        newEndVnode = newCh[--newEndIdx]
      } else if (sameVnode(oldStartVnode, newEndVnode)) { // Vnode moved right
        patchVnode(oldStartVnode, newEndVnode, insertedVnodeQueue, newCh, newEndIdx)
        canMove && nodeOps.insertBefore(parentElm, oldStartVnode.elm, nodeOps.nextSibling(oldEndVnode.elm))
        oldStartVnode = oldCh[++oldStartIdx]
        newEndVnode = newCh[--newEndIdx]
      } else if (sameVnode(oldEndVnode, newStartVnode)) { // Vnode moved left
        patchVnode(oldEndVnode, newStartVnode, insertedVnodeQueue, newCh, newStartIdx)
        canMove && nodeOps.insertBefore(parentElm, oldEndVnode.elm, oldStartVnode.elm)
        oldEndVnode = oldCh[--oldEndIdx]
        newStartVnode = newCh[++newStartIdx]
      } else {
        if (isUndef(oldKeyToIdx)) oldKeyToIdx = createKeyToOldIdx(oldCh, oldStartIdx, oldEndIdx) // 创建一个old {key: index} 的映射对象
        idxInOld = isDef(newStartVnode.key)
          ? oldKeyToIdx[newStartVnode.key] // 如果newCh提供了key，就返回对应的key对应的old index
          : findIdxInOld(newStartVnode, oldCh, oldStartIdx, oldEndIdx) // 如果没有，就只能传入oldStartIdx和oldEndIdx，遍历寻找

        if (isUndef(idxInOld)) { // 找不到的话，是一个新的元素，创建这个元素，注意是：插入到oldStartVnode的前面！！！！！第四个参数就是标注插在哪个元素的前面，需要插在第一个未处理元素的前面
          createElm(newStartVnode, insertedVnodeQueue, parentElm, oldStartVnode.elm, false, newCh, newStartIdx)
        } else {
          vnodeToMove = oldCh[idxInOld] 
          if (sameVnode(vnodeToMove, newStartVnode)) {
            patchVnode(vnodeToMove, newStartVnode, insertedVnodeQueue, newCh, newStartIdx)
            oldCh[idxInOld] = undefined  // 这个oldCh标记为 undefined
            canMove && nodeOps.insertBefore(parentElm, vnodeToMove.elm, oldStartVnode.elm) // 移动到第一个未处理元素的前面
          } else {
            // same key but different element. treat as new element
            createElm(newStartVnode, insertedVnodeQueue, parentElm, oldStartVnode.elm, false, newCh, newStartIdx)
          }
        }
        newStartVnode = newCh[++newStartIdx]
      }
    }
    if (oldStartIdx > oldEndIdx) {
      // 说明old节点已经都处理完了，new节点还有未处理的，需要剩余新节点都插入到dom里
      refElm = isUndef(newCh[newEndIdx + 1]) ? null : newCh[newEndIdx + 1].elm
      addVnodes(parentElm, refElm, newCh, newStartIdx, newEndIdx, insertedVnodeQueue)
    } else if (newStartIdx > newEndIdx) {
      // new节点已经处理完成了，old节点还有未处理的，需要删除这些节点
      removeVnodes(oldCh, oldStartIdx, oldEndIdx)
    }
  }
```

