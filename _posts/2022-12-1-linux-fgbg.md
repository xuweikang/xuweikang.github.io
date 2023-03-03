---
layout: post
title: 任务中断和复原
tags: fg bg
categories: linux
---

* TOC 
{:toc}

### 任务中断和复原

linux中，Control + z 和 Control + c 都是**中断** 命令。
不同点：
> Control+z是任务中断，但任务并没有结束，它还在进程中，但状态是维持 **挂起** 状态，可以使用fg或者bg来继续前台或者后台的任务。（fg是重新启动前台被中断的任务，bg是后台重新执行）。