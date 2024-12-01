【Linux 内核 内存管理】物理分配页 ② ( __alloc_pages_nodemask 函数参数分析 | __alloc_pages_nodemask 函数分配物理页流程 )

#### 文章目录

-   [一、\_\_alloc\_pages\_nodemask 函数参数分析](https://cloud.tencent.com/developer?from_column=20421&from=20421)
-   [二、 \_\_alloc\_pages\_nodemask 函数分配物理页流程](https://cloud.tencent.com/developer?from_column=20421&from=20421)

## 一、\_\_alloc\_pages\_nodemask 函数参数分析

* * *

`__alloc_pages_nodemask` 函数 定义在 Linux 内核源码的 linux-4.12\\mm\\page\_alloc.c#4003 位置 , 函数原型如下 :

① `gfp_t gfp_mask` 参数 表示 物理页 " 分配标志位 " ;

② `unsigned int order` 参数 表示 物理页 " 阶数 " , " 阶 " 是 物理页 的 数量单位 ,

nn

阶页块 指的是

2n2^n

个 连续的 " 物理页 " ;

③ `struct zonelist *zonelist` 参数 表示 " 内存节点 “ 首选 ” 备用区域列表 " ;

④ `nodemask_t *nodemask` 参数 表示 可以分配物理页 的 " 内存节点 " , 如果没有要求 , 可以设置为 NULL ;

代码语言：javascript

复制

    /*
     * This is the 'heart' of the zoned buddy allocator.
     */
    struct page *
    __alloc_pages_nodemask(gfp_t gfp_mask, unsigned int order,
    			struct zonelist *zonelist, nodemask_t *nodemask)

**源码路径 :** linux-4.12\\mm\\page\_alloc.c#4003

## 二、 \_\_alloc\_pages\_nodemask 函数分配物理页流程

* * *

**\_\_alloc\_pages\_nodemask 函数分配物理页流程 :**

**首先** , 根据 `gfp_t gfp_mask` 分配标志位 参数 , 得到 " 内存节点 “ 的 首选 ” 区域类型 " 和 " 迁移类型 " ;

**然后** , 执行 " 快速路径 " , 第一次分配 尝试使用 低水线分配 ;

如果上述 " 快速路径 " 分配失败 , 则执行 " 慢速路径 " 分配 ;

## 参考

[【Linux 内核 内存管理】物理分配页 ② ( __alloc_pages_nodemask 函数参数分析 | __alloc_pages_nodemask 函数分配物理页流程 )-腾讯云开发者社区-腾讯云 (tencent.com)](https://cloud.tencent.com/developer/article/2253551)