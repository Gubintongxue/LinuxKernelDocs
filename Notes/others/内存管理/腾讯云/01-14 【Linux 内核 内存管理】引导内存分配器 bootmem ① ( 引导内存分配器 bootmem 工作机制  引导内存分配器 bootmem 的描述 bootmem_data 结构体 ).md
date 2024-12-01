【Linux 内核 内存管理】引导内存分配器 bootmem ① ( 引导内存分配器 bootmem 工作机制 | 引导内存分配器 bootmem 的描述 bootmem_data 结构体 )

#### 文章目录

-   [一、引导内存分配器 bootmem 简介](https://cloud.tencent.com/developer?from_column=20421&from=20421)
-   -   [1、引导内存分配器 bootmem 引入](https://cloud.tencent.com/developer?from_column=20421&from=20421)
    -   [2、引导内存分配器 bootmem 工作机制](https://cloud.tencent.com/developer?from_column=20421&from=20421)
-   [二、引导内存分配器 bootmem 描述 bootmem\_data 结构体](https://cloud.tencent.com/developer?from_column=20421&from=20421)

## 一、引导内存分配器 bootmem 简介

* * *

### 1、引导内存分配器 bootmem 引入

Linux 内核 初始化 时 , 需要进行内存分配 , 启动阶段的 内存分配 与 运行时的 内存分配 机制不同 ;

此时 Linux 内核 提供了一个 临时的 " 引导内存分配器 bootmem " , 该 内存分配器 只在启动过程中使用 , 启动完成后 , 就会被丢弃 ;

### 2、引导内存分配器 bootmem 工作机制

**" 引导内存分配器 bootmem " 工作机制如下 :**

Linux 内核初始化过程中 , 临时提供一个 " 引导内存分配器 bootmem " ,

引导内存分配器 bootmem 的主要作用是 初始化 " 页分配器 " 和 " 块分配器 " ,

将 空闲物理页 纳入到 " 页分配器 " 管理之下 ,

完成上述工作后 , 将 " 引导内存分配器 bootmem " 丢弃 ;

## 二、引导内存分配器 bootmem 描述 bootmem\_data 结构体

* * *

在 Linux 内核中 , 使用 `struct bootmem_data` 结构体 , 描述 " 引导内存分配器 bootmem " ;

`struct bootmem_data` 结构体 定义在 Linux 内核源码的 linux-4.12\\include\\linux\\bootmem.h#33 位置 , **源码如下 :**

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

## 参考

[【Linux 内核 内存管理】引导内存分配器 bootmem ① ( 引导内存分配器 bootmem 工作机制 | 引导内存分配器 bootmem 的描述 bootmem_data 结构体 )-腾讯云开发者社区-腾讯云 (tencent.com)](https://cloud.tencent.com/developer/article/2253505)