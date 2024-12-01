# [Linux源码分析]内存管理

### 一、 内核空间

**1\. 页page（是内核空间管理基本单位）源代码分析如下：/include/linux/mm\_types.h  
内存管理单元（MMU,把虚拟地址转换为物理地址的硬件设备）通常是以页为单位处理。**  
内核用[struct](https://so.csdn.net/so/search?q=struct&spm=1001.2101.3001.7020) page结构体表示每个物理页，struct page占用40个字节。  
2\. 区（zone，内核把页划分在不同的区）  
共记3个区：  
ZONE\_DMA(DMA使用的页，物理内存<16MB)  
ZONE\_NORMAL(可以正常寻址的页，物理内存16—896MB)  
ZONE\_HIGHMEM（动态映射的页，物理内存>896MB）  
执行DMA操作必须从ZOME\_DMA区分配，一般内存，既可以从ZOME\_DMA，也可以从ZONE\_NORMAL分配，但不能从两个区分配。  
**3\. 页的分配与释放**  
所有页为单位进行连续的物理内存分配，也称为低级页分配，alloc\_page、alloc\_pages、  
\_\_get\_free\_pages、\_\_get\_free\_pages、\_\_get\_zeroed\_page.  
释放函数：\_\_free\_pages、free\_pages、free\_page.  
**4\. 字节分配与释放(kmalloc/vmalloc，分配都是以字节为单位)**  
Void \*kmalloc(size\_t size,gpf\_t flags);  
Kmalloc 函数返回指向内存块的指针，内存块大小至少size，所分配内存在物理内存中连续且保持原有的数据（不清零）。  
Flags取值说明（常用部分如下）：

> > **GFP\_USER-用于用户空间分配内存，可能休眠  
> > GFP\_KERNEL-用于内核空间分配内存，可能休眠  
> > GFP\_ATOMATIC-用于原子性的分配内存，不会休眠，典型原子性场景，中断处理程序、软中断。**

Kamlloc函数用于内存分配最终会调用\_get\_free\_pages进行实际分配，前缀都是GFP\_开头。 Kamalloc分配最多只能分配32个page大小的内存，每一个page=4k，也就是分配128k大小，其中16个字节用来记录页的描述结构。所分配的是常驻内存，不会被交换到文件中，最小分配单位是32或64个字节。

Vmalloc函数：返回是一个指向内存块的指针，内存块大小至少是size，它所分配的内存是逻辑上了连续的，该函数也有flags，默认它是可以休眠的。 Kmalloc:所在区域内核空间，物理地址连续，最大值为128k-16k，释放函数free，性能最佳 Vmalloc: 所在区域内核空间，虚拟地址连续，更大，释放函数vfree，更容易分配大内存。 Malloc: 所在区域用户空间，虚拟地址连续，更大，释放函数free。

### 二、 slab分配器的功能

对于频繁地分配和释放的数据结构，会缓存它；  
频繁分配和回收，比如导致内存碎片、为了避免，空间链表的缓存会连续的存放，已释放的数据结构又会放回空闲链表，不会导致碎片。  
记部分缓存专属单个处理器，分配和释放操作可以不加SMP锁。

### 三、 slab分配器层的设计

**slab层把不同的对象划分所谓的高速缓存组，其中每个缓存组都存放不类型的对象。每一种对象类型对应一个高速缓存。比如：一个高速缓存用于进程描述符（task\_struct结构的一个空闲链表），另外一个高速缓存存放索引节点对象（struct inode）.  
Kmalloc()接口建立在slab层上的，使用一组通用高速缓存。  
这些高速缓存又被划分为slab，slab由一个或多个物理上连续的页page组成，一般情况下，slab也就仅仅由页组成。每个高速缓存可以由多个slab组成。**

    Struct slab{
    Struct list_head list;//满，部分满，或者空链表
    Unsigned long colouroff; /*slab着色的偏移量*/
    Void * s_mem; /* 在slab中的第一个对象*/
    Unsigned int inuse;//slab中已分配的对象数
    Kmem bufctl free;//第一个空闲对象
    Unsigned short nodeid;
    };


### 四、 slab的接口分配器（参考源代码fork.c）

每当进程调用fork时，一定会创建一个新的进程描符，这是在dup\_task\_struct()进程中完成，而该函数会被do\_fork()调用.执行完毕后，如果没有子进程在等待的话，它的进程描述符就会被释放。返回给task\_struct\_cachep slab高速缓存。  
Slab层负责在内存紧缺情况下所有底层的对齐、着色、分配、释放、回收等。如果我们要频繁创建很多相同类型的对象，就因该考虑使用slab高速缓存。