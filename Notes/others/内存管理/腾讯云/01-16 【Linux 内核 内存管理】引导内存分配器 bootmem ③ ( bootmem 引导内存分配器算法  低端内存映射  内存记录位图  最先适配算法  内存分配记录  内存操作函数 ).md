【Linux 内核 内存管理】引导内存分配器 bootmem ③ ( bootmem 引导内存分配器算法 | 低端内存映射 | 内存记录位图 | 最先适配算法 | 内存分配记录 | 内存操作函数 )

#### 文章目录

-   [一、bootmem 引导内存分配器算法](https://cloud.tencent.com/developer?from_column=20421&from=20421)
-   -   [1、低端内存映射](https://cloud.tencent.com/developer?from_column=20421&from=20421)
    -   [2、内存记录位图](https://cloud.tencent.com/developer?from_column=20421&from=20421)
    -   [3、最先适配算法](https://cloud.tencent.com/developer?from_column=20421&from=20421)
    -   [4、内存分配记录](https://cloud.tencent.com/developer?from_column=20421&from=20421)
-   [二、bootmem 引导内存分配器 内存操作 函数 ( alloc\_bootmem | free\_bootmem )](https://cloud.tencent.com/developer?from_column=20421&from=20421)

## 一、bootmem 引导内存分配器算法

* * *

**bootmem 引导内存分配器算法 ;**

### 1、低端内存映射

**低端****内存映射** **:** 内核启动过程中 , 将 " 低端内存 " 交给 " 引导内存分配器 " 管理 ,

低端内存 可以 直接映射到 内核虚拟地址空间 对应的 物理内存 ;

### 2、内存记录位图

**内存记录位图 :** 引导内存分配器 中 , 使用 " 位图 " 记录 物理页 的分配情况 ,

如果物理页 **分配** , 在 位图中物理页对应的为 置

11

;

如果物理页 **回收** , 在 位图中物理页对应的为 置

00

;

### 3、最先适配算法

**最先适配算法 :** 分配内存时 , 扫描 " 位图 " , 找到 满足 内存需求大小 的 第一块 空闲的内存块 ;

### 4、内存分配记录

**内存分配记录 :** 为了有效利用内存 , " 引导内存分配器 " 支持小于

11

页的内存块分配 ,

`bootmem_data` 结构体中

-   `last_end_off` 成员 记录 上一次分配 内存块 的结束位置 +1 地址 , 也就是 分配内存块 结束位置 后面一个字节 , 下一个将要开始分配内存的位置 ;
-   `hint_idx` 成员 表示 上一次分配 内存块 的结束位置 后面的 物理页位置 索引 , 下次分配优先分配该索引 物理页 ;

在下一次分配内存时 , 如果 上次内存分配的物理页 的剩余空间 小于等于 要分配的内存 , 那么 直接在该 物理页 上分配内存 ;

## 二、bootmem 引导内存分配器 内存操作 函数 ( alloc\_bootmem | free\_bootmem )

* * *

**" bootmem 引导内存分配器 " 对外提供的 内存操作 函数如下 :**

**内存分配函数 :** `alloc_bootmem`

**内存释放函数 :** `free_bootmem`

**注意 : 在 ARM64 架构中 , 没有使用 引导内存分配器 ;**