# 《深入理解Linux内核（第三版）》笔记（十一），第八章内存管理（二）

#### 目录

-   [各种数据结构](https://blog.csdn.net/weixin_42346852/article/details/127225698?spm=1001.2014.3001.5506#_4)
-   -   [高速缓存描述符](https://blog.csdn.net/weixin_42346852/article/details/127225698?spm=1001.2014.3001.5506#_5)
    -   [slab 描述符](https://blog.csdn.net/weixin_42346852/article/details/127225698?spm=1001.2014.3001.5506#slab__110)
    -   [对象描述符](https://blog.csdn.net/weixin_42346852/article/details/127225698?spm=1001.2014.3001.5506#_122)
-   [kmem\_cache 的初始化](https://blog.csdn.net/weixin_42346852/article/details/127225698?spm=1001.2014.3001.5506#kmem_cache__131)
-   [为 kmem\_cache 增加一个 slab](https://blog.csdn.net/weixin_42346852/article/details/127225698?spm=1001.2014.3001.5506#_kmem_cache__slab_184)
-   -   [入口的函数调用序列](https://blog.csdn.net/weixin_42346852/article/details/127225698?spm=1001.2014.3001.5506#_186)
    -   [cache\_grow()](https://blog.csdn.net/weixin_42346852/article/details/127225698?spm=1001.2014.3001.5506#cache_grow_214)
    -   [array\_cache 管理](https://blog.csdn.net/weixin_42346852/article/details/127225698?spm=1001.2014.3001.5506#array_cache__316)
    -   [kmalloc](https://blog.csdn.net/weixin_42346852/article/details/127225698?spm=1001.2014.3001.5506#kmalloc_320)
-   [内存池](https://blog.csdn.net/weixin_42346852/article/details/127225698?spm=1001.2014.3001.5506#_365)

这个和上篇是有区别的，上篇主要是关心页框的处理；这篇主要是对内存（即起始地址和长度）的管理。  
 

## 各种数据结构

### [高速缓存](https://so.csdn.net/so/search?q=%E9%AB%98%E9%80%9F%E7%BC%93%E5%AD%98&spm=1001.2101.3001.7020)描述符

    // include/linux/slab.h/line: 12
    typedef struct kmem_cache_s kmem_cache_t;
    
    // mm/slab.c/line: 302
    struct kmem_cache_s {
    	// 这是个每 CPU 变量
    	// 这是个指针，它的实体需要一些操作来进行实例化
    	struct array_cache	*array[NR_CPUS];
    	...
    	// 这里是结构体实体，而不是指针
    	struct kmem_list3	lists;
    	...
    	// 这个描述一个 slab 需要多少个页框；直接了当，而不是基于 slab 的大小对页框大小取余取整什么的
    	unsigned int		gfporder;
    	...
    	// 这个感觉更像是 obj 的对齐大小
    	unsigned int		colour_off;	/* colour offset */
    	...
    	// 这个不是整个 slab 的内存大小，一个 slab 的内存大小其实就是多少个页框
    	// 这个应该是 slab 描述符和对象描述符数组的大小
    	unsigned int		slab_size;
    	...
    	// 一切皆链表，一个 kmem_cache 被创建以后，会统一放到链表 cache_chain 中
    	struct list_head	next;
    	...
    }


    // mm/slab.c/line: 250
    struct array_cache {
    	...
    };
    // mm/slab.c/line: 261
    struct arraycache_init {
    	struct array_cache cache;
    	void * entries[BOOT_CPUCACHE_ENTRIES];
    };
    // 这个结构体的实例化主要是在 kmem_cache_init() 函数中
    // 在 kmalloc 环境具备以后进行实例化
    
    // 感觉这个功能不是很明朗，暂时不研究了


    // mm/slab.c/line: 273
    struct kmem_list3 {
    	// 这三个链表也是实体，会是双向链表的表头
    	struct list_head	slabs_partial;	/* partial list first, better asm code */
    	struct list_head	slabs_full;
    	struct list_head	slabs_free;
    	...
    }
    
    // mm/slab.c/line: 550
    static struct list_head cache_chain;


分为普通高速缓存和专用高速缓存。  
这里的普通，一点都不普通；实际上是 cache 的[管理者](https://so.csdn.net/so/search?q=%E7%AE%A1%E7%90%86%E8%80%85&spm=1001.2101.3001.7020)。  
专用，指的是用于特定的功能，比如管理进程描述符，管理[文件描述符](https://so.csdn.net/so/search?q=%E6%96%87%E4%BB%B6%E6%8F%8F%E8%BF%B0%E7%AC%A6&spm=1001.2101.3001.7020)。

普通高速缓存中最特殊的那个：

    // mm/slab.c/line: 535
    static kmem_cache_t cache_cache = {
    	// 这个是个实体，需要初始化；关键是前三个双向链表的初始化
    	// 知识基础，一个 list_head 的初始化，是把它本身的地址赋给它的头指针和尾指针
    	// 这是个静态变量，定义的时候空间就分配好了
    	// 所以，LIST3_INIT 需要做的工作就是把自己的对应的元素的地址赋给自己的内容
    	.lists		= LIST3_INIT(cache_cache.lists),
    	...
    	.objsize	= sizeof(kmem_cache_t),
    	...
    	.name		= "kmem_cache",
    	...
    };
    
    // mm/slab.c/line: 283
    #define LIST3_INIT(parent) \
    	{ \
    		.slabs_full	= LIST_HEAD_INIT(parent.slabs_full), \
    		.slabs_partial	= LIST_HEAD_INIT(parent.slabs_partial), \
    		.slabs_free	= LIST_HEAD_INIT(parent.slabs_free) \
    	}
    
    // 展开后：
    static kmem_cache_t cache_cache = {
    	.lists = {
    		.slabs_full = {&(cache_cache.lists.slabs_full), &(cache_cache.lists.slabs_full)},
    		.slabs_partial = {&(cache_cache.lists.slabs_partial), &(cache_cache.lists.slabs_partial)},
    		.slabs_free = {&(cache_cache.lists.slabs_free), &(cache_cache.lists.slabs_free)}
    	}
    	...
    }


回头分析下，它自己是否作为 object 放到了自己的描述符里面。

在 Ubuntu 系统中可以直接查看电脑的 kmem\_cache

    user@pc:~$sudo cat /proc/slabinfo | grep kmem_cache
    kmem_cache          3227   3312    448   36    4 : tunables    0    0    0 : slabdata     92     92      0


kmem\_cache 只要一个结构体大小的空间就好了，不用额外的再申请其他的空间。  
有了 kmem\_cache 以后，就可以往上面挂 slab 了。

### slab 描述符

    // slab 描述符
    // mm/slab.c/line: 207
    struct slab {
    	...
    	unsigned long		colouroff;	// 第一个对象的偏移，这个偏移是以字节为单位的
    	void			*s_mem;		/* including colour offset */	// 真实地址
    	...
    }


“slab 由一个或多个页框组成”这句话很关键啊，这说明什么？说明一个 slab 就是若干个页框。

### 对象描述符

令人惊奇的事情是，对象描述符只是一个无符号整型.

    // include/asm-i386/types.h/line: 66
    typedef unsigned short kmem_bufctl_t;


但是，对象描述符的生命周期和存放位置，暂时还不是特别清楚。

## kmem\_cache 的初始化

    // mm/slab.c/line: 1187
    kmem_cache_t *
    kmem_cache_create (const char *name, size_t size, size_t align,
    	unsigned long flags, void (*ctor)(void*, kmem_cache_t *, unsigned long),
    	void (*dtor)(void*, kmem_cache_t *, unsigned long))
    {
    	...
    	// 对齐相关的代码
    	// 对齐的是对象嘛？应该是的
    	if (flags & SLAB_HWCACHE_ALIGN) {
    		ralign = cache_line_size();
    		while (size <= ralign/2)
    			ralign /= 2;
    	} else {
    		ralign = BYTES_PER_WORD;
    	}
    	...
    }


这个函数巨长，不过调试代码和注释也很多；可以推测这是一个非常重要的函数。后续细看。  
并且这个函数对 kmalloc 也没有依赖，这是一个很神奇的事情。  
需要后续重点研究下，**在没有 kmalloc 的情况下，内核是怎么动态开辟空间的**。  
目前的信息是，有一个 \_\_init data area 的存在，支持 kmalloc 前的内存申请。  
 

    // mm/slab.c/line: 744
    void __init kmem_cache_init(void)
    {
    	...
    	/* Bootstrap is tricky, because several objects are allocated
    	 * from caches that do not exist yet:
    	 * 1) initialize the cache_cache cache: it contains the kmem_cache_t
    	 *    structures of all caches, except cache_cache itself: cache_cache
    	 *    is statically allocated.
    	 *    Initially an __init data area is used for the head array, it's
    	 *    replaced with a kmalloc allocated array at the end of the bootstrap.
    	 * 2) Create the first kmalloc cache.
    	 *    The kmem_cache_t for the new cache is allocated normally. An __init
    	 *    data area is used for the head array.
    	 * 3) Create the remaining kmalloc caches, with minimally sized head arrays.
    	 * 4) Replace the __init data head arrays for cache_cache and the first
    	 *    kmalloc cache with kmalloc allocated arrays.
    	 * 5) Resize the head arrays of the kmalloc caches to their final sizes.
    	 */
    	// 这段注释很关键，关键信息有：
    	// 先使用 __init data area 把数据搞起来，把 kmalloc 的环境搞起来
    	// 再用 kmalloc 把 __init data area 替换掉
    	// 代码应该是比较复杂的，暂时不细分析了
    }


## 为 kmem\_cache 增加一个 slab

增加 slab 的入口是增加一个 object：内核没事不会自己去申请 slab，只有当需要一个 object 但是又没有空间（也就是没有可用的 slab）时，才会去申请一个新的 slab

### 入口的函数调用序列

    // mm/slab.c/line: 2297
    /**
     * kmem_cache_alloc - Allocate an object
    	...
     */
    void * kmem_cache_alloc (kmem_cache_t *cachep, int flags)
    {
    	return __cache_alloc(cachep, flags);
    }
    
    // mm/slab.c/line: 2135
    static inline void * __cache_alloc (kmem_cache_t *cachep, int flags)
    {
    	...
    	objp = cache_alloc_refill(cachep, flags);
    	...
    }
    
    // mm/slab.c/line: 1983
    static void* cache_alloc_refill(kmem_cache_t* cachep, int flags)
    {
    	...
    	x = cache_grow(cachep, flags, -1);
    	...
    }


### cache\_grow()

    // mm/slab.c/line: 1780
    static int cache_grow (kmem_cache_t * cachep, int flags, int nodeid)
    {
    	...
    	...		// 一系列的标志位检查和处理
    	...		/* Get colour for the slab, and cal the next value. */ // 还没看到
    	...
    
    	/* Get mem for the objs. */
    	if (!(objp = kmem_getpages(cachep, flags, nodeid)))
    		goto failed;
    	/* Get slab management. */
    	// 涉及到了 slab 着色等内容，后续详细分析
    	if (!(slabp = alloc_slabmgmt(cachep, objp, offset, local_flags)))
    		goto opps1;
    	set_slab_attr(cachep, slabp, objp);
    	cache_init_objs(cachep, slabp, ctor_flags);		// 会将整个 slab 都按 obj 来初始化
    	...
    }


    // include/linux/slab.h/line: 22
    #define	SLAB_NOFS		GFP_NOFS
    #define	SLAB_NOIO		GFP_NOIO
    #define	SLAB_ATOMIC		GFP_ATOMIC
    ...
    // 通过这些标志位控制 slab 相关的行为


    // mm/slab.c/line: 889
    static void *kmem_getpages(kmem_cache_t *cachep, int flags, int nodeid)
    {
    	...
    	addr = page_address(page);		// 注定不能使用高段内存
    
    	i = (1 << cachep->gfporder);	// 页框数
    	...
    	add_page_state(nr_slab, i);		// 暂时不清楚 page_state 的相关细节，好像是个每 CPU 变量
    	while (i--) {		// 每一页都会更新标志位
    		SetPageSlab(page);
    		page++;
    	}
    	return addr;
    }


   
所谓着色，就是：  
1）在同一类型的 slab 中  
2）按照 obj 的对齐粒度，让第一个 obj 的起始地址产生一个偏移；  
3）当然，后面的 obj 也会依次偏移  
4）一个偏移就是一个色，有多少个颜色是随缘的  
5）slab 的大小和 obj 的大小共同决定，slab 中不够一个 obj 空间作为着色的资源  
6）目的大概是为了提高硬件高速 cache 的利用率，具体细节暂时不详

    // mm/slab.c/line: 1678
    static struct slab* alloc_slabmgmt (kmem_cache_t *cachep,
    			void *objp, int colour_off, int local_flags)
    {
    	...
    	if (OFF_SLAB(cachep)) {
    		// 把 slab 描述符和对象描述符数组放到外面的这个机制有点复杂；暂时没有全理解
    		slabp = kmem_cache_alloc(cachep->slabp_cache, local_flags);
    		...
    	} else {
    		// objp 是申请到的页框的线性地址
    		// colour_off 是计算后的着色的总大小
    		slabp = objp+colour_off;
    		colour_off += cachep->slab_size;
    	}
    	slabp->inuse = 0;
    	slabp->colouroff = colour_off;		// 第一个 obj 在 slab 中的偏移量
    	slabp->s_mem = objp+colour_off;		// 第一 obj 的线性地址
    
    	return slabp;
    }


    // mm/slab.c/line: 1761
    static void set_slab_attr(kmem_cache_t *cachep, struct slab *slabp, void *objp)
    {
    	...
    	i = 1 << cachep->gfporder;
    	page = virt_to_page(objp);
    	do {
    		SET_PAGE_CACHE(page, cachep);
    		SET_PAGE_SLAB(page, slabp);
    		// 上面这两个宏定义展开以后分别是：
    		// ((page)->lru.next = (struct list_head *)(cachep))
    		// ((page)->lru.prev = (struct list_head *)(slabp))
    		// 就是说，lru 已经丧失了双向链表的作用；此时它们只是一组普通的指针
    
    		page++;
    	} while (--i);
    }


### array\_cache 管理

这块比较复杂，并且对外比较透明。  
暂时不细看了

### kmalloc

    // include/linux/slab.h/line: 85
    static inline void *kmalloc(size_t size, int flags)
    {
    	...
    	return __kmalloc(size, flags);
    }
    
    // mm/slab.c/line: 2454
    void * __kmalloc (size_t size, int flags)
    {
    	struct cache_sizes *csizep = malloc_sizes;
    
    	for (; csizep->cs_size; csizep++) {
    		if (size > csizep->cs_size)
    			continue;
    		...
    		return __cache_alloc(flags & GFP_DMA ?
    			 csizep->cs_dmacachep : csizep->cs_cachep, flags);
    	}
    	return NULL;
    }
    
    // mm/slab.c/line: 2135
    static inline void * __cache_alloc (kmem_cache_t *cachep, int flags)
    {
    	...
    	struct array_cache *ac;
    	...
    	ac = ac_data(cachep);
    	if (likely(ac->avail)) {	// 优先在缓存区申请
    		...
    		ac->touched = 1;
    		objp = ac_entry(ac)[--ac->avail];
    	} else {
    		...
    		objp = cache_alloc_refill(cachep, flags);
    	}
    	...
    	return objp;
    }


## 内存池



## 参考

[《深入理解Linux内核（第三版）》笔记（十一），第八章内存管理（二）_《深入理解linux内核》第三版-CSDN博客](https://blog.csdn.net/weixin_42346852/article/details/127225698?spm=1001.2014.3001.5506)