# 《深入理解Linux内核（第三版）》笔记（十），第八章内存管理（一）

[内存管理](https://so.csdn.net/so/search?q=%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86&spm=1001.2101.3001.7020)（一）主要分析的是对内存页框的管理。  
看这章得到了一个感想，内存管理不止是 [mmu](https://so.csdn.net/so/search?q=mmu&spm=1001.2101.3001.7020)、页表这些内容。  
还有页框应该怎么分配，存储空间应该怎么分配等问题需要内核来解决。

> 为啥总是把部分代码复制到文档里再为代码写注释？直接在源代码上加注释不好吗？  
> 不一样的。  
> 最直观的，回头在源代码中都不一定能找到你写的注释。  
> 其次，源代码的注释是按照代码走的，没有章节性。  
> 然后，不会破坏代码的结构，并且可以不单独维护一套自己的代码（只要记好原始参考代码的版本就可以了）。

## 数据结构

    // 页结构体 -- 页描述符
    // include/linux/mm.h/line: 223
    /*
     * Each physical page in the system has a struct page associated with
     * it to keep track of whatever it is we are using the page for at the
     * moment. Note that we have no way to track which tasks are using
     * a page.
     */
    struct page {
    	...
    }
    // 每个物理页都一个这样的结构体，那么对于一个 4GiB 的内存 0x1_0000_0000，需要有 0x10_0000 个结构体
    // 0x10_0000 正好等于 1MiB；如果结构体为 32Byte 的话，就需要 32MiB 的空间
    // 32Byte/4KiB = 1/128
    // 页表结构体的 RAM 占用率是固定的 1/128
    // 存放这些结构体的全局变量的数组是：
    // mm/memory.c/line: 65
    struct page *mem_map;
    // 这里引发了一个思考，kernel 的全局变量的存在形式是什么样的？应该是常驻静态内核内存吧
    // 是不是只有 kernel 才能访问这些全局变量？
    // 当 kernel 要访问这些变量的时候，是不是只需要拿过名字来就能用


​    
​    // include/linux/page-flags.h/line: 54
​    #define PG_locked	 	 0	/* Page is locked. Don't touch. */


​    
​    
​    // include/asm-i386/page.h/line: 142
​    #define virt_to_page(kaddr)	pfn_to_page(__pa(kaddr) >> PAGE_SHIFT)
​    // include/asm-i386/page.h/line: 138
​    #define pfn_to_page(pfn)	(mem_map + (pfn))
​    // 
​    // 这里的 kaddr 是线性地址，内核的线性地址。
​    // 内核本身的线性地址是大于 0xC000_0000 的
​    // 内核加载到内存的低地址处，并且不再移动了
​    // 内核的线性地址基于物理地址偏移了 0xC000_0000，所以减 0xC000_0000 后就是物理地址


   
内存被分成了若干个 pg\_data\_t  
划分依据是内存的物理性质等导致的 NUMA（非一致内存访问）；比如 SRAM 和 DDR  
pg\_data\_t 是以链表的形式组织的；如果不存在 NUMA 的话，链表中就一个节点  
这两个应该是内核的数据。

    // include/linux/mmzone.h/line: 250
    typedef struct pglist_data {
    	struct zone node_zones[MAX_NR_ZONES];	// pg_data_t 又可以分成若干个 zone
    	...
    } pg_data_t;
    
    // mm/page_alloc.c/line: 43
    struct pglist_data *pgdat_list;
    // 这只是一个指针，不知道结构体的实体内存是在哪里在什么时候申请的
    
    // include/linux/mmzone.h/line: 110
    // 描述的是物理内存的情况
    struct zone {
    	...
    	struct pglist_data	*zone_pgdat;
    	...
    	struct pglist_data *pgdat_next;
    	...
    }
    
    // include/linux/mmzone.h/line: 233
    struct zonelist {
    	struct zone *zones[MAX_NUMNODES * MAX_NR_ZONES + 1]; // NULL delimited
    };


关于 ZONE\_HIGHMEM 有个疑问没搞懂，为什么内核不能直接访问？  
“内核不能直接访问大于 896M 以上的 RAM” 是多么可怕的一件事情？  
为啥还需要做一系列复杂的事情才能访问？难道 2.6 版本的时候，电脑的内存还是这么的小吗？  
自己想到的解释是：

1.  内核只能访问线性地址0xC000\_0000以后的空间，大小只有一个G
2.  内核实际是放到物理内存第一个GB的位置
3.  1个G的线性空间就映射到这段物理内存的位置上了

猜测，内核不能直接访问 896M 以上的 RAM，但是普通进程应该是可以的；这个后面考证吧。  
 

    // mm/page_alloc.c/line: 68
    int min_free_kbytes = 1024;
    // 相关操作函数都在同一个文件中
    // 1024 是默认值，后续会重新计算初始化，并且可以在运行中修改


## zoned page frame allocator

分区页框分配器  
按照《第三版》的章节进行组织的。  
先分析了接口，然后描述数据存放，然后是伙伴算法，然后是管理区的分配器。

### 接口

看起来，申请和释放页框，就是下面这几个函数，数量不算多。  
《第三版》用紧跟的后续章节分析这些接口的实现。

    // include/linux/gfp.h/line: 109
    #define alloc_pages(gfp_mask, order) \
    	...
    ...
    // 有若干个函数或者宏在 gfp.h 文件里
    // 并且操作的标志位等宏定义也在这个文件中


核心操作实现在 page\_alloc.c 中。

    // mm/page_alloc.c/line: 695
    /*
     * This is the 'heart' of the zoned buddy allocator.
     */
    struct page * fastcall
    __alloc_pages(unsigned int gfp_mask, unsigned int order,
    		struct zonelist *zonelist)
    // 看起来这个函数很重要；并且这个函数确实是很长
    // 后文会有详细分析
    
    // mm/page_alloc.c/line: 863
    fastcall unsigned long __get_free_pages(unsigned int gfp_mask, unsigned int order)


### 高端内存页框的内核映射

高端内存页框的内核映射是个比较复杂的事情，其实就是上面分析的内核访问 ZONE\_HIGHMEM 区的问题。  
内核的线性空间只有 896M，是 0xC000\_0000 之后的 1G 减去最高位的 128M  
内核要用这减去的 128M 作为线性空间，来访问高端内存页框。当用到时，需要不停的倒腾。  
BTW，如果想访问大于4G的物理空间，需要芯片的硬件支持，需要芯片提供 32bit 以上的地址线。

永久映射比较复杂，临时映射简单些。  
感觉已经是一个比较落后的技术了，未来常用的都是64位机。暂时不详细研究了。  
相关的操作主要在下面代码段提到的三个函数中。

    // include/asm-i386/highmem.h/line: 43
    // 开启扩展页表以后，需要64bit的寻址了，页表能放的最大表项数会变小
    #ifdef CONFIG_X86_PAE
    #define LAST_PKMAP 512
    #else
    #define LAST_PKMAP 1024
    #endif
    
    // mm/highmem.c/line: 58
    pte_t * pkmap_page_table;
    // 索引一个页表，页表中的数量和下面的这个数组数量一致，等于 LAST_PKMAP
    
    // mm/highmem.c/line: 54
    static int pkmap_count[LAST_PKMAP];
    // 通过计数，记录表征每个页表的使用状态
    
    // arch/i386/mm/highmem.c/line: 3
    void *kmap(struct page *page)


### 伙伴系统算法

是不是听说”伙伴算法“已经很久了？现在仔细研究下。

相关数据：

    // include/linux/mmzone.h/line: 23
    struct free_area {
    	// 不同大小的区块的链表
    	// 注意到这是个实体而不是指针，就是说它会作为链表的锚点
    	// 极限情况，只有一个元素的时候，这个节点的头和尾都指向同一个元素
    	// 所以，所谓为空，就是此处这个节点的头尾都是它自己
    	// 忘记这是不是 list_head 的通用做法了
    	struct list_head	free_list;
    	unsigned long		nr_free;
    };
    
    struct zone {
    	...
    	// include/linux/mmzone.h/line: 139
    	// 数组里有11个元素
    	// 注意到，这里同样是结构体实体，而不是指针
    	struct free_area	free_area[MAX_ORDER];
    	...
    	// include/linux/mmzone.h/line: 201
    	// 整个 mem_map 被三个 zone 瓜分，且不重不漏
    	// 每个 zone 瓜分的是哪一部分，用 zone_mem_map 记录
    	struct page		*zone_mem_map;
    	...
    }


    struct page {
    	...
    	// include/linux/mm.h/line: 231
    	// 存放自己属于多大的块；只有块的第一个页才会用这个字段
    	unsigned long private;
    	...
    	// include/linux/mm.h/line: 246
    	// 这个挺重要的，但是具体怎么使用，可以结合下文的代码进行理解
    	struct list_head lru;		/* Pageout list, eg. active_list
    	...
    }


   
分配操作：

    // mm/page_alloc.c/line: 431
    static struct page *__rmqueue(struct zone *zone, unsigned int order)
    {
    	...
    	// lru 是 page 在 free_list 里的代表
    	// free_list 的头是不变的，每次把头的下一个节点拿出来
    	page = list_entry(area->free_list.next, struct page, lru);	// line: 442
    	list_del(&page->lru);
    	rmv_page_order(page);	// 只有空闲的 page 的 private 字段才可能有意义
    	// 原则上 order < current_order
    	return expand(zone, page, order, current_order, area);
    	...
    }
    
    // mm/page_alloc.c/line: 367
    static inline struct page *
    expand(struct zone *zone, struct page *page,
     	int low, int high, struct free_area *area)
    {
    	unsigned long size = 1 << high;		// 是 1 左移了 n 位；不是 n 左移了 1 位
    	while (high > low) {
    		// 数组元素减减，就是变成了前一个元素
    		// 如果本来就是最前的元素了，是不会进这个 whilld 的
    		area--;
    		high--;
    		size >>= 1;
    		...
    		// 拿到的 page 是内存块的第一个 page
    		// size 变量发挥了极大的作用
    		// &page[size] 是先括号后取址，取的是高位 、后半段的那一半
    		// 这时候，area 已经是降级的了 free_area 数组中的元素了；所以这一句就完成了插入
    		list_add(&page[size].lru, &area->free_list);
    		area->nr_free++;5
    		// 确保的是块链表元素的第一个 page 的 private 有 order 值
    		set_page_order(&page[size], high);
    	}
    	return page;
    }


   
释放操作：

    // mm/page_alloc.c/line: 236
    static inline void __free_pages_bulk (struct page *page, struct page *base,
    		struct zone *zone, unsigned int order)
    {
    	...
    	// 终止条件分析下
    	// 要知道，最大的块的页框数是 power(2, MAX_ORDER-1)
    	// 并且 free_area 数组有 MAX_ORDER 个元素，数组的最大下标也是 MAX_ORDER-1
    	// “最后一级、需要合并的块”的大小是 power(2, MAX_ORDER-2)
    	while (order < MAX_ORDER-1) {
    		...
    		// 这一行有搞头啊
    		// 这里运用的异或的基础是：宏观上看相同数异或等于零，微观上看是处理 (1<<order) 的最高位
    		// 异或的另一个解释是，右边的数类似于一个 mask，右边的数相对于 mask 为 1 的位，会取反
    		// 这句话的作用，是为 page_idx 向上或者向下找它的 buddy
    		// 如果 page_idx 的第 order 位（按最低位是 bit0 算）是0，那么它的 buddy 就是相邻的高地址的块
    		// 如果是 1，那么它的 buddy 就是相邻的低地址的块
    		buddy_idx = (page_idx ^ (1 << order));
    		...
    		/* Move the buddy up one level. */
    		// 把找到的 buddy 从低一个 level 的 free 链表中拿出来
    		list_del(&buddy->lru);
    		...
    		// 把 page_idx 升一下级，第 order 位被清零了
    		// 要知道，page_idx 和 buddy_idx，总有一个的第 order 位会是零的
    		page_idx &= buddy_idx;
    		...
    	}
    	...
    }


### 每 CPU 页框高速缓存

再梳理下内存管理的层次：

-   基于物理内存的特性，内存被分成了若干个 pg\_data\_t
-   基于空间地址，每个 pg\_data\_t 被分为三个 zone；感觉在 arm 中，这个特性已经不明显了
-   每个 zone 结构体中，有一个每 CPU 变量结构体（也就是一个结构体数组，数组个数等于 CPU 数量）  
    那就是结构体 per\_cpu\_pageset
-   结构体 per\_cpu\_pageset 又包含两个数组，冷热 per\_cpu\_pages
-   冷热 per\_cpu\_pages 维护的核心的数据是一个 page 的链表

这个页框高速缓存是区别于物理硬件高速缓存的。  
这个页框高速缓存的目的，是一次性申请若干个页框（可以称之为“预申请”），然后当真正需要单个页框时，优先使用预申请的页框，这样能提高系统的效率。  
 

    struct zone {
    	...
    	// include/linux/mmzone.h/line: 124
    	// 这是结构体实体，且是每 CPU 变量
    	struct per_cpu_pageset	pageset[NR_CPUS];
    	...
    }
    
    // include/linux/mmzone.h/line: 53
    struct per_cpu_pageset {
    	struct per_cpu_pages pcp[2];	/* 0: hot.  1: cold */
    	...
    }
    
    // include/linux/mmzone.h/line: 45
    struct per_cpu_pages {
    	int count;		/* number of pages in the list */
    	int low;		/* low watermark, refill needed */
    	int high;		/* high watermark, emptying needed */
    	int batch;		/* chunk size for buddy add/remove */
    	struct list_head list;	/* the list of pages */
    };


    // mm/page_alloc.c/line: 617
    static struct page *
    buffered_rmqueue(struct zone *zone, int order, int gfp_flags)
    {
    	...
    	int cold = !!(gfp_flags & __GFP_COLD);	// 这句话可以优雅地实现，不等于 0 的 n 变为 1
    
    	// 当只需要申请一个，也就是 power(2,0) 个，页框时，才会进这个 if
    	// 所以，当需要一个页框时，总是会优先使用页框高速缓存
    	if (order == 0) {
    		...
    		if (pcp->count) {
    			page = list_entry(pcp->list.next, struct page, lru);
    			list_del(&page->lru);	// 一个页框在某一刻只能隶属于某一个链表
    			pcp->count--;	// if 语句保证了减 1 就可以
    		}
    		...
    	}
    
    	// 不管是否使用了高速缓存，都要有这个判断，因为：
    	// 有可能申请的不是单个页框，没进上面的 if
    	// 有可能是上面的 if 申请单个页框没有成功	
    	if (page == NULL) {
    		...
    	}
    
    	// 经过上面的两个 if，这个是操作主体；申请的页框需要做这个 if 里的操作
    	if (page != NULL) {
    		...
    	}
    	return page;
    }


    // mm/page_alloc.c/line: 569
    static void fastcall free_hot_cold_page(struct page *page, int cold)
    
    // mm/page_alloc.c/line: 592
    void fastcall free_hot_page(struct page *page)
    
    // mm/page_alloc.c/line: 597
    void fastcall free_cold_page(struct page *page)


### 管理区分配器

    // mm/page_alloc.c/line: 695
    struct page * fastcall
    __alloc_pages(unsigned int gfp_mask, unsigned int order, struct zonelist *zonelist)
    {
    	...
    restart:
    	/* Go through the zonelist once, looking for a zone with enough free */
    	for (i = 0; (z = zones[i]) != NULL; i++) {
    		// 先用触发页框回收的阈值进行过滤
    		if (!zone_watermark_ok(z, order, z->pages_low, classzone_idx, 0, 0))
    			continue;
    		...
    	}
    	...		// 没过滤到，进行了页框回收
    
    	for (i = 0; (z = zones[i]) != NULL; i++) {
    		// 再用保留页框的阈值进行过滤
    		if (!zone_watermark_ok(z, order, z->pages_min, classzone_idx, 
    			can_try_harder, gfp_mask & __GFP_HIGH))
    			continue;
    		...
    	}
    	// 到这里了，就是真没有内存了
    	// 如果属于不可阻塞线程，就尝试使用保留内存，如果还是没有，就返回 NULL 退出
    	// 关于不可阻塞的属性，比如说中断线程，比如说一个线程已经尝试过页框回收了（标志位 PF_MEMALLOC）
    	...
    	// 如果属于可阻塞线程，就：
    	/* Atomic allocations - we can't balance anything */
    	if (!wait)
    		goto nopage;	// 但是配置成了不阻塞的属性
    
    rebalance:
    	...
    	/* We now go into synchronous reclaim */
    	p->flags |= PF_MEMALLOC;
    	reclaim_state.reclaimed_slab = 0;
    	// 这是把局部变量的地址赋给进程描述符的元素了啊，有点艺高人胆大的意思啊
    	// 就是说，退出这个函数（退栈）以后，描述符的对应元素就失效了
    	p->reclaim_state = &reclaim_state;
    	did_some_progress = try_to_free_pages(zones, gfp_mask, order);	// 线程会在这里被阻塞
    	// 再回来时，已经不知道外面发生了多少故事了
    	p->reclaim_state = NULL;
    	p->flags &= ~PF_MEMALLOC;
    
    	...
    	do_retry = 0;
    	if (...) {		// 一系列的条件判断，确定是否能赋 1
    		do_retry = 1;
    	}
    	if (do_retry) {
    		blk_congestion_wait(WRITE, HZ/50);	// 知道是休眠，但是不知道这个函数的细节
    		// 是不是，如果条件足够的话，这个进程会一直等下去？如此的随机性？
    		goto rebalance;
    	}
    	...
    }
    
    // mm/page_alloc.c/line: 664
    int zone_watermark_ok(struct zone *z, int order, unsigned long mark,
    		      int classzone_idx, int can_try_harder, int gfp_high)
    {
    	...
    	// 这个循环需要好好理解一下
    	// 这个循环的总体的目的是，
    	for (o = 0; o < order; o++) {
    		/* At the next order, this order's pages become unavailable */
    		// 剩余的块数左移一个 o 是什么意思？
    		// 是要清理一下剩余的页框数，规则是比目标 order 小的块所占的页框数，都不算
    		// 当循环到 order 的大小的时候，求得的 zone 中剩余的页框的数目中，包含的都是\
    		   至少不小于 order 对应块的组合的页框数
    		// 对这些页框数做比较
    		free_pages -= z->free_area[o].nr_free << o;
    		// 这句话的意思，大概是，当请求的内存块比较大时，忽略 min 的作用
    		// 目前还没有明白这个规则的目的
    		min >>= 1;	
    		...
    	}
    }
    // 返回 1 的话，就算是成了


​    

## 突然想到的

一个进程的进程描述符，不会常驻内核内存；而是进程信息常驻内存；可以通过进程信息找到进程描述符。  
不考虑内存交换的情况下。  
进程描述符实际上是被内核管理的，进程本身原则上感知不到进程描述符。  
每个进程在内核中的“代表”所占用的空间是 8K（内核有自己的页目录，不够了会自己申请，并且会常驻），所以 1M 内存只够 128 个进程的。  
猜测，进程描述符占用的那个页框，算在内核身上，而不是相对应的进程身上。  
并且进程描述符没必要独占一个页框，应该就是按普通的数据处理的，可以好多进程描述符“挤”在同一个页框里。

一个进程：

-   至少要自己占一个“页目录转换表”的页框吧
-   至少要占两个“页表转换表”的页框吧
    -   一个页框映射最高的线性地址，存储环境变量、作为栈顶
    -   一个页框映射最低的线性地址，存储正文和各个段
-   至少要占两个实际用到的页框吧，上文说的，一个存储最高线性地址的数据，一个存储最低线性地址的数据。

再加上常驻内核的两个页框。  
所以猜测，一个进程的创建，至少要用掉7个页框。



## 参考

[《深入理解Linux内核（第三版）》笔记（十），第八章内存管理（一）_深入理解linux内核第三版读书笔记-CSDN博客](https://blog.csdn.net/weixin_42346852/article/details/126887706?spm=1001.2014.3001.5506)