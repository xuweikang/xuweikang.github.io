---
layout: post
title: vue3源码 -- 1. 项目结构和开发环境
tags: vue3 源码
categories: SourceCodeAnalysis
---

* TOC 
{:toc}

## 1. Vue 2和Vue 3的项目结构比较
vue2的项目结构如下：

![image.png](/static/img/vue3/start/1.png)

### 1.1 核心库
- Vue 2的核心库位于src/core目录，包含了Vue的核心功能和逻辑，如数据绑定、虚拟DOM、组件系统等。
- Vue 2的代码库采用了单体应用架构(Monolithic)，所有核心功能都包含在一个巨大的src目录下，这使得代码的维护和理解变得相对困难。

### 1.2 运行时
- 运行时构建是Vue 2的一个独立部分，它包括运行时版本的Vue.js，用于在浏览器中运行的应用程序。
- 运行时构建不包含编译器，因此需要在构建时将Vue模板编译为渲染函数。
- 在 `src/platforms/web/runtime` 下。

### 1.3 编译器
用于将Vue模板编译成渲染函数。`src/compiler`。

### 1.4 辅助工具和插件
Vue 2还包含了一些辅助工具和插件，如Vue Router、Vuex等，它们都是Vue生态系统的一部分。需要npm引入。

Vue 3的项目结构完全不一样，项目组织也完全不同：

![image.png](/static/img/vue3/start/2.png)

可以看到，项目解构更扁平化了，并且各种功能都拆分到packages目录下了。

### 1.1 核心库
- Vue 3的核心库位于packages目录，它将核心功能划分为多个独立的包，每个包都有明确定义的职责
- 这种模块化的结构使得理解和维护代码变得更容易，同时也为未来的扩展和优化提供了更大的灵活性。

### 1.2 运行时
与Vue 2不同的是，Vue 3的运行时构建已经包括了编译器，这意味着它能够在客户端动态编译模板，减少了构建时的复杂性。

### 1.3 编译器
Vue 3的编译器成为了独立的包，位于`packages/compiler-core`目录下，与核心库相分离。

### 1.4 辅助工具和插件
与Vue 2一样，Vue 3仍然支持众多辅助工具和插件，但它们通常也采用了更模块化的结构。

