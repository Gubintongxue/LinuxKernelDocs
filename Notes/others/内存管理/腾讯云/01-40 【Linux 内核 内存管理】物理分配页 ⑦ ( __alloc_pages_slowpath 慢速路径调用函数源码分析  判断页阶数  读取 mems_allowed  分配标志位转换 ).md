【Linux 内核 内存管理】物理分配页 ⑦ ( __alloc_pages_slowpath 慢速路径调用函数源码分析 | 判断页阶数 | 读取 mems_allowed | 分配标志位转换 )

#### 文章目录

-   [一、\_\_alloc\_pages\_slowpath 慢速路径调用函数](https://cloud.tencent.com/developer?from_column=20421&from=20421)
-   [二、判断页阶数](https://cloud.tencent.com/developer?from_column=20421&from=20421)
-   [三、读取进程 mems\_allowed 成员](https://cloud.tencent.com/developer?from_column=20421&from=20421)
-   [四、分配标志位转换](https://cloud.tencent.com/developer?from_column=20421&from=20421)
-   [五、\_\_alloc\_pages\_slowpath 慢速路径调用完整函数源码](https://cloud.tencent.com/developer?from_column=20421&from=20421)

在 [【Linux 内核 内存管理】物理分配页 ② ( \_\_alloc\_pages\_nodemask 函数参数分析 | \_\_alloc\_pages\_nodemask 函数分配物理页流程 )](https://cloud.tencent.com/developer/article/2253551?from_column=20421&from=20421) 博客中 , 分析了 `__alloc_pages_nodemask` 函数分配物理页流程如下 :

**首先** , 根据 `gfp_t gfp_mask` 分配标志位 参数 , 得到 " 内存节点 “ 的 首选 ” 区域类型 " 和 " 迁移类型 " ;

**然后** , 执行 " 快速路径 " , 第一次分配 尝试使用 低水线分配 ;

如果上述 " 快速路径 " 分配失败 , 则执行 " 慢速路径 " 分配 ;

上述涉及到了 " 快速路径 " 和 " 慢速路径 "

22

种物理页分配方式 ;

前面几篇博客 , 分析了 " 快速路径 " 内存分配核心函数 `get_page_from_freelist` , 本博客开始分析 " 慢速路径 " 内存分配 函数 `__alloc_pages_slowpath` 函数 ;

## 一、\_\_alloc\_pages\_slowpath 慢速路径调用函数

* * *

内存区域 内 进行 物理页分配 时 , 优先尝试使用 " 快速路径 " 内存分配 , 执行 `get_page_from_freelist` 核心函数 ;

假如上述 " 低水线内存分配 " 分配 , 即 " 快速路径 " 内存分配失败 , 则执行 " 慢速路径 " 内存分配 ;

" 慢速路径 " 内存分配 的核心函数 是 `__alloc_pages_slowpath` 函数 , 定义在 Linux 内核源码的 linux-4.12\\mm\\page\_alloc.c#3676 位置 ;

![](https://developer.qcloudimg.com/http-save/yehe-2542479/897d56509b175a6f7d0d1c3641e049c0.png)

**源码路径 :** linux-4.12\\mm\\page\_alloc.c#3676

## 二、判断页阶数

* * *

先判断 内存分配 的 物理页的 阶数 , 申请 物理页内存 的 " 阶数 " , 必须 小于 页分配器 支持的 最大分配 阶数 ;

**阶 ( Order ) :** 物理页 的 数量单位 ,

nn

阶页块 指的是

2n2^n

个 连续的 " 物理页 " ; 完整概念参考 [【Linux 内核 内存管理】伙伴分配器 ① ( 伙伴分配器引入 | 页块、阶 | 伙伴 )](https://cloud.tencent.com/developer/tools/blog-entry?target=https%3A%2F%2Fblog.csdn.net%2Fshulianghan%2Farticle%2Fdetails%2F124304776&source=article&objectId=2253557) ;

代码语言：javascript

复制

    	/*
    	 * In the slowpath, we sanity check order to avoid ever trying to
    	 * reclaim >= MAX_ORDER areas which will never succeed. Callers may
    	 * be using allocators in order of preference for an area that is
    	 * too large.
    	 */
    	if (order >= MAX_ORDER) {
    		WARN_ON_ONCE(!(gfp_mask & __GFP_NOWARN));
    		return NULL;
    	}

**源码路径 :** linux-4.12\\mm\\page\_alloc.c#3699

## 三、读取进程 mems\_allowed 成员

* * *

在后面代码中 , 会 检查 cpuset , 查看是否允许 当前进程 从 内存节点 申请 物理页 ,

上述判断 , 需要读取 当前进程的 mems\_allowed 成员 , 读取时需要使用 " 顺序保护锁 " ;

代码语言：javascript

复制

    	cpuset_mems_cookie = read_mems_allowed_begin();

**源码路径 :** linux-4.12\\mm\\page\_alloc.c#3716

## 四、分配标志位转换

* * *

将 " 分配标志位 " 转为 " 内部分配标志位 " ;

代码语言：javascript

复制

    	/*
    	 * The fast path uses conservative alloc_flags to succeed only until
    	 * kswapd needs to be woken up, and to avoid the cost of setting up
    	 * alloc_flags precisely. So we do that now.
    	 */
    	alloc_flags = gfp_to_alloc_flags(gfp_mask);

**源码路径 :** linux-4.12\\mm\\page\_alloc.c#3723

## 五、\_\_alloc\_pages\_slowpath 慢速路径调用完整函数源码

* * *

代码语言：javascript

复制

    static inline struct page *
    __alloc_pages_slowpath(gfp_t gfp_mask, unsigned int order,
    						struct alloc_context *ac)
    {
    	bool can_direct_reclaim = gfp_mask & __GFP_DIRECT_RECLAIM;
    	const bool costly_order = order > PAGE_ALLOC_COSTLY_ORDER;
    	struct page *page = NULL;
    	unsigned int alloc_flags;
    	unsigned long did_some_progress;
    	enum compact_priority compact_priority;
    	enum compact_result compact_result;
    	int compaction_retries;
    	int no_progress_loops;
    	unsigned long alloc_start = jiffies;
    	unsigned int stall_timeout = 10 * HZ;
    	unsigned int cpuset_mems_cookie;
    
    	/*
    	 * In the slowpath, we sanity check order to avoid ever trying to
    	 * reclaim >= MAX_ORDER areas which will never succeed. Callers may
    	 * be using allocators in order of preference for an area that is
    	 * too large.
    	 */
    	if (order >= MAX_ORDER) {
    		WARN_ON_ONCE(!(gfp_mask & __GFP_NOWARN));
    		return NULL;
    	}
    
    	/*
    	 * We also sanity check to catch abuse of atomic reserves being used by
    	 * callers that are not in atomic context.
    	 */
    	if (WARN_ON_ONCE((gfp_mask & (__GFP_ATOMIC|__GFP_DIRECT_RECLAIM)) ==
    				(__GFP_ATOMIC|__GFP_DIRECT_RECLAIM)))
    		gfp_mask &= ~__GFP_ATOMIC;
    
    retry_cpuset:
    	compaction_retries = 0;
    	no_progress_loops = 0;
    	compact_priority = DEF_COMPACT_PRIORITY;
    	cpuset_mems_cookie = read_mems_allowed_begin();
    
    	/*
    	 * The fast path uses conservative alloc_flags to succeed only until
    	 * kswapd needs to be woken up, and to avoid the cost of setting up
    	 * alloc_flags precisely. So we do that now.
    	 */
    	alloc_flags = gfp_to_alloc_flags(gfp_mask);
    
    	/*
    	 * We need to recalculate the starting point for the zonelist iterator
    	 * because we might have used different nodemask in the fast path, or
    	 * there was a cpuset modification and we are retrying - otherwise we
    	 * could end up iterating over non-eligible zones endlessly.
    	 */
    	ac->preferred_zoneref = first_zones_zonelist(ac->zonelist,
    					ac->high_zoneidx, ac->nodemask);
    	if (!ac->preferred_zoneref->zone)
    		goto nopage;
    
    	if (gfp_mask & __GFP_KSWAPD_RECLAIM)
    		wake_all_kswapds(order, ac);
    
    	/*
    	 * The adjusted alloc_flags might result in immediate success, so try
    	 * that first
    	 */
    	page = get_page_from_freelist(gfp_mask, order, alloc_flags, ac);
    	if (page)
    		goto got_pg;
    
    	/*
    	 * For costly allocations, try direct compaction first, as it's likely
    	 * that we have enough base pages and don't need to reclaim. For non-
    	 * movable high-order allocations, do that as well, as compaction will
    	 * try prevent permanent fragmentation by migrating from blocks of the
    	 * same migratetype.
    	 * Don't try this for allocations that are allowed to ignore
    	 * watermarks, as the ALLOC_NO_WATERMARKS attempt didn't yet happen.
    	 */
    	if (can_direct_reclaim &&
    			(costly_order ||
    			   (order > 0 && ac->migratetype != MIGRATE_MOVABLE))
    			&& !gfp_pfmemalloc_allowed(gfp_mask)) {
    		page = __alloc_pages_direct_compact(gfp_mask, order,
    						alloc_flags, ac,
    						INIT_COMPACT_PRIORITY,
    						&compact_result);
    		if (page)
    			goto got_pg;
    
    		/*
    		 * Checks for costly allocations with __GFP_NORETRY, which
    		 * includes THP page fault allocations
    		 */
    		if (costly_order && (gfp_mask & __GFP_NORETRY)) {
    			/*
    			 * If compaction is deferred for high-order allocations,
    			 * it is because sync compaction recently failed. If
    			 * this is the case and the caller requested a THP
    			 * allocation, we do not want to heavily disrupt the
    			 * system, so we fail the allocation instead of entering
    			 * direct reclaim.
    			 */
    			if (compact_result == COMPACT_DEFERRED)
    				goto nopage;
    
    			/*
    			 * Looks like reclaim/compaction is worth trying, but
    			 * sync compaction could be very expensive, so keep
    			 * using async compaction.
    			 */
    			compact_priority = INIT_COMPACT_PRIORITY;
    		}
    	}
    
    retry:
    	/* Ensure kswapd doesn't accidentally go to sleep as long as we loop */
    	if (gfp_mask & __GFP_KSWAPD_RECLAIM)
    		wake_all_kswapds(order, ac);
    
    	if (gfp_pfmemalloc_allowed(gfp_mask))
    		alloc_flags = ALLOC_NO_WATERMARKS;
    
    	/*
    	 * Reset the zonelist iterators if memory policies can be ignored.
    	 * These allocations are high priority and system rather than user
    	 * orientated.
    	 */
    	if (!(alloc_flags & ALLOC_CPUSET) || (alloc_flags & ALLOC_NO_WATERMARKS)) {
    		ac->zonelist = node_zonelist(numa_node_id(), gfp_mask);
    		ac->preferred_zoneref = first_zones_zonelist(ac->zonelist,
    					ac->high_zoneidx, ac->nodemask);
    	}
    
    	/* Attempt with potentially adjusted zonelist and alloc_flags */
    	page = get_page_from_freelist(gfp_mask, order, alloc_flags, ac);
    	if (page)
    		goto got_pg;
    
    	/* Caller is not willing to reclaim, we can't balance anything */
    	if (!can_direct_reclaim)
    		goto nopage;
    
    	/* Make sure we know about allocations which stall for too long */
    	if (time_after(jiffies, alloc_start + stall_timeout)) {
    		warn_alloc(gfp_mask & ~__GFP_NOWARN, ac->nodemask,
    			"page allocation stalls for %ums, order:%u",
    			jiffies_to_msecs(jiffies-alloc_start), order);
    		stall_timeout += 10 * HZ;
    	}
    
    	/* Avoid recursion of direct reclaim */
    	if (current->flags & PF_MEMALLOC)
    		goto nopage;
    
    	/* Try direct reclaim and then allocating */
    	page = __alloc_pages_direct_reclaim(gfp_mask, order, alloc_flags, ac,
    							&did_some_progress);
    	if (page)
    		goto got_pg;
    
    	/* Try direct compaction and then allocating */
    	page = __alloc_pages_direct_compact(gfp_mask, order, alloc_flags, ac,
    					compact_priority, &compact_result);
    	if (page)
    		goto got_pg;
    
    	/* Do not loop if specifically requested */
    	if (gfp_mask & __GFP_NORETRY)
    		goto nopage;
    
    	/*
    	 * Do not retry costly high order allocations unless they are
    	 * __GFP_REPEAT
    	 */
    	if (costly_order && !(gfp_mask & __GFP_REPEAT))
    		goto nopage;
    
    	if (should_reclaim_retry(gfp_mask, order, ac, alloc_flags,
    				 did_some_progress > 0, &no_progress_loops))
    		goto retry;
    
    	/*
    	 * It doesn't make any sense to retry for the compaction if the order-0
    	 * reclaim is not able to make any progress because the current
    	 * implementation of the compaction depends on the sufficient amount
    	 * of free memory (see __compaction_suitable)
    	 */
    	if (did_some_progress > 0 &&
    			should_compact_retry(ac, order, alloc_flags,
    				compact_result, &compact_priority,
    				&compaction_retries))
    		goto retry;
    
    	/*
    	 * It's possible we raced with cpuset update so the OOM would be
    	 * premature (see below the nopage: label for full explanation).
    	 */
    	if (read_mems_allowed_retry(cpuset_mems_cookie))
    		goto retry_cpuset;
    
    	/* Reclaim has failed us, start killing things */
    	page = __alloc_pages_may_oom(gfp_mask, order, ac, &did_some_progress);
    	if (page)
    		goto got_pg;
    
    	/* Avoid allocations with no watermarks from looping endlessly */
    	if (test_thread_flag(TIF_MEMDIE) &&
    	    (alloc_flags == ALLOC_NO_WATERMARKS ||
    	     (gfp_mask & __GFP_NOMEMALLOC)))
    		goto nopage;
    
    	/* Retry as long as the OOM killer is making progress */
    	if (did_some_progress) {
    		no_progress_loops = 0;
    		goto retry;
    	}
    
    nopage:
    	/*
    	 * When updating a task's mems_allowed or mempolicy nodemask, it is
    	 * possible to race with parallel threads in such a way that our
    	 * allocation can fail while the mask is being updated. If we are about
    	 * to fail, check if the cpuset changed during allocation and if so,
    	 * retry.
    	 */
    	if (read_mems_allowed_retry(cpuset_mems_cookie))
    		goto retry_cpuset;
    
    	/*
    	 * Make sure that __GFP_NOFAIL request doesn't leak out and make sure
    	 * we always retry
    	 */
    	if (gfp_mask & __GFP_NOFAIL) {
    		/*
    		 * All existing users of the __GFP_NOFAIL are blockable, so warn
    		 * of any new users that actually require GFP_NOWAIT
    		 */
    		if (WARN_ON_ONCE(!can_direct_reclaim))
    			goto fail;
    
    		/*
    		 * PF_MEMALLOC request from this context is rather bizarre
    		 * because we cannot reclaim anything and only can loop waiting
    		 * for somebody to do a work for us
    		 */
    		WARN_ON_ONCE(current->flags & PF_MEMALLOC);
    
    		/*
    		 * non failing costly orders are a hard requirement which we
    		 * are not prepared for much so let's warn about these users
    		 * so that we can identify them and convert them to something
    		 * else.
    		 */
    		WARN_ON_ONCE(order > PAGE_ALLOC_COSTLY_ORDER);
    
    		/*
    		 * Help non-failing allocations by giving them access to memory
    		 * reserves but do not use ALLOC_NO_WATERMARKS because this
    		 * could deplete whole memory reserves which would just make
    		 * the situation worse
    		 */
    		page = __alloc_pages_cpuset_fallback(gfp_mask, order, ALLOC_HARDER, ac);
    		if (page)
    			goto got_pg;
    
    		cond_resched();
    		goto retry;
    	}
    fail:
    	warn_alloc(gfp_mask, ac->nodemask,
    			"page allocation failure: order:%u", order);
    got_pg:
    	return page;
    }

**源码路径 :** linux-4.12\\mm\\page\_alloc.c#3676

## 参考

[【Linux 内核 内存管理】物理分配页 ⑦ ( __alloc_pages_slowpath 慢速路径调用函数源码分析 | 判断页阶数 | 读取 mems_allowed | 分配标志位转换 )-腾讯云开发者社区-腾讯云 (tencent.com)](https://cloud.tencent.com/developer/article/2253557)