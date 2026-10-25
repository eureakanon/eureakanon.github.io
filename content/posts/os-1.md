# 操作系统课程中的经典分页内存模型

## 连续的内存分配模型:以30天自制操作系统为例

连续内存分配的算法非常直接,可以将所有能分配的内存以及是否空闲的信息作为一个数组,然后遇到大小为size的分配请求,就遍历寻找,找到第一块比请求大的块,然后进行分配
具体代码如下
```c
struct FreeInfo
{
    unsigned int addr,size;
};
struct MemMan
{
    int frees,maxfrees,lostsize,losts;
    struct FreeInfo free[MEMMAN_FREES];
};
...
unsigned int memman_alloc(struct MemMan* man,unsigned int size)
{
    for(unsigned int i=0;i<man->frees;i++)
    {
        if(man->free[i].size>=size)
        {
            unsigned int a=man->free[i].addr;
            man->free[i].addr+=size;
            man->free[i].size-=size;
            if(man->free[i].size<=0)
            {
                man->frees--;
                for(unsigned int j=i;j<man->frees;j++)
                {
                    man->free[j]=man->free[j+1];
                }/*如果free[i].size==0,就往前移动*/
            }
            return a;
        }
    }
    return 0;
}
```
当然内存回收的算法也非常的狂野,遍历找到最后一个内存地址小于请求回收的地址的内存块,然后插入到这个内存块之后,并且如果可以与前面的地址,或者后面的地址进行合并,那就合并,防止内存的碎块
```c
int memman_free(struct MemMan* man,unsigned int addr,unsigned int size)
{
    int i,j;

    for(i=0;i<man->frees;i++)
    {
        if(man->free[i].addr>addr)
        {
            break;
        }
    }

    if(i>0)
    {
        if(man->free[i-1].addr+man->free[i-1].size==addr)
        {
            man->free[i-1].size+=size;
            if(i<man->frees)
            {
                if(addr+size==man->free[i].addr)
                {
                    man->free[i-1].size+=man->free[i].size;
                    man->frees--;
                    for(int k=i;k<man->frees;k++)
                    {
                        man->free[k]=man->free[k+1];
                    }
                }
            }
            return 0;
        }
    }
    // 不能与前面的可用空间归纳到一起
    if (i < man->frees) {
    // 后面还有
        if (addr + size == man->free[i].addr) {
            man->free[i].addr = addr;
            man->free[i].size += size;
            return 0; // 成功完成
        }
    }
    if(man->frees<MEMMAN_FREES)
    {
        for(j=man->frees;j>i;j--)
        {
            man->free[j]=man->free[j-1];
        }
        man->frees++;
        if (man->maxfrees < man->frees) {
            man->maxfrees = man->frees; // 更新最大值
        }
        man->free[i].addr=addr;
        man->free[i].size=size;
        return 0;
    }
    man->losts++;
    man->lostsize += size;
    return -1;
}
```

但是随着进程不断创建和终止，物理内存中会留下许多大小不一的“空洞”。总空闲内存可能很充足，但可能没有任何一个单独的空洞足以容纳一个新进程。在长期运行的系统里，内存会变得像“瑞士奶酪”一样千疮百孔。

并且进程无法申请顺利的申请更多的内存资源,如果这个进程希望为自己申请堆栈,它可能发现周边的内存早已被其他程序占有,导致必须迁移到另一片连续内存,造成效率的低下。

还有一个问题,假如进程申请了比较大的内存,导致系统进程的内存资源被挤占,安全性如何保证。

于是我们在想,如何避免这样的缺点,于是我们首先设计:1.必须要能按照进程的需要分配内存的资源,所以两个进程所申请的资源必须隔离,2.权限必须分开,用户程序不可以申请系统程序的内存,最好用户程序工作在一个"虚拟的"环境中
3.最好资源的大小统一,保证不会有碎片和浪费,在这些需求的推动下,分页机制登场。

分页机制让每一个进程所分配的连续地址被映射到实际上是隔离的物理页表,给每一个物理页赋予了权限,如用户权限和内核权限,并且让用户程序运行在虚拟地址成为可能

## 分页机制的逻辑抽象

假如我们把所有的物理页都集中在一个数组当中,一般来说一个物理页是4KB,然后4GB的寻址空间让我们必须有1M个物理页的表项,于是这个表项的大小是1M*4字节的地址=4MB大小,听起来不错，但是如果我们有一百个进程呢?我们一个个分配就是400MB的大小,在早期这是不可以接受的(表项不可以共用,因为两个进程的内存空间必须隔离)

于是我们把数组折叠起来变成了一棵树,只有两层

第一层叫PDE,PDE的基址在CR3中取到,PDE以一个1024项的数组存在,每一个又都存储着一个1024项的数组,但是以指针的形式存在,指示这个数组的基址

第二层叫PTE,PTE就是这个1024项的数组,存储着物理页的基址

简单而言就是`PDE=CR3[1024]`,`PTE=PDE[1024]`

于是,一个具体的32位的地址,可以表示成为10+10+12,高10位为PDI,用于指示在PDE的哪一项,低10位为PTI,用于指示在PTE的哪一项，12为页内偏移

举例:0x00403ABC,CR3=0x00100000

那么先取前10位:0x00000000001=1,于是查询PDE[1]=CR3+4得到PTE的基址,0x00201007,(这里我们只需要0x00201000,为什么请看下一节)

取中间的10位:0x00000000003=3,于是查询PTE[3]=CR3+12得到物理页的基址,0x00345007，(同样我们只需要0x00345000)

加入页内偏移得到0x00345ABC,这就是具体的物理地址

于是用户进程可以工作在这样的虚拟地址中,通过这样的翻译得到隔离在不同的页的物理地址,互不打扰

## 分页机制的权限设置

为什么32位的数字我们只取了前20位,那是因为最低12位是权限的设置

速记表:

对于PDE:
标志位 (Flags)：用于控制权限和状态，常见的有：

P (Present)：表示该页表是否存在于物理内存中。

R/W (Read/Write)：表示对该 PDE 所映射的所有页面是只读还是可读写。

U/S (User/Supervisor)：控制访问权限是用户模式还是内核（超级用户）模式,这一个位置让用户程序无法劫持系统程序的运行空间

A (Accessed)：表示该 PDE 是否被访问过。

PS (Page Size)：在支持大页的系统中，该位若置 1，则表示该 PDE 直接映射一个 4MB 的大页，而不再指向一个页表。

对于PTE,除了PDE,还有一些独有的位置:

D (Dirty)：表示该物理页是否被修改（写过）。

G (Global)：表示该页面是否为全局页面，在切换地址空间（如进程切换）时，其在 TLB 中的缓存不会被刷新。

好的,原则上我们理解了二级页表的机制之后,就可以用C语言写一个为进程分配内存的程序,具体实现等待后续文章的推送。


