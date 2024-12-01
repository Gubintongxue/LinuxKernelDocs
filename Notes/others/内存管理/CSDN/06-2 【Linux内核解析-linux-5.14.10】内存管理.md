# 【Linux内核解析-linux-5.14.10】内存管理

内核中的[内存管理](https://so.csdn.net/so/search?q=%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86&spm=1001.2101.3001.7020)主要包括以下内容：

1.  内存分配和释放：内核需要负责为进程分配和释放内存，以确保进程能够正常运行。内存分配和释放的方式有多种，例如伙伴系统、slab分配器等。

-   [伙伴系统](https://so.csdn.net/so/search?q=%E4%BC%99%E4%BC%B4%E7%B3%BB%E7%BB%9F&spm=1001.2101.3001.7020)：伙伴系统是一种动态内存分配算法，它将可用内存划分为一系列大小相等的块，每个块大小为2的幂次方。当需要分配内存时，伙伴系统会找到最小的大于等于所需大小的2的幂次方的块，然后将该块一分为二，将其中一半分配给进程使用，另一半标记为可用块。当需要释放内存时，伙伴系统会将该块和其“伙伴”（即大小和自己相等的另一块）合并，形成一个更大的块。
    
-   slab分配器：slab分配器是一种静态内存分配算法，它将内存划分为一系列大小相等的区域，称为slab。每个slab中包含一定数量的相同大小的对象，例如文件描述符、inode等。当需要分配对象时，slab分配器会从相应大小的slab中分配一个对象，当对象不再使用时，它会被返回到相应的slab中。
    

2.  虚拟内存管理：内核需要对虚拟地址空间进行管理，以支持进程的内存隔离和虚拟内存映射。虚拟内存管理的主要功能包括：

-   地址空间的创建和销毁：内核需要为每个进程创建一个独立的虚拟地址空间，以实现内存隔离和保护。
    
-   虚拟内存映射：内核需要将进程的虚拟地址映射到物理内存中，以实现虚拟内存的功能。虚拟内存映射可以使用页表来实现，即将虚拟地址映射到物理地址的过程。
    
-   页面置换：当物理内存不足时，内核需要将一部分虚拟内存页置换到磁盘上，以释放物理内存。页面置换算法有多种，例如最近最少使用（LRU）、最不经常使用（LFU）等。
    

3.  内存保护和访问控制：内核需要保护进程的内存，防止进程越界访问或者非法访问其他进程的内存。内存保护和访问控制可以使用硬件机制和软件机制来实现，例如MMU和权限位等。
    
4.  内存共享和通信：内核需要支持进程间的内存共享和通信，以实现进程间的数据交换和协作。内存共享和通信可以使用共享内存、消息队列、管道等机制来实现。
    

## 代码初始化解释

    /*
     * Set up kernel memory allocators
     */
    static void __init mm_init(void)
    {
    	/*
    	 * page_ext requires contiguous pages,
    	 * bigger than MAX_ORDER unless SPARSEMEM.
    	 */
    	page_ext_init_flatmem();
    	init_mem_debugging_and_hardening();
    	kfence_alloc_pool();
    	report_meminit();
    	stack_depot_init();
    	mem_init();
    	mem_init_print_info();
    	/* page_owner must be initialized after buddy is ready */
    	page_ext_init_flatmem_late();
    	kmem_cache_init();
    	kmemleak_init();
    	pgtable_init();
    	debug_objects_mem_init();
    	vmalloc_init();
    	/* Should be run before the first non-init thread is created */
    	init_espfix_bsp();
    	/* Should be run after espfix64 is set up. */
    	pti_init();
    }


这个函数是在内核初始化时调用的，用于设置内核内存分配器。函数内部依次调用了以下函数：

-   `page_ext_init_flatmem()`：初始化 `page_ext`，该结构体用于跟踪页面的一些扩展属性。该函数要求页面是连续的，并且大于 `MAX_ORDER`（如果没有使用 `SPARSEMEM`）。
-   `init_mem_debugging_and_hardening()`：初始化内存调试和加固功能。
-   `kfence_alloc_pool()`：为 KFence（内存安全检查工具）分配内存池。
-   `report_meminit()`：在控制台上报告内存初始化信息。
-   `stack_depot_init()`：初始化内核堆栈存储库。
-   `mem_init()`：初始化内存管理子系统。
-   `mem_init_print_info()`：在控制台上报告内存初始化信息。
-   `page_ext_init_flatmem_late()`：在伙伴系统准备好之后初始化 `page_ext`。
-   `kmem_cache_init()`：初始化内核内存缓存。
-   `kmemleak_init()`：初始化内存泄漏检测工具。
-   `pgtable_init()`：初始化页表。
-   `debug_objects_mem_init()`：初始化内核调试对象。
-   `vmalloc_init()`：初始化虚拟内存分配器。
-   `init_espfix_bsp()`：初始化 `espfix`，该功能用于修复 32 位系统上的 ESP 寄存器。
-   `pti_init()`：初始化页表隔离功能。

这些函数的调用顺序和具体实现细节可能会因不同内核版本而有所不同。

## 参考

[【Linux内核解析-linux-5.14.10】内存管理_linux 5.14-CSDN博客](https://blog.csdn.net/qq_21688871/article/details/130187843?spm=1001.2014.3001.5506)