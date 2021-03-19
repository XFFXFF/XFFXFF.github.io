---
title: Trap
date: 2021-03-19 21:45:32
tags: xv6
---
[video](https://www.youtube.com/watch?v=T26UuauaxWA&feature=youtu.be)  
[文字翻译](https://mit-public-courses-cn-translatio.gitbook.io/mit6-s081/lec06-isolation-and-system-call-entry-exit-robert)  
上面关于这节视频课的翻译非常的棒，基本还原的课程的内容，还是我英语太菜了，很多点是看翻译才搞懂的。很多细节还是直接看video和翻译比较好，我这里简单梳理一下trap的核心概念。  

## 什么是Trap
有三种情况会让CPU搁置当下执行的任务，转而去执行一段特殊的代码去处理这个情况：  
* system call
* exception：比如page fault、除以0等
* device interrupt
这三种情况都叫做trap。  

## Traps from user space
Traps的整个过程需要从用户态切换到内核态，再切换回用户态。大概有一下几个步骤：  
1. 保存32个用户寄存器。因为很显然我们需要恢复用户应用程序的执行，尤其是当用户程序随机的被设备中断所打断时。我们希望内核能够响应中断，之后在用户程序完全无感知的情况下再恢复用户代码的执行。所以这意味着32个用户寄存器不能被内核弄乱。但是这些寄存器又要被内核代码所使用，所以在trap之前，你必须先在某处保存这32个用户寄存器。  
2. 程序计数器也需要在某个地方保存，它几乎跟一个用户寄存器的地位是一样的，我们需要能够在用户程序运行中断的位置继续执行用户程序。  
3. 将mode改成supervisor mode，因为我们想要使用内核中的各种各样的特权指令。  
4. 将user page table切换为kernel page table，user page table只包含了用户程序所需要的内存映射和一两个其他的映射，它并没有包含整个内核数据的内存映射。
5. 将堆栈寄存器指向位于内核的一个地址，因为我们需要一个堆栈来调用内核的C函数。
6. 一旦我们设置好了，并且所有的硬件状态都适合在内核中使用， 我们需要跳入内核的C代码。  

做page table lab的时候就一直很困惑，没有在代码中找到
什么时候会切换到user page table，没有看到satp指向
user page table。原来切换的过程发生在trap的过程中，
切换的代码在kernel/trampoline.S中uservec和userret中。  
