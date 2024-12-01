# linux 3.4.10 内核内存管理源代码分析6:Slab分配器

### Slab分配器

         伙伴系统一次[内存](https://so.csdn.net/so/search?q=%E5%86%85%E5%AD%98&spm=1001.2101.3001.7020)分配最少都是一页，实际上很多时候我们分配内存的时候都只是一小块，不需要一页这样大的空间。这时候我们可以从slab来分配内存，Slab是伙伴系统之上的一个内存分配器，设计slab的目的是避免伙伴系统的一些缺陷。

内存分配的实质就是根据申请者申请的参数查找一块合适的内存，然后把内存地址返回。伙伴系统是最底层的内存管理系统，Slab是在伙伴系统之上的一个内存分配器，slab把从伙伴系统申请的内存块分割成更小的块来管理，分配的时候查找一个合适的块返回首地址。

slab从伙伴系统申请的内存块用[struct](https://so.csdn.net/so/search?q=struct&spm=1001.2101.3001.7020) slab描述，但Slab并不是直接基于struct slab来管理内存，因为还有几个需要考虑的因素：

一：在多cpu环境下，访问一个变量需要获得回环锁，如果对每个cpu缓存一些内存空间，这样可以提高性能。Slab采取的方法是对每个cpu缓存一些对象，如果cpu缓存的对象为空了则一次从获取一批对象缓存起来。为了这个因素，定义了结构struct array\_cache。

二：在numa系统上，每个节点上的访问速度可能不一样，分配时候需要选择合适节点来进行分配，对slab块也应该能按节点管理。针对这个因素定义了结构structkmem\_list3。

三：能同时对多个slab块进行管理，可以提高程序灵活性。在structkmem\_list3中设定了三个链表分别是块，闲，和半空slab块链表。

处于slab顶层的是slab缓存，slab缓存用struct kmem\_cache描述。上面的结构都在struct kmem\_cache中组织起来。每次内存分配都是在一个slab缓存中进行的，一个slab缓存中分配的对象的长度都是相同的。

下面是这些结构的定义：

struct slab定义如下：

struct slab {

         union {
    
                   struct {
    
                            struct list\_headlist;         //一个缓存中的所有slab块组成的链表
    
                            unsigned longcolouroff;           //着色区的大小
    
                            void \*s\_mem;           //指向对象区的起点
    
                            unsigned int inuse;  //Slab中所分配对象的个数
    
                            kmem\_bufctl\_t free;        //指向空闲对象链中的第一个对象
    
                            unsigned shortnodeid;    //slab块所在结点的结点号
    
                   };
    
                   struct slab\_rcu\_\_slab\_cover\_slab\_rcu;
    
         };

};

structarray\_cache定义如下：

structarray\_cache {

         unsigned int avail;             //当前缓存的对象数目
    
         unsigned int limit;             //最多可以缓存的对象数目
    
         unsigned int batchcount;         //缓存对象空的情况下一次获取的对象数
    
         unsigned int touched;      //用来表示这个对象最新是否被使用
    
         spinlock\_t lock;         //回环锁
    
         void \*entry\[\];            //指向缓存对象数组

};

structkmem\_list3定义如下：

structkmem\_list3 {

         struct list\_head slabs\_partial;                  //分配空的slab块链表
    
         struct list\_head slabs\_full;                //没有空闲对应的slab块链表
    
         struct list\_head slabs\_free;              //空闲的slab块链表
    
         unsigned long free\_objects;    //空闲对象数
    
         unsigned int free\_limit;            //空闲对象数上限
    
         unsigned int colour\_next;        //下一个颜色
    
         spinlock\_t list\_lock;
    
         struct array\_cache \*shared;   //共享的对象缓存
    
         struct array\_cache \*\*alien;    /\* on other nodes \*/
    
         unsigned long next\_reap;        //定义了内核在两次尝试收缩缓存之间，必须经过的时间间隔
    
         int free\_touched;              //表示三链表是否是活动的

};

structkmem\_cache定义如下：

structkmem\_cache {

/\* 1) Cachetunables. Protected by cache\_chain\_mutex \*/

         unsigned int batchcount;         //一批对象的数量
    
         unsigned int limit;             //空闲对象数上限
    
         unsigned int shared;        //共享对象数
    
         unsigned int buffer\_size;                   //对象实际长度
    
         u32 reciprocal\_buffer\_size;     //

/\* 2) touched byevery alloc & free from the backend \*/

         unsigned int flags;            //一些固定的标志位
    
         unsigned int num;             //每个slab块包含的对象数

/\* 3)cache\_grow/shrink \*/

         /\* order of pgs per slab (2^n) \*/
    
         unsigned int gfporder;     //slab块的阶
    
         /\* force GFP flags, e.g. GFP\_DMA \*/
    
         gfp\_t gfpflags;          //分配标志位
    
         size\_t colour;                     //颜色数
    
         unsigned int colour\_off;           //色差，相邻两种颜色的便宜值的差
    
         struct kmem\_cache \*slabp\_cache;                   //如果slab块的管理数据是独立的，则在这个缓存中分配空间
    
         unsigned int slab\_size;            
    
         unsigned int dflags;          /\* dynamic flags \*/
    
         /\* constructor func \*/
    
         void (\*ctor)(void \*obj);              //构造函数

/\* 4) cachecreation/removal \*/

         const char \*name;            //缓存名称，在整个系统中是唯一的
    
         struct list\_head next;      //所有的slab缓存用这个结果连接成一个链表

/\* 5) statistics\*/

#ifdefCONFIG\_DEBUG\_SLAB

         unsigned long num\_active;
    
         unsigned long num\_allocations;
    
         unsigned long high\_mark;
    
         unsigned long grown;
    
         unsigned long reaped;
    
         unsigned long errors;
    
         unsigned long max\_freeable;
    
         unsigned long node\_allocs;
    
         unsigned long node\_frees;
    
         unsigned long node\_overflow;
    
         atomic\_t allochit;
    
         atomic\_t allocmiss;
    
         atomic\_t freehit;
    
         atomic\_t freemiss;
    
         /\*
    
          \* If debugging is enabled, then the allocatorcan add additional
    
          \* fields and/or padding to every object.buffer\_size contains the total
    
          \* object size including these internal fields,the following two
    
          \* variables contain the offset to the userobject and its size.
    
          \*/
    
         int obj\_offset;
    
         int obj\_size;

#endif /\*CONFIG\_DEBUG\_SLAB \*/

/\* 6)per-cpu/per-node data, touched during every alloc/free \*/

         /\*
    
          \* We put array\[\] at the end of kmem\_cache,because we want to size
    
          \* this array to nr\_cpu\_ids slots instead ofNR\_CPUS
    
          \* (see kmem\_cache\_init())
    
          \* We still use \[NR\_CPUS\] and not \[1\] or \[0\]because cache\_cache
    
          \* is statically defined, so we reserve the maxnumber of cpus.
    
          \*/
    
         struct kmem\_list3 \*\*nodelists;                //指向三链表数组
    
         struct array\_cache \*array\[NR\_CPUS\];    //对每个cpu，指向每个cpu的对象缓存
    
         /\*
    
          \* Do not add fields after array\[\]
    
          \*/

};

对象：一块内存空间，用空间的首地址标识。

三链表：对应struct kmem\_list3，包含空，满，部分空三个链表。

对象缓存；缓存堆栈：对应struct array\_cache，用来缓存一些对象。

缓存；slab缓存：对应struct kmem\_cache。