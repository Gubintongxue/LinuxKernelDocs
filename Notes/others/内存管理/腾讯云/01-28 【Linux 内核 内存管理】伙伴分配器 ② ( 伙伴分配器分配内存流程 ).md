【Linux 内核 内存管理】伙伴分配器 ② ( 伙伴分配器分配内存流程 )

#### 文章目录

-   [一、伙伴分配器分配内存流程](https://cloud.tencent.com/developer?from_column=20421&from=20421)
-   -   [1、查询 n 阶页块](https://cloud.tencent.com/developer?from_column=20421&from=20421)
    -   [2、查询 n + 1 阶页块](https://cloud.tencent.com/developer?from_column=20421&from=20421)
    -   [3、查询 n + 2 阶页块](https://cloud.tencent.com/developer?from_column=20421&from=20421)

## 一、伙伴分配器分配内存流程

* * *

伙伴分配器 以 " 阶 " 为单位 , 分配 / 释放 物理页 ;

> **阶 ( Order ) :** 物理页 的 数量单位 ,

nn

阶页块 指的是

2n2^n

个 连续的 " 物理页 " ;

页 / 阶 概念参考 [【Linux 内核 内存管理】伙伴分配器 ① ( 伙伴分配器引入 | 页块、阶 | 伙伴 )](https://cloud.tencent.com/developer/tools/blog-entry?target=https%3A%2F%2Fhanshuliang.blog.csdn.net%2Farticle%2Fdetails%2F124304776%3Fspm%3D1001.2014.3001.5502&source=article&objectId=2253537) 博客 ;

**" 伙伴分配器 " 分配内存流程 :** 假设要 分配

nn

阶页块 ;

### 1、查询 n 阶页块

查询当前是否有 空闲的

nn

阶页块 ,

-   如果有则 直接分配 ,
-   如果没有 , 则进入下一步 , 查询

n+1n + 1

阶页块 ;

### 2、查询 n + 1 阶页块

查询当前是否有 空闲的

n+1n + 1

阶页块 ,

-   如果有 , 将

n+1n + 1

阶页块 分成

22

个

nn

阶页块 ,

-   一块插入 空闲

nn

阶页块链表 ;

-   一块 直接分配 ,

-   如果没有 , 则进入下一步 , 查询

n+2n + 2

阶页块 ;

### 3、查询 n + 2 阶页块

查询当前是否有 空闲的

n+2n + 2

阶页块 ,

-   如果有 , 将

n+2n + 2

阶页块 分成

22

个

n+1n + 1

阶页块 ,

-   一块插入 空闲

n+1n + 1

阶页块链表 ;

-   一块将

n+1n + 1

阶页块 分成

22

个

nn

阶页块 ,

-   一块插入 空闲

nn

阶页块链表 ;

-   一块 直接分配 ,

-   如果没有 , 则进入下一步 , 查询

n+3n + 3

阶页块 ;

## 参考

[【Linux 内核 内存管理】伙伴分配器 ② ( 伙伴分配器分配内存流程 )-腾讯云开发者社区-腾讯云 (tencent.com)](https://cloud.tencent.com/developer/article/2253537)