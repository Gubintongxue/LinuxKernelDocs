【Linux 内核 内存管理】物理分配页 ⑨ ( __alloc_pages_slowpath 慢速路径调用函数源码分析 | retry 标号代码分析 )

#### 文章目录

-   [一、retry 标号代码分析](https://cloud.tencent.com/developer?from_column=20421&from=20421)
-   [二、retry 标号完整代码](https://cloud.tencent.com/developer?from_column=20421&from=20421)

在 [【Linux 内核 内存管理】物理分配页 ② ( \_\_alloc\_pages\_nodemask 函数参数分析 | \_\_alloc\_pages\_nodemask 函数分配物理页流程 )](https://cloud.tencent.com/developer/article/2253551?from_column=20421&from=20421) 博客中 , 分析了 `__alloc_pages_nodemask` 函数分配物理页流程如下 :

**首先** , 根据 `gfp_t gfp_mask` 分配标志位 参数 , 得到 " 内存节点 “ 的 首选 ” 区域类型 " 和 " 迁移类型 " ;

**然后** , 执行 " 快速路径 " , 第一次分配 尝试使用 低水线分配 ;

如果上述 " 快速路径 " 分配失败 , 则执行 " 慢速路径 " 分配 ;

上述涉及到了 " 快速路径 " 和 " 慢速路径 "

22

种物理页分配方式 ;

继续接着上一篇博客 [【Linux 内核 内存管理】物理分配页 ⑧ ( \_\_alloc\_pages\_slowpath 慢速路径调用函数源码分析 | 获取首选内存区域 | 异步回收内存页 | 最低水线也分配 | 直接分配 )](https://cloud.tencent.com/developer/article/2253558?from_column=20421&from=20421) 分析 `__alloc_pages_slowpath` 慢速路径 内存分配 调用函数 的后续部分源码 ;

## 一、retry 标号代码分析

* * *

下面开始分析 `__alloc_pages_slowpath` 慢速路径 内存分配 调用函数 中的 `retry` 标号下的代码 ,

调用 `wake_all_kswapds` 函数 , 确保 " 页回收线程 " 在遍历时 保持唤醒状态 , 不会由于意外导致休眠 ;

代码语言：javascript

复制

    retry:
    	/* Ensure kswapd doesn't accidentally go to sleep as long as we loop */
    	if (gfp_mask & __GFP_KSWAPD_RECLAIM)
    		wake_all_kswapds(order, ac);

**源码路径 :** linux-4.12\\mm\\page\_alloc.c#3794

调用 `get_page_from_freelist` 函数 , 尝试使用 调整过的 区域列表 和 分配标志位 进行 内存分配 , 如果 内存分配成功 , 则跳转到 `got_pg` 标号执行 ;

代码语言：javascript

复制

    	/* Attempt with potentially adjusted zonelist and alloc_flags */
    	page = get_page_from_freelist(gfp_mask, order, alloc_flags, ac);
    	if (page)
    		goto got_pg;

**源码路径 :** linux-4.12\\mm\\page\_alloc.c#3811

如果 调用者 不想等待 浪费时间 , 则不执行后续操作 , 跳转到 `nopage` 处执行 后续代码 ;

代码语言：javascript

复制

    	/* Caller is not willing to reclaim, we can't balance anything */
    	if (!can_direct_reclaim)
    		goto nopage;

**源码路径 :** linux-4.12\\mm\\page\_alloc.c#3817

调用 `__alloc_pages_direct_reclaim` 函数 , 直接进行页回收 ;

代码语言：javascript

复制

    	/* Try direct reclaim and then allocating */
    	page = __alloc_pages_direct_reclaim(gfp_mask, order, alloc_flags, ac,
    							&did_some_progress);
    	if (page)
    		goto got_pg;

**源码路径 :** linux-4.12\\mm\\page\_alloc.c#3833

调用 `__alloc_pages_direct_compact` 函数 , 针对申请 物理页 阶数 大于 0 的情况 , 执行 同步模式 下的 内存碎片整理 操作 ;

代码语言：javascript

复制

    	/* Try direct compaction and then allocating */
    	page = __alloc_pages_direct_compact(gfp_mask, order, alloc_flags, ac,
    					compact_priority, &compact_result);
    	if (page)
    		goto got_pg;

**源码路径 :** linux-4.12\\mm\\page\_alloc.c#3839

再次进行判断 , 如果调用者 不想进行循环 , 则放弃内存申请 , 跳转到 `nopage` 标号执行 ;

代码语言：javascript

复制

    	/* Do not loop if specifically requested */
    	if (gfp_mask & __GFP_NORETRY)
    		goto nopage;

**源码路径 :** linux-4.12\\mm\\page\_alloc.c#3845

如果 申请 物理页 阶数 大于 0 , 则调用 `should_compact_retry` 函数 , 判断是否重新尝试 执行 内存碎片整理操作 , 如果判定成功 , 则继续跳转到 `retry` 标号处再执行一遍 ;

代码语言：javascript

复制

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

**源码路径 :** linux-4.12\\mm\\page\_alloc.c#3865

调用 `read_mems_allowed_retry` 函数 , 判定 cpuset 是否允许修改当前进程 从 指定的内存节点申请 物理页内存 ;

代码语言：javascript

复制

    	/*
    	 * It's possible we raced with cpuset update so the OOM would be
    	 * premature (see below the nopage: label for full explanation).
    	 */
    	if (read_mems_allowed_retry(cpuset_mems_cookie))
    		goto retry_cpuset;

**源码路径 :** linux-4.12\\mm\\page\_alloc.c#3875

调用 `__alloc_pages_may_oom` 函数 , 如果内存耗尽 , 分配内存失败 , 则杀死一个进程 , 以获取足够的内存空间 ;

代码语言：javascript

复制

    	/* Reclaim has failed us, start killing things */
    	page = __alloc_pages_may_oom(gfp_mask, order, ac, &did_some_progress);
    	if (page)
    		goto got_pg;

**源码路径 :** linux-4.12\\mm\\page\_alloc.c#3879

假如 当前进程 出现内存耗尽的情况 , 则忽略 最低水线 的限制 , 或者 不允许使用 紧急保留内存 ;

代码语言：javascript

复制

    	/* Avoid allocations with no watermarks from looping endlessly */
    	if (test_thread_flag(TIF_MEMDIE) &&
    	    (alloc_flags == ALLOC_NO_WATERMARKS ||
    	     (gfp_mask & __GFP_NOMEMALLOC)))
    		goto nopage;

**源码路径 :** linux-4.12\\mm\\page\_alloc.c#3884

内存耗尽杀手 取得一定进展 , 继续跳转到 `retry` 标号重新尝试分配内存 ;

代码语言：javascript

复制

    	/* Retry as long as the OOM killer is making progress */
    	if (did_some_progress) {
    		no_progress_loops = 0;
    		goto retry;
    	}

**源码路径 :** linux-4.12\\mm\\page\_alloc.c#3890

## 二、retry 标号完整代码

* * *

`retry` **标号完整代码 :**

代码语言：javascript

复制

    static inline struct page *
    __alloc_pages_slowpath(gfp_t gfp_mask, unsigned int order,
    						struct alloc_context *ac)
    {
    	...
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
    	...
    }

**源码路径 :** linux-4.12\\mm\\page\_alloc.c#3792

## 参考

[【Linux 内核 内存管理】物理分配页 ⑨ ( __alloc_pages_slowpath 慢速路径调用函数源码分析 | retry 标号代码分析 )-腾讯云开发者社区-腾讯云 (tencent.com)](https://cloud.tencent.com/developer/article/2253560)