【Linux 内核 内存管理】物理分配页 ⑤ ( get_page_from_freelist 快速路径调用函数源码分析 | 遍历备用区域列表 | 启用 cpuset 检查判定 | 判定脏页数量 )

#### 文章目录

-   [一、遍历备用区域列表](https://cloud.tencent.com/developer?from_column=20421&from=20421)
-   [二、启用 cpuset 检查判定](https://cloud.tencent.com/developer?from_column=20421&from=20421)
-   [三、判定内存节点的脏页数量](https://cloud.tencent.com/developer?from_column=20421&from=20421)

在 [【Linux 内核 内存管理】物理分配页 ② ( \_\_alloc\_pages\_nodemask 函数参数分析 | \_\_alloc\_pages\_nodemask 函数分配物理页流程 )](https://cloud.tencent.com/developer/article/2253551?from_column=20421&from=20421) 博客中 , 分析了 `__alloc_pages_nodemask` 函数分配物理页流程如下 :

**首先** , 根据 `gfp_t gfp_mask` 分配标志位 参数 , 得到 " 内存节点 “ 的 首选 ” 区域类型 " 和 " 迁移类型 " ;

**然后** , 执行 " 快速路径 " , 第一次分配 尝试使用 低水线分配 ;

如果上述 " 快速路径 " 分配失败 , 则执行 " 慢速路径 " 分配 ;

上述涉及到了 " 快速路径 " 和 " 慢速路径 "

22

种物理页分配方式 ;

在 [【Linux 内核 内存管理】物理分配页 ④ ( \_\_alloc\_pages\_nodemask 函数源码分析 | 快速路径 | 慢速路径 | get\_page\_from\_freelist 源码 )](https://cloud.tencent.com/developer/tools/blog-entry?target=https%3A%2F%2Fblog.csdn.net%2Fshulianghan%2Farticle%2Fdetails%2F124400845&source=article&objectId=2253554) 博客中 , 介绍了 快速路径 主要调用 定义在 Linux 内核源码的 linux-4.12\\mm\\page\_alloc.c#3017 位置的 `get_page_from_freelist` 函数 , 分配物理页内存 ;

## 一、遍历备用区域列表

* * *

在 函数中 , 主要操作是遍历 备用区域列表 ,

查找满足如下条件的 内存区域 :

① 区域类型 小于等于 首选区域类型 ,

② 内存节点 对应的 节点掩码 位 被设置为 处理状态 ;

代码语言：javascript

复制

    static struct page *
    get_page_from_freelist(gfp_t gfp_mask, unsigned int order, int alloc_flags,
    						const struct alloc_context *ac)
    {
    	/*
    	 * Scan zonelist, looking for a zone with enough free.
    	 * See also __cpuset_node_allowed() comment in kernel/cpuset.c.
    	 */
    	for_next_zone_zonelist_nodemask(zone, z, ac->zonelist, ac->high_zoneidx,
    								ac->nodemask){}
    }

**源码路径 :** linux-4.12\\mm\\page\_alloc.c#3029

## 二、启用 cpuset 检查判定

* * *

如果 启用了 cpuset 功能 , 用户设置了 `ALLOC_CPUSET` 标志位 , 要求 检查 cpuset ,

如果 cpuset 不允许当前 进程 分配 该 内存节点 内存页 , 则直接 `continue` , 本次循环 " 备用区域列表 " 操作退出 , 执行下一次循环 ;

代码语言：javascript

复制

    static struct page *
    get_page_from_freelist(gfp_t gfp_mask, unsigned int order, int alloc_flags,
    						const struct alloc_context *ac)
    {
    	/*
    	 * Scan zonelist, looking for a zone with enough free.
    	 * See also __cpuset_node_allowed() comment in kernel/cpuset.c.
    	 */
    	for_next_zone_zonelist_nodemask(zone, z, ac->zonelist, ac->high_zoneidx,
    								ac->nodemask){
    		if (cpusets_enabled() &&
    			(alloc_flags & ALLOC_CPUSET) &&
    			!__cpuset_zone_allowed(zone, gfp_mask))
    				continue;
    	}
    }

**源码路径 :** linux-4.12\\mm\\page\_alloc.c#3034

相关标志位含义 , 参考 [【Linux 内核 内存管理】物理分配页 ③ ( 物理页分配标志位分析 | ALLOC\_WMARK\_MIN | ALLOC\_WMARK\_MASK | ALLOC\_HARDER )](https://cloud.tencent.com/developer/article/2253552?from_column=20421&from=20421) 博客 ;

`ALLOC_CPUSET` 宏定义 , 表示 检查 cpuset , 是否允许分配内存页 ;

代码语言：javascript

复制

    #define ALLOC_HARDER		0x10 /* try to alloc harder */
    #define ALLOC_HIGH		0x20 /* __GFP_HIGH set */
    #define ALLOC_CPUSET		0x40 /* check for correct cpuset */
    #define ALLOC_CMA		0x80 /* allow allocations from CMA areas */

**源码路径 :** linux-4.12\\mm\\internal.h#483

## 三、判定内存节点的脏页数量

* * *

调用者 假如 设置了 `__GFP_WRITE` 标志位 , 表明 文件系统 写文件 需要 申请一个页缓存 , 需要检查 " 内存节点 “ 中的 ” 脏页数量 " 是否超出了限制 ,

假如 超出了限制 , 也是 不能从 该 内存区域 分配内存 , `continue` 中断本次遍历 , 继续执行下一次遍历 ;

代码语言：javascript

复制

    static struct page *
    get_page_from_freelist(gfp_t gfp_mask, unsigned int order, int alloc_flags,
    						const struct alloc_context *ac)
    {
    	/*
    	 * Scan zonelist, looking for a zone with enough free.
    	 * See also __cpuset_node_allowed() comment in kernel/cpuset.c.
    	 */
    	for_next_zone_zonelist_nodemask(zone, z, ac->zonelist, ac->high_zoneidx,
    								ac->nodemask){
    		/*
    		 * When allocating a page cache page for writing, we
    		 * want to get it from a node that is within its dirty
    		 * limit, such that no single node holds more than its
    		 * proportional share of globally allowed dirty pages.
    		 * The dirty limits take into account the node's
    		 * lowmem reserves and high watermark so that kswapd
    		 * should be able to balance it without having to
    		 * write pages from its LRU list.
    		 *
    		 * XXX: For now, allow allocations to potentially
    		 * exceed the per-node dirty limit in the slowpath
    		 * (spread_dirty_pages unset) before going into reclaim,
    		 * which is important when on a NUMA setup the allowed
    		 * nodes are together not big enough to reach the
    		 * global limit.  The proper fix for these situations
    		 * will require awareness of nodes in the
    		 * dirty-throttling and the flusher threads.
    		 */
    		if (ac->spread_dirty_pages) {
    			if (last_pgdat_dirty_limit == zone->zone_pgdat)
    				continue;
    
    			if (!node_dirty_ok(zone->zone_pgdat)) {
    				last_pgdat_dirty_limit = zone->zone_pgdat;
    				continue;
    			}
    		}
    	}
    }

**源码路径 :** linux-4.12\\mm\\internal.h#3057

## 参考

[【Linux 内核 内存管理】物理分配页 ⑤ ( get_page_from_freelist 快速路径调用函数源码分析 | 遍历备用区域列表 | 启用 cpuset 检查判定 | 判定脏页数量 )-腾讯云开发者社区-腾讯云 (tencent.com)](https://cloud.tencent.com/developer/article/2253554)