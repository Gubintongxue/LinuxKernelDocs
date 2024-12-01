【Linux 内核 内存管理】memblock 分配器编程接口 ⑤ ( memblock_free 函数 | memblock_remove_range 函数 )

#### 文章目录

-   [一、memblock\_free 函数分析](https://cloud.tencent.com/developer?from_column=20421&from=20421)
-   [二、memblock\_remove\_range 函数分析](https://cloud.tencent.com/developer?from_column=20421&from=20421)

**memblock 分配器提供了如下编程接口 :**

**① 添加内存 :** `memblock_add` 函数 , 将 内存块区域 添加到 `memblock.memory` 成员中 , 即 插入一块可用的物理内存 ;

**② 删除内存 :** `memblock_remove` 函数 , 删除 内存块区域 ;

**③ 分配内存 :** `memblock_alloc` 函数 , 申请分配内存 ;

**④ 释放内存 :** `memblock_free` 函数 , 释放之前分配的内存 ;

在之前的博客中介绍了 `memblock_add` `memblock_remove` `memblock_alloc` 函数源码 , 本篇博客开始介绍 `memblock_free` 分配内存函数 ;

## 一、memblock\_free 函数分析

* * *

`memblock_free` 函数 的作用是 释放内存 , 从 " 预留内存区域 " 中, 删除一块 内存块 ;

**`memblock_free` 函数参数说明 :**

-   `phys_addr_t base` 参数 表示 要删除的内存区域的 起始地址 ;
-   `phys_addr_t size` 参数 表示 要删除的内存区域的 大小 ;

在 `memblock_free` 函数中 ,

调用 `kmemleak_free_part_phys` 函数 , 计算 要删除的 物理内存区域 的 终止地址 ,

最后调用了 `memblock_remove_range` 函数 , 继续向后执行 ;

`memblock_free` 函数 定义在 Linux 内核源码的 linux-4.12\\mm\\memblock.c#710 位置 ;

代码语言：javascript

复制

    int __init_memblock memblock_free(phys_addr_t base, phys_addr_t size)
    {
    	phys_addr_t end = base + size - 1;
    
    	memblock_dbg("   memblock_free: [%pa-%pa] %pF\n",
    		     &base, &end, (void *)_RET_IP_);
    
    	kmemleak_free_part_phys(base, size);
    	return memblock_remove_range(&memblock.reserved, base, size);
    }

**源码路径 :** linux-4.12\\mm\\memblock.c#710

## 二、memblock\_remove\_range 函数分析

* * *

参考 [【Linux 内核 内存管理】memblock 分配器编程接口 ③ ( memblock\_remove 函数 | memblock\_remove\_range 函数 ) 二、memblock\_remove\_range 函数分析](https://cloud.tencent.com/developer/tools/blog-entry?target=https%3A%2F%2Fblog.csdn.net%2Fshulianghan%2Farticle%2Fdetails%2F124293877%23memblock_remove_range__64&source=article&objectId=2253534) 博客章节 , 在该博客章节中 , 详细地分析了该函数的执行流程 , 源码 ;

## 参考

[【Linux 内核 内存管理】memblock 分配器编程接口 ⑤ ( memblock_free 函数 | memblock_remove_range 函数 )-腾讯云开发者社区-腾讯云 (tencent.com)](https://cloud.tencent.com/developer/article/2253534)