vue3项目结构的变化，采用了更好的模块化结构，使得项目代码更好维护和扩展。而且也比vue2代码更便于阅读。（vue2源码读起来层级很深，有时候会看了后面找不到前面的代码位置）。但是各自侧重点不同，传统的单体应用架构(Monolithic）组织架构，更适用于规模不大的小型项目。而后者更便于大型项目进行功能拆分。

## 2. Monorepo架构

目前几乎所有的大型前端开源项目，都采用该架构进行项目搭建了。比如 Google、Facebook、React、Vue3等等。
既然大家都用它，那它究竟有啥“过人之处”呢。

#### 发展历程可用下面这个图来概括：

![image.png](/static/img/vue3/start/3.png)

从初期的业务系统不复杂，只用一个仓库来管理项目，项目为单体应用架构 `Monolithic。`

到后来发现项目越来越大，业务越来越复杂，导致了一系列的问题：比如项目编译速度变慢（调试成本变大）、部署效率/频率低（非业务开发耗时增加）、很多公共的模块不需要每次都重新编译打包，技术债务越积越多。这时候迫切需要组件化/模块化开方来分割下这个项目。

后来大家都把公共的部分分割成一个独立的仓库，然后通过npm引入，这样的优势就是每个仓库都能独立进行各模块的编码、测试和发版，又能实现多项目共享代码，开方人员的关注点分离。这种管理模式称之为多仓多模块管理 `Multirepo。`

再随着时间的沉淀，模块数量也在飞速增长。Multirepo 这种方式虽然从业务逻辑上解耦了，但也同时增加了项目的工程管理难度。比如：
1. 代码和配置很难共享：每个仓库都需要做一些重复的工程化能力配置（如 eslint/test/ci 等）且无法统一维护，当有工程上的升级时，没能同步更新到所有涉及模块，就会一直存在一个过渡态的情况，对工程的不断优化非常不利。
2. 依赖的治理复杂：模块越来越多，涉及多模块同时改动的可能性急剧增加。如何保障底层组件升级后，其引用到的组件也能同步更新到位。这点很难做到，如果没及时升级，各工程的依赖版本不一致，往往会引发一些意想不到的问题。
3. 存储和构建消耗增加：假如多个工程依赖 pkg-a，那么每个工程下 node_modules 都会重复安装 pkg-a，对本地磁盘内存和本地启动都是个很大的挑战，增加了开发时调试的困难。而且每个模块的发布都是相对独立的，当一次迭代修改较多模块时，总体发布时效就是每个发布流程的串联。对发布者来说是一个非常大的负担。

#### 这时候就诞生了**Monolithic单体应用架构，其实也就是上面两种方式的结合。**

**Pnpm的worksoace 实现monorepo**
> pnpm 内置了对单一存储库（也称为多包存储库、多项目存储库或单体存储库）的支持， 你可以创建一个 workspace 以将多个项目合并到一个仓库中。

一个 workspace 的根目录下必须有 `pnpm-workspace.yaml` 文件， 也可能会有 `.npmrc` 文件。

下面是一个`pnpm-workspace.yaml`文件例子：

```ts
packages:
  # all packages in direct subdirs of packages/
  - 'packages/*'
  # all packages in subdirs of components/
  - 'components/'
  # exclude packages that are inside test directories
  - '!/test/**'
```

文件很简单，就是告诉pnpm，哪些目录需要被定义为Workspace目录，可以使用通配符。

#### pnpm天然支持monorepo架构。

### Workspace 协议 (workspace:)

默认情况下，如果可用的 packages 与已声明的可用范围相匹配，pnpm 将从工作区链接这些 packages。 例如, 如果bar引用"foo": "^1.0.0"并且foo@1.0.0存在工作区，那么pnpm会从工作区将foo@1.0.0链接到bar。 但是，如果 bar 的依赖项中有 "foo": "2.0.0"，而 foo@2.0.0 在工作空间中并不存在，则将从 npm registry 安装 foo@2.0.0 。 这种行为带来了一些不确定性。

幸运的是，pnpm 支持 workspace 协议 workspace: 。 当使用此协议时，pnpm 将拒绝解析除本地 workspace 包含的 package 之外的任何内容。 因此，如果您设置为 "foo": "workspace:2.0.0" 时，安装将会失败，因为 "foo@2.0.0" 不存在于此 workspace 中。

所以，**workspace是安全的，能避免产生意料之外的包引入错误。**

### 如何使用？

假如 workspace 中有两个包：

```ts
+ packages
    + foo
    + bar
```
如果需要引入这两个包，需要在子包的package.json的dependencies或者devDependencies里定义：

常规使用："foo": "workspace:*"

### 举个例子：

1. 新建目录 monorepoTest，pnpm init进行初始化生成package.json，设置 private 为true，因为monorepo包通常不容许发布。
    ```ts
  {
  "name": "monorepo-test",
  "private": true,
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "dev": "node scripts/dev.js reactivity -f global"
  },
  "keywords": [],
  "author": "",
  "license": "ISC",
  "dependencies": {
    "esbuild": "^0.19.5",
    "minimist": "^1.2.8"
  }
}
  ```

2. 新建 tsconfig.json，  配置 "@my-vue/*" 路径别名

    ```ts
    {
      "compilerOptions": {
        "outDir": "dist", // 输出的目录
        "sourceMap": true, // 开启 sourcemap
        "target": "es2016", // 转译的目标语法
        "module": "esnext", // 模块格式
        "moduleResolution": "node", // 模块解析方式
        "strict": false, // 关闭严格模式，就能使用 any 了
        "resolveJsonModule": true, // 解析 json 模块
        "esModuleInterop": true, // 允许通过 es6 语法引入 commonjs 模块
        "jsx": "preserve", // jsx 不转义
        "lib": ["esnext", "dom"], // 支持的类库 esnext及dom
        "baseUrl": ".",  // 当前目录，即项目根目录作为基础目录
        "paths": { // 路径别名配置
          "@my-vue/*": ["packages/*/src"]  // 当引入 @my-vue/时，去 packages/*/src中找
        },
      }
    }
    ```
3. 新建文件pnpm-workspace.yaml，设置monorepo。
    ```ts
    packages:
      - 'packages/*'
    ```

4. 紧接着就是创建子包了，新建文件夹packages，并且在其下新建，reactivity、shared、runtimeCore3个子包目录，并且分别 pnpm init  创建package.json 使其真正成为一个单独可发布的子包。

reactivity：
```ts
// package.json
{
  "name": "@my-vue/reactivity",
  "version": "1.0.0",
  "description": "",
  "main": "dist/reactivity.cjs.js",
  "module": "dist/reactivity.esm-bundler.js",
  "buildOptions": {
    "name": "VueReactivity"
  },
  "dependencies": {
    "runtime-core": "workspace:*"
  },
  "keywords": [],
  "author": "",
  "license": "ISC"
}
```

```ts
// src/index.js
import { isObject } from '@my-vue/shared'
import { printConsole  } from 'runtime-core'
const obj = {name: 'Vue3'}
console.log(isObject(obj))
printConsole()
```

Shared:
```ts
// package.json
{
  "name": "@my-vue/shared",
  "version": "1.0.0",
  "description": "",
  "main": "dist/shared.cjs.js",
  "module": "dist/shared.esm-bundler.js",
  "keywords": [],
  "author": "",
  "license": "ISC"
}

```
```ts
// src/ index.js
/**
 * 判断对象
 */
export const isObject = (value) =>{
  return typeof value === 'object' && value !== null
}
/**
* 判断函数
*/
export const isFunction= (value) =>{
  return typeof value === 'function'
}
/**
* 判断字符串
*/
export const isString = (value) => {
  return typeof value === 'string'
}
```

runtimeCore:

```ts
// package.json
{
  "name": "@my-vue/shared",
  "version": "1.0.0",
  "description": "",
  "main": "dist/shared.cjs.js",
  "module": "dist/shared.esm-bundler.js",
  "keywords": [],
  "author": "",
  "license": "ISC"
}

```
```ts
// src/index.js
export const printConsole = () => {
  console.log('runtimeCore')
}
```

主要逻辑在reactivity，

```ts
import { isObject } from '@my-vue/shared'
import { printConsole  } from 'runtime-core'
```

这两种导入方式都是本地导入的，@my-vue/shared是通过tsconfig配置目录的，而runtime-core是通过workspace协议进行导入的。

5. 打包脚本

```ts
// script/dev.js：
// 使用 minimist 解析命令行参数
const args = require('minimist')(process.argv.slice(2))
const path = require('path')
// 使用 esbuild 作为构建工具
const { build } = require('esbuild')
// 需要打包的模块。默认打包 reactivity 模块
const target = args._[0] || 'reactivity'
// 打包的格式。默认为 global，即打包成 IIFE 格式，在浏览器中使用
const format = args.f || 'global'
// 打包的入口文件。每个模块的 src/index.ts 作为该模块的入口文件
const entry = path.resolve(__dirname, ../packages/${target}/src/index.ts)
// 打包文件的输出格式
const outputFormat = format.startsWith('global') ? 'iife' : format === 'cjs' ? 'cjs' : 'esm'
// 文件输出路径。输出到模块目录下的 dist 目录下，并以各自的模块规范为后缀名作为区分
const outfile = path.resolve(__dirname, ../packages/${target}/dist/${target}.${format}.js)
// 读取模块的 package.json，它包含了一些打包时需要用到的配置信息
const pkg = require(path.resolve(__dirname, ../packages/${target}/package.json))
// buildOptions.name 是模块打包为 IIFE 格式时的全局变量名字
const pgkGlobalName = pkg?.buildOptions?.name
console.log('模块信息：\n', entry, '\n', format, '\n', outputFormat, '\n', outfile)
// 使用 esbuild 打包
build({
  // 打包入口文件，是一个数组或者对象
  entryPoints: [entry], 
  // external: [...Object.keys(pkg.dependencies || {})],
  // 输入文件路径
  outfile, 
  // 将依赖的文件递归的打包到一个文件中，默认不会进行打包
  bundle: true, 
  // 开启 sourceMap
  sourcemap: true,
  // 打包文件的输出格式，值有三种：iife、cjs 和 esm
  format: outputFormat, 
  // 如果输出格式为 IIFE，需要为其指定一个全局变量名字
  globalName: pgkGlobalName, 
  // 默认情况下，esbuild 构建会生成用于浏览器的代码。如果打包的文件是在 node 环境运行，需要将平台设置为node
  platform: format === 'cjs' ? 'node' : 'browser',
}).then(() => {
  console.log('watching ...')
})
```

## 3.  Vue源码

执行 `npm run dev`   ，实际执行的是 `node scripts/dev.js`

script/dev.js:

关键代码如下：

```ts
// 。。。省略
const target = args._[0] || 'vue'
const format = args.f || 'global'
const pkg = require(`../packages/${target}/package.json`)
// resolve output
const outputFormat = format.startsWith('global')
  ? 'iife'
  : format === 'cjs'
  ? 'cjs'
  : 'esm'
// 。。。省略
const outfile = resolve(
    __dirname,
    `../packages/${target}/dist/${
    target === 'vue-compat' ? `vue` : target
    }.${postfix}.js`
)
// 。。。省略
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
  .then(ctx => ctx.watch())
```


