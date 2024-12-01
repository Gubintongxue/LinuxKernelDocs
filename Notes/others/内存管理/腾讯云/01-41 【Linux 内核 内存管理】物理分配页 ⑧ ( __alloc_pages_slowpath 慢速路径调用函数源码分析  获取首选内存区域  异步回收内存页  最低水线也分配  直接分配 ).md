#### 文章目录

-   [一、获取首选内存区域](https://cloud.tencent.com/developer?from_column=20421&from=20421)
-   [二、异步回收内存页](https://cloud.tencent.com/developer?from_column=20421&from=20421)
-   [三、最低水线也分配](https://cloud.tencent.com/developer?from_column=20421&from=20421)
-   [四、直接分配内存](https://cloud.tencent.com/developer?from_column=20421&from=20421)

在 [【Linux 内核 内存管理】物理分配页 ② ( \_\_alloc\_pages\_nodemask 函数参数分析 | \_\_alloc\_pages\_nodemask 函数分配物理页流程 )](https://cloud.tencent.com/developer/article/2253551?from_column=20421&from=20421) 博客中 , 分析了 `__alloc_pages_nodemask` 函数分配物理页流程如下 :

**首先** , 根据 `gfp_t gfp_mask` 分配标志位 参数 , 得到 " 内存节点 “ 的 首选 ” 区域类型 " 和 " 迁移类型 " ;

**然后** , 执行 " 快速路径 " , 第一次分配 尝试使用 低水线分配 ;

如果上述 " 快速路径 " 分配失败 , 则执行 " 慢速路径 " 分配 ;

上述涉及到了 " 快速路径 " 和 " 慢速路径 "

22

种物理页分配方式 ;

继续接着上一篇博客 [【Linux 内核 内存管理】物理分配页 ⑦ ( \_\_alloc\_pages\_slowpath 慢速路径调用函数源码分析 | 判断页阶数 | 读取 mems\_allowed | 分配标志位转换 )](https://cloud.tencent.com/developer/article/2253557?from_column=20421&from=20421) 分析 `__alloc_pages_slowpath` 慢速路径 内存分配 调用函数 的后续部分源码 ;

## 一、获取首选内存区域

* * *

获取 " 首选内存区域 " , 如果获取失败 , 则 `goto` 跳转到 `nopage` 标号位置运行后续代码 ;

代码语言：javascript

复制

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

**源码路径 :** linux-4.12\\mm\\page\_alloc.c#3731

## 二、异步回收内存页

* * *

调用 `wake_all_kswapds` 函数 , 异步 回收 物理内存页 ,

这里的异步 是通过 唤醒 " 回收线程 " 进行回收内存页的 ;

代码语言：javascript

复制

    	if (gfp_mask & __GFP_KSWAPD_RECLAIM)
    		wake_all_kswapds(order, ac);

**源码路径 :** linux-4.12\\mm\\page\_alloc.c#3736

## 三、最低水线也分配

* * *

调用 `get_page_from_freelist` 函数 , 使用 " 最低水线 " 进行物理页分配 ,

如果处理成功 , 则跳转到 `got_pg` 标号处执行 ;

代码语言：javascript

复制

    	/*
    	 * The adjusted alloc_flags might result in immediate success, so try
    	 * that first
    	 */
    	page = get_page_from_freelist(gfp_mask, order, alloc_flags, ac);
    	if (page)
    		goto got_pg;

**源码路径 :** linux-4.12\\mm\\page\_alloc.c#3743

## 四、直接分配内存

* * *

申请 物理页 内存 的阶数 , 满足以下

33

个条件 :

`can_direct_reclaim`

`(costly_order || (order > 0 && ac->migratetype != MIGRATE_MOVABLE))`

`!gfp_pfmemalloc_allowed(gfp_mask)`

执行该分支 " 直接分配内存 " 操作 ;

代码语言：javascript

复制

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

**源码路径 :** linux-4.12\\mm\\page\_alloc.c#3756

## 参考

[【Linux 内核 内存管理】物理分配页 ⑧ ( __alloc_pages_slowpath 慢速路径调用函数源码分析 | 获取首选内存区域 | 异步回收内存页 | 最低水线也分配 | 直接分配 )-腾讯云开发者社区-腾讯云 (tencent.com)](https://cloud.tencent.com/developer/article/2253558)