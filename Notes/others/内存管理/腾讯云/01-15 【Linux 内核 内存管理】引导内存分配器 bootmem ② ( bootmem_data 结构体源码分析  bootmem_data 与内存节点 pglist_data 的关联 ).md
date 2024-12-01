【Linux 内核 内存管理】引导内存分配器 bootmem ② ( bootmem_data 结构体源码分析 | bootmem_data 与内存节点 pglist_data 的关联 )

#### 文章目录

-   [一、bootmem\_data 结构体源码分析](https://cloud.tencent.com/developer?from_column=20421&from=20421)
-   
    -   [1、node\_min\_pfn 成员](https://cloud.tencent.com/developer?from_column=20421&from=20421)
    -   [2、node\_low\_pfn 成员](https://cloud.tencent.com/developer?from_column=20421&from=20421)
    -   [3、node\_bootmem\_map 成员](https://cloud.tencent.com/developer?from_column=20421&from=20421)
    -   [4、last\_end\_off 成员](https://cloud.tencent.com/developer?from_column=20421&from=20421)
    -   [5、hint\_idx成员](https://cloud.tencent.com/developer?from_column=20421&from=20421)
-   [二、引导内存分配器 bootmem\_data 与 内存节点 pglist\_data 的关联](https://cloud.tencent.com/developer?from_column=20421&from=20421)

在上一篇博客 [【Linux 内核 内存管理】引导内存分配器 bootmem ① ( 引导内存分配器 bootmem 工作机制 | 引导内存分配器 bootmem 的描述 bootmem\_data 结构体 )](https://cloud.tencent.com/developer/article/2253505?from_column=20421&from=20421) 引入了 " 引导内存分配器 bootmem " 其作用是在 Linux 内核启动阶段 , 进行内存管理 ;

引导内存分配器 使用 `bootmem_data` 结构体描述 ;

代码语言：javascript

复制

    /*
     * node_bootmem_map is a map pointer - the bits represent all physical 
     * memory pages (including holes) on the node.
     */
    typedef struct bootmem_data {
    	unsigned long node_min_pfn;
    	unsigned long node_low_pfn;
    	void *node_bootmem_map;
    	unsigned long last_end_off;
    	unsigned long hint_idx;
    	struct list_head list;
    } bootmem_data_t;

**源码路径 :** linux-4.12\\include\\linux\\bootmem.h#33

## 一、bootmem\_data 结构体源码分析

* * *

**bootmem\_data 结构体 成员分析 :**

### 1、node\_min\_pfn 成员

`node_min_pfn` 成员表示 起始的物理页 编号 ;

代码语言：javascript

复制

    	unsigned long node_min_pfn;

### 2、node\_low\_pfn 成员

`node_low_pfn` 成员表示 结束的物理页 编号 ;

代码语言：javascript

复制

    	unsigned long node_low_pfn;

### 3、node\_bootmem\_map 成员

`node_bootmem_map` 成员 是一个指针 , 指向一个位图 , 位图中的每一位都代表了一个物理页 ,

-   如果分配该物理页 , 则将该位 置

11

,

-   如果 回收该 物理页 , 则将该位 置

00

;

代码语言：javascript

复制

    	void *node_bootmem_map;

### 4、last\_end\_off 成员

`last_end_off` 成员 表示 上一次分配 内存块 的结束位置 +1 地址 ;

代码语言：javascript

复制

    	unsigned long last_end_off;

### 5、hint\_idx成员

`hint_idx` 成员 表示 上一次分配 内存块 的结束位置 后面的 物理页位置 索引 , 下次分配优先分配该索引 物理页 ;

代码语言：javascript

复制

    	unsigned long hint_idx;

## 二、引导内存分配器 bootmem\_data 与 内存节点 pglist\_data 的关联

* * *

在 内存节点 `pglist_data` 结构体中 , 有一个成员 , `struct bootmem_data *bdata;` , 该指针指向 引导内存分配器 `bootmem_data` 实例 ;

代码语言：javascript

复制

    typedef struct pglist_data {
    	...
    #ifndef CONFIG_NO_BOOTMEM
    	struct bootmem_data *bdata;
    #endif
    	...
    }

## 参考

[【Linux 内核 内存管理】引导内存分配器 bootmem ② ( bootmem_data 结构体源码分析 | bootmem_data 与内存节点 pglist_data 的关联 )-腾讯云开发者社区-腾讯云 (tencent.com)](https://cloud.tencent.com/developer/article/2253507)