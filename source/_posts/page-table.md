---
title: page-table
date: 2021-03-15 18:58:23
tags:
---
[Page Table Lab](https://pdos.csail.mit.edu/6.S081/2020/labs/pgtbl.html)一开始真的是一脸懵逼，做完实验再加上[Q&A labs](https://www.youtube.com/watch?v=_WWjNIJAfVg)，现在对page table有了些理解，值得记录一下。  

**为什么process需要一个自己的kernel page table？**  
Process并不一定需要一个kernel page table，这个lab提供的初始代码中process就是没有kernel page table的，也是可以work的。现在问题就转化为了  
**Process拥有一个自己的kernel page table有什么好处**  
记kernel自己的page table为gkpagetable，process的kernel page table 为ukpagetable，process
的page table为upagetable。
假设你现在用write把buffer中的东西写道文件，write是一个系统调用，buffer是在user address space
的一个地址，cpu用任何一地址都要先经过gkpagetable转换，所以现在kernel不能直接用一个
来自user address space的地址。那怎么办呢？在xv6中有两种方式： 
一是把来自user address space的地址翻译成物理地址pa，pa在gkpagetable中是可用的，因为在xv6中gkpagetable到phsical address space是一个direct mapping。 
```c
// Copy from user to kernel.
// Copy len bytes to dst from virtual address srcva in a given page table.
// Return 0 on success, -1 on error.
int
copyin(pagetable_t pagetable, char *dst, uint64 srcva, uint64 len)
{
  uint64 n, va0, pa0;

  while(len > 0){
    va0 = PGROUNDDOWN(srcva);
    pa0 = walkaddr(pagetable, va0);
    if(pa0 == 0)
      return -1;
    n = PGSIZE - (srcva - va0);
    if(n > len)
      n = len;
    memmove(dst, (void *)(pa0 + (srcva - va0)), n);

    len -= n;
    dst += n;
    srcva = va0 + PGSIZE;
  }
  return 0;
}
```
第二种方式就是process单独维护一个kernal page table，即ukpagetable。ukpagetable是
gkpagetable一样是一个direct mapping，但除此之外，将upagetable的mapping复制到ukpagetable
中去，这样每次切换进程时，让wstap指向该process的ukpagetable。这样，kernel也可以直接使用
user address space的地址了。  
```c
// Copy from user to kernel.
// Copy len bytes to dst from virtual address srcva in a given page table.
// Return 0 on success, -1 on error.
int
copyin_new(pagetable_t pagetable, char *dst, uint64 srcva, uint64 len)
{
  struct proc *p = myproc();

  if (srcva >= p->sz || srcva+len >= p->sz || srcva+len < srcva)
    return -1;
  memmove((void *) dst, (void *)srcva, len);
  stats.ncopyin++;   // XXX lock
  return 0;
}
```
现在新的copyin方法就不需要将user space的地址srcva翻译为物理地址了。但这种方法有一个前提是
将upagetable的mapping复制到ukpagetable时，不能覆盖ukpagetable原有的mapping。庆幸的是
upagetable的va是从0开始的，而ukpagetable从PLIC即0x0C000000才开始有映射，所以只要process的
virtual address超过0x0c000000就没问题。