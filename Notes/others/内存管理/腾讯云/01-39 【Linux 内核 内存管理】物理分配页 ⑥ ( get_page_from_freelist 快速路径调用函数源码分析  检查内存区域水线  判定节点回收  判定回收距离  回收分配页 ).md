【Linux 内核 内存管理】物理分配页 ⑥ ( get_page_from_freelist 快速路径调用函数源码分析 | 检查内存区域水线 | 判定节点回收 | 判定回收距离 | 回收分配页 )

#### 文章目录

-   [一、检查内存区域水线](https://cloud.tencent.com/developer?from_column=20421&from=20421)
-   [二、判定节点收回是否开启、回收距离是否合法](https://cloud.tencent.com/developer?from_column=20421&from=20421)
-   [三、回收没有使用的页、再次检查区域水线](https://cloud.tencent.com/developer?from_column=20421&from=20421)
-   [四、分配物理页](https://cloud.tencent.com/developer?from_column=20421&from=20421)
-   [五、本博客涉及到的处理过程源码](https://cloud.tencent.com/developer?from_column=20421&from=20421)

在 [【Linux 内核 内存管理】物理分配页 ② ( \_\_alloc\_pages\_nodemask 函数参数分析 | \_\_alloc\_pages\_nodemask 函数分配物理页流程 )](https://cloud.tencent.com/developer/article/2253551?from_column=20421&from=20421) 博客中 , 分析了 `__alloc_pages_nodemask` 函数分配物理页流程如下 :

**首先** , 根据 `gfp_t gfp_mask` 分配标志位 参数 , 得到 " 内存节点 “ 的 首选 ” 区域类型 " 和 " 迁移类型 " ;

**然后** , 执行 " 快速路径 " , 第一次分配 尝试使用 低水线分配 ;

如果上述 " 快速路径 " 分配失败 , 则执行 " 慢速路径 " 分配 ;

上述涉及到了 " 快速路径 " 和 " 慢速路径 "

22

种物理页分配方式 ;

在 [【Linux 内核 内存管理】物理分配页 ④ ( \_\_alloc\_pages\_nodemask 函数源码分析 | 快速路径 | 慢速路径 | get\_page\_from\_freelist 源码 )](https://cloud.tencent.com/developer/tools/blog-entry?target=https%3A%2F%2Fblog.csdn.net%2Fshulianghan%2Farticle%2Fdetails%2F124400845&source=article&objectId=2253555) 博客中 , 介绍了 快速路径 主要调用 定义在 Linux 内核源码的 linux-4.12\\mm\\page\_alloc.c#3017 位置的 `get_page_from_freelist` 函数 , 分配物理页内存 ;

接着 [【Linux 内核 内存管理】物理分配页 ⑤ ( get\_page\_from\_freelist 快速路径调用函数源码分析 | 遍历备用区域列表 | 启用 cpuset 检查判定 | 判定脏页数量 )](https://cloud.tencent.com/developer/tools/blog-entry?target=https%3A%2F%2Fblog.csdn.net%2Fshulianghan%2Farticle%2Fdetails%2F124401424&source=article&objectId=2253555) 博客 , 分析 `get_page_from_freelist` 函数中的源码 ;

## 一、检查内存区域水线

* * *

在 `get_page_from_freelist` 快速路径调用函数 中 , 执行如下操作 :

遍历备用区域列表

启用 cpuset 检查判定

判定脏页数量

然后 , 检查 内存区域水线 , 如果 内存区域 " 空闲页数 - 申请内存页数 " 小于 区域水线 , 则执行对应操作 ;

先获取 区域水线 ,

代码语言：javascript

复制

    mark = zone->watermark[alloc_flags & ALLOC_WMARK_MASK];

**源码路径 :** linux-4.12\\mm\\page\_alloc.c#3067

判定 内存区域 " 空闲页数 - 申请内存页数 " 是否 小于 区域水线 , 如果小于 , 则命中该分支 , 执行对应操作 ;

代码语言：javascript

复制

    		if (!zone_watermark_fast(zone, order, mark,
    				       ac_classzone_idx(ac), alloc_flags))

**源码路径 :** linux-4.12\\mm\\page\_alloc.c#3068

## 二、判定节点收回是否开启、回收距离是否合法

* * *

假如 当前 内存节点 没有开启 节点回收 功能 , 或者 当前内存节点 距离 首选节点 的长度 大于 " 回收距离 " ,

则 不能从该 " 内存区域 " 分配 物理页 , `continue` 中断本次循环 , 继续遍历其它 内存区域 ;

代码语言：javascript

复制

    			if (node_reclaim_mode == 0 ||
    			    !zone_allows_reclaim(ac->preferred_zoneref->zone, zone))
    				continue;

**源码路径 :** linux-4.12\\mm\\page\_alloc.c#3077

## 三、回收没有使用的页、再次检查区域水线

* * *

从 内存节点 回收 申请的 没有被映射到 进程虚拟地址空间 的 物理页 ,

再次 检查 内存区域水线 ,

如果 内存区域 " 空闲页数 - 申请内存页数 " 小于 区域水线 ,

则 不能从该 " 内存区域 " 分配 物理页 , `continue` 中断本次循环 , 继续遍历其它 内存区域 ;

代码语言：javascript

复制

    			ret = node_reclaim(zone->zone_pgdat, gfp_mask, order);
    			switch (ret) {
    			case NODE_RECLAIM_NOSCAN:
    				/* did not scan */
    				continue;
    			case NODE_RECLAIM_FULL:
    				/* scanned but unreclaimable */
    				continue;
    			default:
    				/* did we reclaim enough */
    				if (zone_watermark_ok(zone, order, mark,
    						ac_classzone_idx(ac), alloc_flags))
    					goto try_this_zone;
    
    				continue;
    			}

**源码路径 :** linux-4.12\\mm\\page\_alloc.c#3081

## 四、分配物理页

* * *

如果上述判定都得到满足 ,

则 调用 `rmqueue` 函数 , 从当前 内存区域 分配 物理页 ,

如果分配成功 , `page` 不为 0 , 则 `if (page)` 分支命中 , 调用 `prep_new_page` 函数 , 初始化 物理页 ;

代码语言：javascript

复制

    try_this_zone:
    		page = rmqueue(ac->preferred_zoneref->zone, zone, order,
    				gfp_mask, alloc_flags, ac->migratetype);
    		if (page) {
    			prep_new_page(page, order, gfp_mask, alloc_flags);
    
    			/*
    			 * If this is a high-order atomic allocation then check
    			 * if the pageblock should be reserved for the future
    			 */
    			if (unlikely(order && (alloc_flags & ALLOC_HARDER)))
    				reserve_highatomic_pageblock(page, zone, order);
    
    			return page;
    		}
    	}

**源码路径 :** linux-4.12\\mm\\page\_alloc.c#3099

## 五、本博客涉及到的处理过程源码

* * *

**本博客涉及到的处理过程源码 :**

代码语言：javascript

复制

    		mark = zone->watermark[alloc_flags & ALLOC_WMARK_MASK];
    		if (!zone_watermark_fast(zone, order, mark,
    				       ac_classzone_idx(ac), alloc_flags)) {
    			int ret;
    
    			/* Checked here to keep the fast path fast */
    			BUILD_BUG_ON(ALLOC_NO_WATERMARKS < NR_WMARK);
    			if (alloc_flags & ALLOC_NO_WATERMARKS)
    				goto try_this_zone;
    
    			if (node_reclaim_mode == 0 ||
    			    !zone_allows_reclaim(ac->preferred_zoneref->zone, zone))
    				continue;
    
    			ret = node_reclaim(zone->zone_pgdat, gfp_mask, order);
    			switch (ret) {
    			case NODE_RECLAIM_NOSCAN:
    				/* did not scan */
    				continue;
    			case NODE_RECLAIM_FULL:
    				/* scanned but unreclaimable */
    				continue;
    			default:
    				/* did we reclaim enough */
    				if (zone_watermark_ok(zone, order, mark,
    						ac_classzone_idx(ac), alloc_flags))
    					goto try_this_zone;
    
    				continue;
    			}
    		}

**源码路径 :** linux-4.12\\mm\\page\_alloc.c#3067

## 参考

[【Linux 内核 内存管理】物理分配页 ⑥ ( get_page_from_freelist 快速路径调用函数源码分析 | 检查内存区域水线 | 判定节点回收 | 判定回收距离 | 回收分配页 )-腾讯云开发者社区-腾讯云 (tencent.com)](https://cloud.tencent.com/developer/article/2253555)