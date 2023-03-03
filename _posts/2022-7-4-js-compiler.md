---
layout: post
title: js 编译基础（babel）
tags: js 编译 babel 解释器
categories: jsBase
---

* TOC 
{:toc}

### js 编译基础（babel）

babel目前是前端领域的必备工具了，他可以让我们使用一些新的语法和api。babel会在编译的过程中，将一些不支持的api添加polyfill以支持当前环境。

babel更像一种，从源码到源码的转换编译器。

#### 什么是编译器？

编译器（compiler）是一种计算机程序，会将一种编程语言（开发语言）转换成另一种编程语言（目标语言）。

源代码（source code）→ 预处理器（preprocessor）→ 编译器（compiler）→ 汇编程序（assembler）→ 目标代码（object code）→ 链接器（linker）→ 可执行文件（executables），最后打包好的文件就可以给电脑去运行了。

#### 什么是解释器？

解释器（interpreter）是一种计算机程序，能够将解释型语言解释执行。解释器一边解释一遍执行，因此依赖于解释器的语言运行速度慢。解释型语言的好处就是不用编译整个程序，减轻了代码更新后编译的负担。相反，这也是非解释型语言的优缺点。



https://www.teqng.com/2021/07/21/%e5%89%8d%e7%ab%af%e5%ba%94%e8%af%a5%e6%8e%8c%e6%8f%a1%e7%9a%84%e7%bc%96%e8%af%91%e5%9f%ba%e7%a1%80%ef%bc%88%e5%9f%ba%e4%ba%8e-babel%ef%bc%89/