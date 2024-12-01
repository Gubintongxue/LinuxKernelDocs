# linux 3.4.10 内核内存管理源代码分析3:伙伴系统内存分配

### 3伙伴系统内存分配

         伙伴系统通过指定分配阶order和分配标记位gfp\_mask来进行内存分配。分配阶是分配指的是分配2^order个页面。分配标志位是一些位的组合。这些标志位定义在include/[linux](https://so.csdn.net/so/search?q=linux&spm=1001.2101.3001.7020)/gfp.h中，迁移类型和水位线是在分配标志位中指定的。下面是几个分配标志位的说明：

最常用的GFP\_KERNEL,他表示内存分配（最终总是调用get\_free\_pages来实现实际的分配，这就是GFP前缀的由来）是代表运行在内核空间的进程执行的。使用GFP\_KERNEL容许表示在内存分配过程中可以等待。因此这时分配函数必须是可重入的，如果在进程上下文之外如：中断处理程序、tasklet以及内核定时器中这种情况下current进程不该睡眠，驱动程序该使用GFP\_ATOMIC.

GFP\_ATOMIC

用来从中断处理和进程上下文之外的其他代码中分配内存. 从不睡眠.

GFP\_KERNEL

内核内存的正常分配. 可能睡眠.

GFP\_USER

用来为用户空间页来分配内存; 它可能睡眠.

GFP\_HIGHUSER

如同 GFP\_USER, 但是从高端内存分配, 如果有. 高端内存在下一个子节描述.

GFP\_NOIO

GFP\_NOFS

这个标志功能如同 GFP\_KERNEL, 但是它们增加限制到内核能做的来满足请求. 一个 GFP\_NOFS 分配不允许进行任何文件系统调用, 而 GFP\_NOIO 根本不允许任何 I/O 初始化. 它们主要地用在文件系统和虚拟内存代码, 那里允许一个分配睡眠, 但是递归的文件系统调用会是一个坏注意.

上面列出的这些分配标志可以是下列标志的相或来作为参数, 这些标志改变这些分配如何进行:

\_\_GFP\_DMA

这个标志要求分配在能够 DMA 的内存区. 确切的含义是平台依赖的并且在下面章节来解释.

\_\_GFP\_HIGHMEM

这个标志指示分配的内存可以位于高端内存.

\_\_GFP\_COLD

正常地, 内存分配器尽力返回"缓冲热"的页 -- 可能在处理器缓冲中找到的页. 相反, 这个标志请求一个"冷"页, 它在一段时间没被使用. 它对分配页作 DMA 读是有用的, 此时在处理器缓冲中出现是无用的. 一个完整的对如何分配 DMA 缓存的讨论看"直接内存存取"一节在第 1 章.

\_\_GFP\_NOWARN

这个很少用到的标志阻止内核来发出警告(使用 printk ), 当一个分配无法满足.

\_\_GFP\_HIGH

这个标志标识了一个高优先级请求, 它被允许来消耗甚至被内核保留给紧急状况的最后的内存页.

\_\_GFP\_REPEAT

\_\_GFP\_NOFAIL

\_\_GFP\_NORETRY

这些标志修改分配器如何动作, 当它有困难满足一个分配.\_\_GFP\_REPEAT 意思是" 更尽力些尝试" 通过重复尝试 -- 但是分配可能仍然失败. \_\_GFP\_NOFAIL 标志告诉分配器不要失败; 它尽最大努力来满足要求. 使用 \_\_GFP\_NOFAIL 是强烈不推荐的; 可能从不会有有效的理由在一个设备驱动中使用它. 最后, \_\_GFP\_NORETRY 告知分配器立即放弃如果得不到请求的内存.

我们将伙伴系统内存的分配分为为三个阶段，第一个阶段主要是根据传入参数和系统策略来获取区域列表和节点掩码。内存分配的第二阶段是从用节点掩码和最大区域类型作为条件来扫描区域列表里面满足条件的区域，在对区域做一些判断，通过判断在当前扫描的选择当前扫描的区域分配内存。最大分配类型的原因是：如在x86系统上，我们要分配用于dma用到内存，只能在ZONE\_DMA中分配，但一些普通应用程序申请的分配的内存也可以在ZONE\_DMA中分配，最高区域类型由分配者指定，包含在分配标志位中。第三阶段根据在第二阶段扫描到的区域中进行内存分配，分配方法是根据阶和迁移类型来找到对应的空闲链表，在空闲链表中摘除一项，返回这项管理的物理地址，这一阶段也可能会进行内存迁移和块的。

#### 第一阶段

         第一阶段的主要任务是确定区域列表和节点掩码。策略分为几部分，1：即哪些节点可用于分配（这是用nodemask\_t结构实现，对应位为1表示节点可用，空nodemask\_t指针表示所有节点可用），2：怎样从可用的节点里面选择一个节点来进行本次分配（如总是使用一个节点，交错使用每个节点），3：用节点上的哪个列表，4：以及在策略里面指定节点掩码。

##### Mempolicy结构

策略结构中include/linux/mempolicy.c头文件定义：

         structmempolicy {
    
         atomic\_trefcnt;                //引用计数
    
         unsignedshort mode;    /\* See MPOL\_\* above \*/ // 节点选择模式
    
         unsignedshort flags;       /\* See set\_mempolicy()MPOL\_F\_\* above \*/ //节点选择模式辅助标记位
    
         union{
    
                   short                 preferred\_node; /\* preferred \*/            //首选的节点
    
                   nodemask\_t     nodes;             /\*interleave/bind \*/                 //节点掩码，对应位为1表示相应节点可用。
    
                   /\*undefined for default \*/
    
         }v;
    
         union{
    
                   nodemask\_tcpuset\_mems\_allowed;     /\* relative tothese nodes \*/
    
                   nodemask\_tuser\_nodemask;          /\* nodemask passedby user \*/
    
         }w;

};      

         在分配标志为有一位\_\_GFP\_THISNODE用来表示在本地节点分配内存。并定义了一个枚举变量表示几种节点选择方法。
    
         enum{
    
         MPOL\_DEFAULT,
    
         MPOL\_PREFERRED,                   //使用分配策略结构的首选节点号
    
         MPOL\_BIND,            //绑定一个节点
    
         MPOL\_INTERLEAVE,                  //交错使用节点
    
         MPOL\_MAX,    /\* always last member of enum \*/          //策略数目

};

Linux伙伴系统的内存分配的入口函数是unsigned long \_\_get\_free\_pages(gfp\_t gfp\_mask,unsigned int order)，参数gfp\_mask是分配标记位，下后面分析中会逐渐分析到标志位的作用，order是分配阶，即分配2^order个页面, \_\_get\_free\_pages函数直接调用alloc\_pages进行分配，分配成功则调用page\_address把物理地址转换为逻辑地址。

##### \_\_get\_free\_pages函数

       在mm/page\_alloc.c里定义，代码如下

2482unsigned long \_\_get\_free\_pages(gfp\_t gfp\_mask, unsigned int order)

2483 {

2484         struct page \*page;

2485

2486         /\*

2487          \* \_\_get\_free\_pages() returns a 32-bitaddress, which cannot represent

2488          \* a highmem page

2489          \*/

2490         VM\_BUG\_ON((gfp\_mask &\_\_GFP\_HIGHMEM) != 0);

2491

2492         page = alloc\_pages(gfp\_mask, order);

2493         if (!page)

2494                 return 0;

2495         return (unsigned long)page\_address(page);

2496 }

page\_address函数就不分析了，下面分析alloc\_pages在include/linux/gfp.h里面定义

staticinline struct page \*

alloc\_pages(gfp\_tgfp\_mask, unsigned int order)

{

         return alloc\_pages\_current(gfp\_mask,order);

}

###### alloc\_pages\_current函数

内部也是直接对alloc\_pages\_current函数的调用，alloc\_pages\_current在mm/mempolicy.c里面定义，alloc\_pages\_current的实现代码如下：

1910 struct page \*alloc\_pages\_current(gfp\_t gfp, unsigned order)

1911 {

1912         struct mempolicy\*pol = current->mempolicy;

1913         struct page\*page;

1914         unsigned intcpuset\_mems\_cookie;

1915

1916         if (!pol ||in\_interrupt() || (gfp & \_\_GFP\_THISNODE))

1917                 pol =&default\_policy;

1918

1919 retry\_cpuset:

1920        cpuset\_mems\_cookie = get\_mems\_allowed();

1921

1922         /\*

1923          \* No referencecounting needed for current->mempolicy

1924          \* nor systemdefault\_policy

1925          \*/

1926         if (pol->mode== MPOL\_INTERLEAVE)

1927                 page =alloc\_page\_interleave(gfp, order, interleave\_nodes(pol));

1928         else

1929                 page =\_\_alloc\_pages\_nodemask(gfp, order,

1930                                policy\_zonelist(gfp, pol, numa\_node\_id()),

1931                                policy\_nodemask(gfp, pol));

1932

1933         if(unlikely(!put\_mems\_allowed(cpuset\_mems\_cookie) && !page))

1934                 gotoretry\_cpuset;

1935

1936         return page;

1937 }

1912行把策略初始化为本进程的策略，1916行进行判断，如果本进程没有设置内存策略，在中断上下文，或者分配标记指定在本节点分配，则使用系统默认策略。

1920行获取内存访问标记，在进行一次失败的分配后如果发现策略以及改变，则再尝试分配，因为策略改变了，再次分配可能会有不同的结果。

###### interleave\_nodes函数

1926行判断如果分配策略制定了交错模式，则调用interleave\_nodes函数求得节点号，再调用alloc\_page\_interleave进行内存分配，interleave\_nodes也是在mm/mempolicy.c里实现，代码如下：

1563 static unsigned interleave\_nodes(struct mempolicy \*policy)

1564 {

1565         unsigned nid,next;

1566         structtask\_struct \*me = current;

1567

1568         nid =me->il\_next;

1569         next =next\_node(nid, policy->v.nodes);

1570         if (next >=MAX\_NUMNODES)

1571                 next =first\_node(policy->v.nodes);

1572         if (next <MAX\_NUMNODES)

1573                me->il\_next = next;

1574         return nid;

1575 }

函数定义局部变量nid为本次分配的节点号，next为下次分配的节点号，从task\_struct结构的il\_next变量获取本次进行分配的节点号，并把下次要进行分配的节点号存入il\_next变量。

Mempolicy结构里面联合变量v的成员nodes保存了一个位图数组，位为1，表示相应节点可以，1569行在nodes结构里面的第nid位开始位查找第一个能用的位

1570行判断大于等于最大节点数，则调用first\_node函数，查找节点掩码第一能用位的索引。满足条件1571行把first\_node的返回值赋给next

1572行判断next小于最大节点数，把next存入task\_struct结构的il\_next变量，下次可以用这个节点进行分配

###### interleave\_nodes函数

interleave\_nodes函数的返回值做为alloc\_page\_interleave的参数，alloc\_page\_interleave函数也在mm/mempolicy.c里实现，代码如下：

1809 static struct page \*alloc\_page\_interleave(gfp\_t gfp, unsignedorder,

1810                                         unsigned nid)

1811 {

1812         struct zonelist\*zl;

1813         struct page\*page;

1814

1815         zl =node\_zonelist(nid, gfp);

1816         page =\_\_alloc\_pages(gfp, order, zl);

1817         if (page&& page\_zone(page) == zonelist\_zone(&zl->\_zonerefs\[0\]))

1818                inc\_zone\_page\_state(page, NUMA\_INTERLEAVE\_HIT);

1819         return page;

1820 }

1815行调用函数node\_zonelist获取zonelist结构指针，在节点数据结构里面包含一个zonelist数组，第零个zonelist里面所有的区域，第一个zonelist包含本地节点的区域，在分配标记指定在本节点并且numa开启情况下node\_zonelist函数会返回节点的第一个zonelist结构地址，否则返回第零个zonelist结构地址。

在1816行把zonelist结构指针作为参数，调用\_\_alloc\_pages函数进行分配。\_\_alloc\_pages以空节点掩码调用\_\_alloc\_pages\_nodemask函数进行分配，\_\_alloc\_pages\_nodemask在内存分配第二阶段进行分析。

###### alloc\_pages\_current函数

现在回到alloc\_pages\_current函数：

1929                 page =\_\_alloc\_pages\_nodemask(gfp, order,

1930                                policy\_zonelist(gfp, pol, numa\_node\_id()),

1931                                 policy\_nodemask(gfp,pol));

1932

1933         if(unlikely(!put\_mems\_allowed(cpuset\_mems\_cookie) && !page))

1934                 gotoretry\_cpuset;

1935

1936         return page;

1937 }

###### policy\_zonelist函数

在1930行调用policy\_zonelist函数求得zonelist结构指针，并调用policy\_nodemask获取节点掩码。policy\_zonelist在mm/mempolicy.c里实现代码如下：

1537 static struct zonelist \*policy\_zonelist(gfp\_t gfp, structmempolicy \*policy,

1538         int nd)

1539 {

1540         switch(policy->mode) {

1541         case MPOL\_PREFERRED:

1542                 if(!(policy->flags & MPOL\_F\_LOCAL))

1543                        nd = policy->v.preferred\_node;

1544                 break;

1545         case MPOL\_BIND:

1546                 /\*

1547                  \*Normally, MPOL\_BIND allocations are node-local within the

1548                  \*allowed nodemask.  However, if\_\_GFP\_THISNODE is set and the

1549                  \*current node isn't part of the mask, we use the zonelist for

1550                  \* thefirst node in the mask instead.

1551                  \*/

1552                 if(unlikely(gfp & \_\_GFP\_THISNODE) &&

1553                                unlikely(!node\_isset(nd, policy->v.nodes)))

1554                        nd = first\_node(policy->v.nodes);

1555                 break;

1556         default:

1557                 BUG();

1558         }

1559         returnnode\_zonelist(nd, gfp);

1560 }

根据不同的策略模式，mempolicy结构的联合成员变量v有不同的意义，在首选模式下，v的成员preferred\_node保存了首选节点的id好，在其他模式下v的成员nodes是一个节点掩码，用来表示那些节点可用。

1541行当分配策略指定了首选模式时，并且分配选项没有指定本地选项，则把节设置为首选节点。

1545行当分配策略指定了绑定模式时，如果分配标记指定了在本地节点分配，并且传入的节点号在节点掩码里表示是不可用，则在节点掩码里面找到第一个能用的节点。

1559确定节点后用参数节点号和分配标志位，调用node\_zonelist函数获得zonelist结构指针，zonelist在上面已经分析过，这里就不分析了。

###### alloc\_pages\_current函数

回到alloc\_pages\_current函数，在1931行调用policy\_nodemask函数获取节点掩码。

###### policy\_nodemask函数

policy\_nodemask在mm/mempolicy.c里实现，代码如下：

1525 static nodemask\_t \*policy\_nodemask(gfp\_t gfp, struct mempolicy\*policy)

1526 {

1527         /\* Lower zonesdon't get a nodemask applied for MPOL\_BIND \*/

1528         if(unlikely(policy->mode == MPOL\_BIND) &&

1529                        gfp\_zone(gfp) >= policy\_zone &&

1530                        cpuset\_nodemask\_valid\_mems\_allowed(&policy->v.nodes))

1531                 return&policy->v.nodes;

1532

1533         return NULL;

1534 }

         在第一阶段实现了两个任务，求得zonelist结构指针和节点掩码指针。节点掩码的获取和区域的类型有关系，在系统里面定义有个变量policy\_zone，表示系统策略适用的最低的区域类型。还有cpuset里面有一个节点掩码结构。

1528行判断，如果系统策略是绑定模式，区域类型大于等于系统策略制定的最低区域，

并且cpuset\_nodemask\_valid\_mems\_allowed函数也返回真的时候，返回mempolicy结构里面的成员v的成员nodes的地址，否则返回空地址，返回空地址的情况下等价于节点每个为都为1。

       cpuset\_nodemask\_valid\_mems\_allowed函数
    
         cpuset\_nodemask\_valid\_mems\_allowed函数在kernel/cpuset.c里面实现代码如下：
    
         /\*\*

 \* cpuset\_nodemask\_valid\_mems\_allowed - checknodemask vs. curremt mems\_allowed

 \* @nodemask: the nodemask to be checked

 \*

 \* Are any of the nodes in the nodemask allowedin current->mems\_allowed?

 \*/

intcpuset\_nodemask\_valid\_mems\_allowed(nodemask\_t \*nodemask)

{

        return nodes\_intersects(\*nodemask,current->mems\_allowed);

}

         cpuset\_nodemask\_valid\_mems\_allowed的逻辑比较简单，在进程结构里面有个mems\_allowed成员，也是一个节点掩码，参数传进来一个节点掩码，如注释所说，这个函数判断了两个节点掩码是不是相交。
    
         在policy\_zonelist函数和policy\_nodemask函数返回后，alloc\_pages\_current函数在1929调用\_\_alloc\_pages\_nodemask进行分配，\_\_alloc\_pages\_nodemask函数中第二阶段进行分析。

#### 第二阶段

第二阶段的关键任务是确定区域。第一阶段已经取得了区域列表，和节点掩码，在第二阶段节点掩码和第一阶段有不同的作用，在第一阶段节点掩码用来表示节点是否可用，在第二阶段，节点掩码用来表示哪个区域可用。

###### \_\_alloc\_pages\_nodemask函数

第二阶段的入口函数是\_\_alloc\_pages\_nodemask，在page\_alloc.c里面实现，代码如下：

2417 struct page \*

2418 \_\_alloc\_pages\_nodemask(gfp\_t gfp\_mask, unsigned int order,

2419                        struct zonelist \*zonelist, nodemask\_t \*nodemask)

2420 {

2421         enum zone\_typehigh\_zoneidx = gfp\_zone(gfp\_mask);

2422         struct zone\*preferred\_zone;

2423         struct page \*page= NULL;

2424         int migratetype =allocflags\_to\_migratetype(gfp\_mask);

2425         unsigned intcpuset\_mems\_cookie;

2426

2427         gfp\_mask &=gfp\_allowed\_mask;

2428

2429        lockdep\_trace\_alloc(gfp\_mask);

2430

2431        might\_sleep\_if(gfp\_mask & \_\_GFP\_WAIT);

2432

2433         if(should\_fail\_alloc\_page(gfp\_mask, order))

2434                 returnNULL;

2435

2436         /\*

2437          \* Check thezones suitable for the gfp\_mask contain at least one

2438          \* valid zone.It's possible to have an empty zonelist as a result

2439          \* of GFP\_THISNODEand a memoryless node

2440          \*/

2441         if(unlikely(!zonelist->\_zonerefs->zone))

2442                 returnNULL;

2443

2444 retry\_cpuset:

2445        cpuset\_mems\_cookie = get\_mems\_allowed();

2446

2447         /\* The preferredzone is used for statistics later \*/

2448        first\_zones\_zonelist(zonelist, high\_zoneidx,

2449                                nodemask ? : &cpuset\_current\_mems\_allowed,

2450                                &preferred\_zone);

内存分配的时候允许指定最大的区域类型，如某些驱动分配的内存只能在DMA区域，这样可以在分配标记指定位ZONE\_DMA，2421行调用函数gfp\_zone求得的最大的区域类型。gfp\_zone实现比较简单，这里不作分析。

在分配标记里面也可以知道迁移类型，2424行调用allocflags\_to\_migratetype把分配标准转换为迁移类型。

在系统里面定义了一个全局变量gfp\_allowed\_mask，用来过滤一些分配标记选项，2427行实现这个功能。

2429行是调试代码，这里不作分析。

2431行也是调试代码，如果当前上下为允许睡眠，但分配选项指定了睡眠选项，给出警告信息。

2441行检查区域列表是否包含区域，如果第一个区域索引指向的区域是空，直接返回空。

2445行获取策略回环锁

###### first\_zones\_zonelist函数

2448行调用first\_zones\_zonelist函数，first\_zones\_zonelist函数在include/linux/mmzone.h定义，代码如下：

static inline struct zoneref \*first\_zones\_zonelist(struct zonelist\*zonelist,

                                               enum zone\_typehighest\_zoneidx,
    
                                               nodemask\_t\*nodes,
    
                                               structzone \*\*zone)

{

         returnnext\_zones\_zonelist(zonelist->\_zonerefs, highest\_zoneidx, nodes,
    
                                                                           zone);

}

first\_zones\_zonelist函数

first\_zones\_zonelist函数以zonelist的成员\_zonerefs，为参数直接调用next\_zones\_zonelist

next\_zones\_zonelist函数在mm/mmzone.c中实现，代码如下：

55 struct zoneref \*next\_zones\_zonelist(struct zoneref \*z,

 56                                         enumzone\_type highest\_zoneidx,

 57                                        nodemask\_t \*nodes,

 58                                         structzone \*\*zone)

 59 {

 60         /\*

 61          \* Find the next suitable zone to usefor the allocation.

 62          \* Only filter based on nodemask ifit's set

 63          \*/

 64         if (likely(nodes == NULL))

 65                 while (zonelist\_zone\_idx(z)> highest\_zoneidx)

 66                         z++;

 67         else

 68                 while (zonelist\_zone\_idx(z)> highest\_zoneidx ||

 69                                 (z->zone&& !zref\_in\_nodemask(z, nodes)))

 70                         z++;

 71

 72         \*zone = zonelist\_zone(z);

 73         return z;

 74 }

next\_zones\_zonelist函数的参数z是区域索引数组，highest\_zoneidx是最大的区域类型，nodes是节点掩码，zone用来传递找到的区域地址，此函数实现了在区域索引数组中满足区域类型小于等于highest\_zoneidx，并且在节点掩码里对应位是1的下一个索引数组。

代码并不复杂，分两种情况处理，在节点掩码地址为空的情况，第65行的一个循环找到满足，区域类型小于等于highest\_zoneidx的区域索引的地址，这等价与空节点掩码表示所有区域是有效的。

zref\_in\_nodemask函数

第二中情况是节点掩码不为空，只是在循环里面加了个，区域索引索引的区域有效和在节点掩码对应是有效的，判断区域对应的节点掩码有效的函数是zref\_in\_nodemask，代码也是在mm/mmzone.c中实现，代码如下：

45 static inline int zref\_in\_nodemask(struct zoneref \*zref,nodemask\_t \*nodes)

 46 {

 47 #ifdef CONFIG\_NUMA

 48         returnnode\_isset(zonelist\_node\_idx(zref), \*nodes);

 49 #else

 50         return 1;

 51 #endif /\* CONFIG\_NUMA \*/

 52 }

         在开启numa的情况下运行48行的代码，首选调用zonelist\_node\_idx获得区域索引指向的区域的id，zonelist\_node\_idx在include/linux/mmzone.h中实现，代码如下：
    
         static inline intzonelist\_node\_idx(struct zoneref \*zoneref)

{

#ifdefCONFIG\_NUMA

         /\* zone\_to\_nid not available in thiscontext \*/
    
         return zoneref->zone->node;

#else

         return 0;

#endif /\*CONFIG\_NUMA \*/

}

         直接从区域索引指向的区域的的成员node获取，每个区域的id和区域类型是相等的，区域id在每个节点中是唯一的。
    
         node\_isset是一个宏，定义如下：
    
         #define node\_isset(node, nodemask)test\_bit((node), (nodemask).bits)
    
         测试节点掩码对应为是否为1

###### \_\_alloc\_pages\_nodemask函数：

2451         if(!preferred\_zone)

2452                 goto out;

2453

2454         /\* Firstallocation attempt \*/

2455         page =get\_page\_from\_freelist(gfp\_mask|\_\_GFP\_HARDWALL, nodemask, order,

2456                        zonelist, high\_zoneidx, ALLOC\_WMARK\_LOW|ALLOC\_CPUSET,

2457                         preferred\_zone, migratetype);

2458         if(unlikely(!page))

2451行判断首选节点是否为空，如果为空调到标号out处。

因为在numa系统下，节点包含一个节点列表数组，数组有两项，第零个全局列表，包含系统所有区域，第一个包含本地区域。本地址列表的初始化顺序是ZONE\_DMA，ZONE\_NORMAL，ZONE\_HIGHMEM。全局列表的初始化是，对每个节点的初始化顺序和本地节点一样，对节点的初始化顺序是先初始化本节点，然后增加节点号，到底最大节点号后归零，包含每个节点的区域一次。

所以preferred\_zone实际返回的应该是包含本地节点，区域类型小于等于high\_zoneidx的第一区域

###### get\_page\_from\_freelist函数

2455行调用函数get\_page\_from\_freelist，get\_page\_from\_freelist在mm/page\_alloc.c中实现，代码如下：

1738 static struct page \*

1739 get\_page\_from\_freelist(gfp\_t gfp\_mask, nodemask\_t \*nodemask,unsigned int order,

1740                 structzonelist \*zonelist, int high\_zoneidx, int alloc\_flags,

1741                 structzone \*preferred\_zone, int migratetype)

1742 {

1743         struct zoneref\*z;

1744         struct page \*page= NULL;

1745         intclasszone\_idx;

1746         struct zone\*zone;

1747         nodemask\_t\*allowednodes = NULL;/\* zonelist\_cache approximation \*/

1748         int zlc\_active =0;             /\* set if usingzonelist\_cache \*/

1749         int did\_zlc\_setup= 0;          /\* just call zlc\_setup()one time \*/

1750

1751         classzone\_idx =zone\_idx(preferred\_zone);

1752 zonelist\_scan:

1753         /\*

1754          \* Scan zonelist,looking for a zone with enough free.

1755          \* See alsocpuset\_zone\_allowed() comment in kernel/cpuset.c.

1756          \*/

1757        for\_each\_zone\_zonelist\_nodemask(zone, z, zonelist,

1758                                                high\_zoneidx, nodemask) {

1759                 if(NUMA\_BUILD && zlc\_active &&

1760                        !zlc\_zone\_worth\_trying(zonelist, z, allowednodes))

1761                                continue;

1751行获取区域id，区域所在节点的区域数组的索引。

1757行用一个宏for\_each\_zone\_zonelist\_nodemask来实现遍历满足区域类型小于等于high\_zoneidx，并且在节点掩码nodemask中可用的在区域列表zonelist中的所有区域，zone用来传递目前扫描的区域，z用来传递目前扫描的节点索引。

for\_each\_zone\_zonelist\_nodemask宏定义如下：

#define for\_each\_zone\_zonelist\_nodemask(zone, z, zlist, highidx,nodemask) \\

         for (z =first\_zones\_zonelist(zlist, highidx, nodemask, &zone);      \\
    
                   zone;                                                                \\
    
                   z =next\_zones\_zonelist(++z, highidx, nodemask, &zone))    \\

循环开始找到调用first\_zones\_zonelist找到满足类型小于等于zoneidx，并且在节点掩码nodemask中可用的第一个区域，如果区域不空则继续，next\_zones\_zonelist查找类型小于等于zoneidx，并且在节点掩码nodemask中可用的下一个区域，first\_zones\_zonelist函数和next\_zones\_zonelist函数上面已经分析过，这里不再分析。

从1759行开始是对具体的区域做判断，如果满足掉进就在当前区域进行分配。

zlc\_zone\_worth\_trying函数

1759行判断如果开启了numa，并且zlc\_active置位，则调用zlc\_zone\_worth\_trying对区域内存是否充足做判断，lc\_zone\_worth\_trying在page\_alloc.c里面实现，代码如下

1660 static int zlc\_zone\_worth\_trying(struct zonelist \*zonelist,struct zoneref \*z,

1661                                                nodemask\_t \*allowednodes)

1662 {

1663         structzonelist\_cache \*zlc;     /\* cachedzonelist speedup info \*/

1664         int i;                          /\* index of \*z inzonelist zones \*/

1665         int n;                          /\* node that zone \*zis on \*/

1666

1667         zlc =zonelist->zlcache\_ptr;

1668         if (!zlc)

1669                 return 1;

1670

1671         i = z -zonelist->\_zonerefs;

1672         n =zlc->z\_to\_n\[i\];

1673

1674         /\* This zone isworth trying if it is allowed but not full \*/

1675         returnnode\_isset(n, \*allowednodes) && !test\_bit(i, zlc->fullzones);

1676 }

zonelist\_cache结构定义如下

struct zonelist\_cache {

         unsigned shortz\_to\_n\[MAX\_ZONES\_PER\_ZONELIST\];            /\*zone->nid \*/
    
         DECLARE\_BITMAP(fullzones,MAX\_ZONES\_PER\_ZONELIST);         /\* zonefull? \*/
    
         unsigned longlast\_full\_zap;             /\* when lastzap'd (jiffies) \*/

};

Fullzones是一个位数组，位为一表示相应区域内存充足。

short z\_to\_n用来从区域列表里面的区域索引数组的下标到区域id的转换的

1667行获得short z\_to\_n结构指针

1671行获得z的下标

1672行获得区域编号

1675行调用node\_isset测试区域中节点掩码中是否可用，test\_bit测试区域内存是否充足，在区域可用，内存不充足的情况下返回真，否则返回假。

###### get\_page\_from\_freelist函数

返回get\_page\_from\_freelist函数

1762                 if ((alloc\_flags &ALLOC\_CPUSET) &&

1763                        !cpuset\_zone\_allowed\_softwall(zone, gfp\_mask))

1764                                 continue;

1762行判断如果在分配标准位指定了ALLOC\_CPUSET，并且不允许在这个区域分配内存，则继续扫描下一个区域

继续分析get\_page\_from\_freelist函数

1791                 if ((alloc\_flags &ALLOC\_WMARK\_LOW) &&

1792                     (gfp\_mask &\_\_GFP\_WRITE) && !zone\_dirty\_ok(zone))

1793                         goto this\_zone\_full;

         1791行判断分配标志位是否设置了ALLOC\_WMARK\_LOW位
    
         1792行判断分配标志位是否设置了\_\_GFP\_WRITE，设置\_\_GFP\_WRITE标记位是向分配器提供信息，分配的页很可能会变脏，也就是会往这页写数据。这行可以平衡个区域脏页的数量。

1795                BUILD\_BUG\_ON(ALLOC\_NO\_WATERMARKS < NR\_WMARK);

1796                 if(!(alloc\_flags & ALLOC\_NO\_WATERMARKS)) {

1797                        unsigned long mark;

1798                        int ret;

1799

1800                        mark = zone->watermark\[alloc\_flags & ALLOC\_WMARK\_MASK\];

1801                        if (zone\_watermark\_ok(zone, order, mark,

1802                                    classzone\_idx, alloc\_flags))

1803                                goto try\_this\_zone;

1804

1805                        if (NUMA\_BUILD && !did\_zlc\_setup && nr\_online\_nodes >1) {

1806                                 /\*

1807                                 \* we do zlc\_setup if there are multiple nodes

1808                                 \* and before considering the first zone allowed

1809                                 \* by the cpuset.

1810                                  \*/

1811                                allowednodes = zlc\_setup(zonelist, alloc\_flags);

1812                                zlc\_active = 1;

1813                                did\_zlc\_setup = 1;

1814                         }

1815

1816                        if (zone\_reclaim\_mode == 0)

1817                                goto this\_zone\_full;

1818

1819                        /\*

1820                         \* As we may have just activated ZLC, check if the first

1821                          \* eligible zone has failedzone\_reclaim recently.

1822                         \*/

1823                        if (NUMA\_BUILD && zlc\_active &&

1824                                !zlc\_zone\_worth\_trying(zonelist, z, allowednodes))

1825                                 continue;

1826

1827                        ret = zone\_reclaim(zone, gfp\_mask, order);

1828                        switch (ret) {

1829                        case ZONE\_RECLAIM\_NOSCAN:

1830                                /\* did not scan \*/

1831                                continue;

1832                        case ZONE\_RECLAIM\_FULL:

1833                                /\* scanned but unreclaimable \*/

1834                                continue;

1795是调试代码

1796判断是否设置了不作水位判断的标记位

1800获取水位，在每个区域都包含若干水位线，用来表示区域的内存紧张程度，在分配标志位可以知道使用哪个水位线。

1801行调用zone\_watermark\_ok函数来判断区域的水位线是否满足，满足则在这个区域进行分配。

zone\_watermark\_ok函数

zone\_watermark\_ok在page\_alloc.c中实现，代码如下：

bool zone\_watermark\_ok(struct zone \*z, int order, unsigned longmark,

                         int classzone\_idx, int alloc\_flags)

{

         return\_\_zone\_watermark\_ok(z, order, mark, classzone\_idx, alloc\_flags,
    
                                               zone\_page\_state(z,NR\_FREE\_PAGES));

}

\_\_zone\_watermark\_ok函数

直接是对\_\_zone\_watermark\_ok的调用，\_\_zone\_watermark\_ok在page\_alloc.c中实现，代码如下：

1548 static bool \_\_zone\_watermark\_ok(struct zone \*z, int order,unsigned long mark,

1549                       intclasszone\_idx, int alloc\_flags, long free\_pages)

1550 {

1551         /\* free\_pages mygo negative - that's OK \*/

1552         long min = mark;

1553         int o;

1554

1555         free\_pages -= (1<< order) - 1;

1556         if (alloc\_flags& ALLOC\_HIGH)

1557                 min -=min / 2;

1558         if (alloc\_flags& ALLOC\_HARDER)

1559                 min -=min / 4;

1560

1561         if (free\_pages<= min + z->lowmem\_reserve\[classzone\_idx\])

1562                 returnfalse;

1563         for (o = 0; o< order; o++) {

1564                 /\* At thenext order, this order's pages become unavailable \*/

1565                 free\_pages-= z->free\_area\[o\].nr\_free << o;

1566

1567                 /\*Require fewer higher order pages to be free \*/

1568                 min>>= 1;

1569

1570                 if(free\_pages <= min)

1571                        return false;

1572         }

1573         return true;

1574 }

对一个区域水位判断的影响有分配标记，一个区域包含若干水位线，分配标准可以指定使用那条水位线。分配标记还可以知道分配的难度的标志ALLOC\_HARDER。高端内存的分配标志ALLOC\_HIGH也可以影响水位判断，毕竟有高端内存意味着较大的物理内存，还有“低端内存”，保留少些内存对系统的较小。

变量min代表最少应该保留的页数

1555行减去即将要分配的内存页数

1556，1557行如果分配的是高端内存，把水位减半

1558，1559行如果指定了高难度内存分配，再减去水位线的1/4

1561行如果最受应该保留的页数加上区域应该保留的页面数大于等于空闲页数，返回水位判断失败

1563-1572的循环减去阶小于oeder的页面，如果剩余页面数低于最少应该保留的页面数，返回水位判断失败。

1573返回水位判断成功。

###### get\_page\_from\_freelist函数

回到get\_page\_from\_freelist函数，

1805                        if (NUMA\_BUILD && !did\_zlc\_setup && nr\_online\_nodes >1) {

1806                                 /\*

1807                                 \* we do zlc\_setup if there are multiple nodes

1808                                 \* and before considering the first zone allowed

1809                                 \* by the cpuset.

1810                                 \*/

1811                                allowednodes = zlc\_setup(zonelist, alloc\_flags);

1812                                zlc\_active = 1;

1813                                did\_zlc\_setup = 1;

1814                         }

1805行判断如果开启了numa，did\_zlc\_setup配置没有启用，在线节点数大于1

1811行调用zlc\_setup设置区域缓存信息，1812行设置zlc\_active标志，1813行设置did\_zlc\_setup，这样在一次get\_page\_from\_freelist函数调用中只会调用一次zlc\_setup函数。在get\_page\_from\_freelist函数中可能会进行几次分配，但只会进入1806-1813的代码一次，这样保证了只会对区域进行一次内存充足检查。

###### zlc\_setup函数

zlc\_setup在page\_alloc.c中实现，代码如下：

1618 static nodemask\_t \*zlc\_setup(struct zonelist \*zonelist, intalloc\_flags)

1619 {

1620         structzonelist\_cache \*zlc;     /\* cachedzonelist speedup info \*/

1621         nodemask\_t\*allowednodes;       /\* zonelist\_cacheapproximation \*/

1622

1623         zlc =zonelist->zlcache\_ptr;

1624         if (!zlc)

1625                 returnNULL;

1626

1627         if(time\_after(jiffies, zlc->last\_full\_zap + HZ)) {

1628                bitmap\_zero(zlc->fullzones, MAX\_ZONES\_PER\_ZONELIST);

1629                zlc->last\_full\_zap = jiffies;

1630         }

1631

1632         allowednodes =!in\_interrupt() && (alloc\_flags & ALLOC\_CPUSET) ?

1633                                        &cpuset\_current\_mems\_allowed :

1634                                         &node\_states\[N\_HIGH\_MEMORY\];

1635         returnallowednodes;

1636 }

         zonelist\_cache有个成员last\_full\_zap，用来保存最后进行区域缓存信息设置的时间，1623行判断区域列表指向的区域缓存结构是否为空，为空返回空节点掩码。
    
         1627判断在现在是不是最近一次设置的一秒之后，避免频繁进行设置。
    
         1628行把zlc->fullzones全部清零，即表示每个区域内存都不充足。
    
         1629更新最后设置时间
    
         zlc\_setup函数还要返回一个节点掩码地址，这分两种情况，如果不再中断中而且分配选项指定了ALLOC\_CPUSET，则返回cpuset\_current\_mems\_allowed的地址，否则返回&node\_states\[N\_HIGH\_MEMORY\]，node\_states如下定义：
    
         nodemask\_tnode\_states\[NR\_NODE\_STATES\] \_\_read\_mostly = {
    
         \[N\_POSSIBLE\] =NODE\_MASK\_ALL,
    
         \[N\_ONLINE\] = { { \[0\] =1UL } },

#ifndef CONFIG\_NUMA

         \[N\_NORMAL\_MEMORY\] = {{ \[0\] = 1UL } },

#ifdef CONFIG\_HIGHMEM

         \[N\_HIGH\_MEMORY\] = { {\[0\] = 1UL } },

#endif

         \[N\_CPU\] = { { \[0\] =1UL } },

#endif       /\* NUMA \*/

};

         这样区域缓存设置通过两个方面影响了内存分配，一是内存内存是否充足信息，二是会改名节点掩码。

###### get\_page\_from\_freelist函数

回到get\_page\_from\_freelist函数

1816                        if (zone\_reclaim\_mode == 0)

1817                                goto this\_zone\_full;

1818

1819                        /\*

1820                         \* As we may have just activated ZLC, check if the first

1821                         \* eligible zone has failed zone\_reclaim recently.

1822                         \*/

1823                        if (NUMA\_BUILD && zlc\_active &&

1824                                !zlc\_zone\_worth\_trying(zonelist, z, allowednodes))

1825                                continue;

1826

1827                        ret = zone\_reclaim(zone, gfp\_mask, order);

1828                        switch (ret) {

1829                        case ZONE\_RECLAIM\_NOSCAN:

1830                                /\* did not scan \*/

1831                                continue;

1832                        case ZONE\_RECLAIM\_FULL:

1833                                /\* scanned but unreclaimable \*/

1834                                 continue;

1835                        default:

1836                                /\* did we reclaim enough \*/

1837                                if (!zone\_watermark\_ok(zone, order, mark,

1838                                                 classzone\_idx, alloc\_flags))

1839                                         gotothis\_zone\_full;

1840                         }

在系统中定义了一个变量zone\_reclaim\_mode，如果变量的值为假，则不进行内存回收，直接跳到标号try\_this\_zone进行内存分配。1816-1817实现了这个功能。

1823行进行了区域缓存信息的检查和1759行的代码是一样的，即判断区域是否内存充足

1827行是内存回收的代码，不同的情况会返回不同的结果，内存回收的代码中内存交换部分进行分析。1829行是没有进行内存的情况，1832行是没有回收到内存，如不不是这两种情况，表示已经回收了一些内存，在1837行再进行一次水位线判断，如果通过水位线判断，则在本区域进行分配。

下面是get\_page\_from\_freelist函数的最后一部分代码

1843 try\_this\_zone:

1844                 page =buffered\_rmqueue(preferred\_zone, zone, order,

1845                                                gfp\_mask, migratetype);

1846                 if (page)

1847                        break;

1848 this\_zone\_full:

1849                 if(NUMA\_BUILD)

1850                         zlc\_mark\_zone\_full(zonelist, z);

1851         }

1852

1853         if(unlikely(NUMA\_BUILD && page == NULL && zlc\_active)) {

1854                 /\*Disable zlc cache for second zonelist scan \*/

1855                zlc\_active = 0;

1856                 goto zonelist\_scan;

1857         }

1858         return page;

1859 }

1844行调用buffered\_rmqueue函数进行伙伴系统分配。1846-1847行如果分配到了内存则退出循环。

如果开始了numa选项，通过上面一些判断，总是把区域设置为内存充足，这是调用函数zlc\_mark\_zone\_full实现的。zlc\_mark\_zone\_full的代码比较简单，这里就不分析了。

1853-1857行的作用是：如果开启了numa选项，本次分配失败，并且本次分配实现了区域缓存信息，则关闭区域缓存信息，不在不进行内存是否充足的情况下再进行一次分配。

1858行返回分配结果。

#### 第三阶段

#####        buffered\_rmqueue函数

第三阶段的分析从buffered\_rmqueue开始，参数preferred\_zone是首选区域，一般是在本地节点上的区域，这里传过来主要用来做统计，一般如果preferred\_zone区域内存足够，会在preferred\_zone区域分配，因为preferred\_zone是第一个被扫描的区域。zone 是本次要进行分配的区域；order 是分配阶，gfp\_flags 是分配标志位；migratetype是迁移类型。

buffered\_rmqueue在page\_alloc.c里面实现，代码如下：

1386 static inline

1387 struct page \*buffered\_rmqueue(struct zone \*preferred\_zone,

1388                        struct zone \*zone, int order, gfp\_t gfp\_flags,

1389                         int migratetype)

1390 {

1391         unsigned longflags;

1392         struct page\*page;

1393         int cold =!!(gfp\_flags & \_\_GFP\_COLD);

1394

1395 again:

1396         if (likely(order== 0)) {

1397                 structper\_cpu\_pages \*pcp;

1398                 structlist\_head \*list;

1399

1400                local\_irq\_save(flags);

1401                 pcp =&this\_cpu\_ptr(zone->pageset)->pcp;

1402                 list =&pcp->lists\[migratetype\];

1403                 if (list\_empty(list)){

1404                        pcp->count += rmqueue\_bulk(zone, 0,

1405                                        pcp->batch, list,

1406                                        migratetype, cold);

1393行定义了变量cold用作冷热标记，热页的访问效率要高些，冷热标记的意思是该页的内存是否已经加载进cpu缓存，冷页很可能现在没有被加载进cpu缓存，热页现在很可能已经加载进cpu缓存，一般最近被访问的页加载进cpu缓存的可能性要大些。在每个cpu都有一个cpu内存页缓存结构，定义如下：

struct per\_cpu\_pages {

         int count;          列表包含的总页面数
    
         int high;             /\* high watermark, emptying needed\*/
    
         int batch;          但本cpu缓存分配完后，会从伙伴系统分配一定数量的页到cpu缓存链表里，这个变量代表每次分配的页面数。
    
         /\* Lists of pages, oneper migrate type stored on the pcp-lists \*/
    
         struct list\_headlists\[MIGRATE\_PCPTYPES\];   //列表数组，每种迁移类型一个链表。

};

1396-1417但分配阶为0，也就是分配一页内存的时候进入这段代码，这段代码的逻辑是，1400行关中断，1401行获得每cpu内存页缓存结构，1402行获得相应的迁移类型链表。1403如果链表为空，则运行1404行代码，调用rmqueue\_bulk函数从伙伴系统分配每cpu内存页缓存结构成员batch数据的页数到刚才获得的链表list。

###### rmqueue\_bulk函数

rmqueue\_bulk函数在page\_alloc.c里面实现，代码如下：

1069 static int rmqueue\_bulk(struct zone \*zone, unsigned int order,

1070                        unsigned long count, struct list\_head \*list,

1071                         int migratetype, intcold)

1072 {

1073         int i;

1074

1075        spin\_lock(&zone->lock);

1076         for (i = 0; i< count; ++i) {

1077                 structpage \*page = \_\_rmqueue(zone, order, migratetype);

1078                 if (unlikely(page == NULL))

1079                        break;

1080

1081                 /\*

1082                  \* Splitbuddy pages returned by expand() are received here

1083                  \* inphysical page order. The page is added to the callers and

1084                  \* listand the list head then moves forward. From the callers

1085                  \*perspective, the linked list is ordered by page number in

1086                  \* someconditions. This is useful for IO devices that can

1087                  \* mergeIO requests if the physical pages are ordered

1088                  \*properly.

1089                  \*/

1090                 if(likely(cold == 0))

1091                        list\_add(&page->lru, list);

1092                 else

1093                        list\_add\_tail(&page->lru, list);

1094                set\_page\_private(page, migratetype);

1095                 list =&page->lru;

1096         }

1097        \_\_mod\_zone\_page\_state(zone, NR\_FREE\_PAGES, -(i << order));

1098        spin\_unlock(&zone->lock);

1099         return i;

1100 }

整个代码的逻辑比较简单，1075获得区域保护链表的自旋锁

1076进入一个循环

1077调用\_\_rmqueue函数在分配页面，\_\_rmqueue函数中后面进行分析

1078-1079        如果分配失败，退出循环

1090-1091如冷标志为假，把分配到的页面加入到链表头。

1092-1093是冷标志为真的情况，把分配到的页面加入到链表尾。

1094设置页面迁移类型。

1095把链表头设置为本页面的lru链表。

1097设置空闲页面数据信息。

1098行释放自旋锁。

1099行返回获得的块数。

##### buffered\_rmqueue函数

1407                        if (unlikely(list\_empty(list)))

1408                                goto failed;

1409                 }

1410

1411                 if (cold)

1412                        page = list\_entry(list->prev, struct page, lru);

1413                 else

1414                        page = list\_entry(list->next, struct page, lru);

1415

1416                list\_del(&page->lru);

1417                pcp->count--;

1418         } else {

1411-1412行如果冷标志为真，则从链表头获得页面，否则1413-1414行长链表尾获得页面.

1416行把页面从lru链表摘除。

1417行减少pcp变量的块计数。

1419                 if(unlikely(gfp\_flags & \_\_GFP\_NOFAIL)) {

1420                        /\*

1421                         \* \_\_GFP\_NOFAIL is not to be used in new code.

1422                         \*

1423                          \* All \_\_GFP\_NOFAILcallers should be fixed so that they

1424                         \* properly detect and handle allocation failures.

1425                         \*

1426                         \* We most definitely don't want callers attempting to

1427                         \* allocate greater than order-1 page units with

1428                         \* \_\_GFP\_NOFAIL.

1429                         \*/

1430                        WARN\_ON\_ONCE(order > 1);

1431                 }

1432                spin\_lock\_irqsave(&zone->lock, flags);

1433                 page =\_\_rmqueue(zone, order, migratetype);

1434                spin\_unlock(&zone->lock);

1435                 if(!page)

1436                        goto failed;

1437                 \_\_mod\_zone\_page\_state(zone,NR\_FREE\_PAGES, -(1 << order));

1438         }

1419-1431是调试代码，这里不做分析

1432行获得区域的链表的自旋锁，1433行调用\_\_rmqueue函数进行分配，先分析完buffered\_rmqueue函数，在分析完buffered\_rmqueue函数后再分析\_\_rmqueue函数。

1437行因为刚才分配出去了2^order个页面，把空闲的页面计数减少2^order个，是调用\_\_mod\_zone\_page\_state函数实现的，\_\_mod\_zone\_page\_state代码比较简单，这里不分析了。

1440         \_\_count\_zone\_vm\_events(PGALLOC,zone, 1 << order);

1441        zone\_statistics(preferred\_zone, zone, gfp\_flags);

1442        local\_irq\_restore(flags);

1443

1444        VM\_BUG\_ON(bad\_range(zone, page));

1440行增加分配事件计数，这里会记录每个cpu上总共分配的页面数

1441调用zone\_statistics函数记录统计信息，在zone\_statistics函数里主要记录了，在首选区域分配内存次数（NUMA\_HIT），不是在首选区域分配内存次数（NUMA\_MISS），在外部区域分配内存次数（NUMA\_FOREIGN），在本地节点分配内存次数(NUMA\_LOCAL),在其他节点分配次数（NUMA\_OTHER），zone\_statistics函数代码比较简单，这里不做分析。

1442行释放自旋锁。

1444行VM\_BUG\_ON是一个调试宏，bad\_range对页面进行检查。

###### bad\_range函数

bad\_range在page\_alloc.c里面实现，代码如下：

263 static int bad\_range(struct zone \*zone, struct page \*page)

 264 {

 265         if (page\_outside\_zone\_boundaries(zone,page))

 266                 return 1;

265行调用page\_outside\_zone\_boundaries函数。

###### page\_outside\_zone\_boundaries函数

page\_outside\_zone\_boundaries函数也在page\_alloc.c里面实现，代码如下：

234 static int page\_outside\_zone\_boundaries(struct zone \*zone,struct page \*page)

 235 {

 236         int ret = 0;

 237         unsigned seq;

 238         unsigned long pfn = page\_to\_pfn(page);

 239

 240         do {

 241                 seq =zone\_span\_seqbegin(zone);

 242                 if (pfn >=zone->zone\_start\_pfn + zone->spanned\_pages)

 243                         ret = 1;

 244                 else if (pfn <zone->zone\_start\_pfn)

 245                         ret = 1;

 246         } while (zone\_span\_seqretry(zone,seq));

 247

 248         return ret;

 249 }

         每个区域结构有个变量zone\_start\_pfn表示本区域的第一个页面的帧号，spanned\_pages表示区域页面的总数。238行获得页面的帧号，241获得区域顺序锁，顺序锁避免在检测期间，zone的数据被改变，被改名则需要总线做一次检测，242-245行检测页帧是不是在区域的范围内，不是把ret变量值为真，246行释放区域顺序锁，在获得顺序锁期间，如果顺序锁没有被写者获取，则zone\_span\_seqretry会返回0，表示读成功，否则表示在读取数据期间数据被写者重写了，要重新检测一次。

###### bad\_range函数

267         if(!page\_is\_consistent(zone, page))

 268                 return 1;

 269

 270         return 0;

 271 }

267行调用page\_is\_consistent函数判断页面是否容许的。不过是不允许的，返回1

270行返回1

###### page\_is\_consistent函数

page\_is\_consistent函数在page\_alloc.c里面实现，参数zone是区域，page是页面几个指针，代码如下：

251 static int page\_is\_consistent(struct zone \*zone, struct page\*page)

 252 {

 253         if(!pfn\_valid\_within(page\_to\_pfn(page)))

 254                 return 0;

 255         if (zone != page\_zone(page))

 256                 return 0;

 257

 258         return 1;

 259 }

253行判断页帧是不是合法的，在page\_outside\_zone\_boundaries函数是判断页帧在区域中，在里是分配页帧在整个系统是不是合法的，pfn\_valid\_within是一个宏，定义为，pfn\_valid，pfn\_valid也是一个宏，定义为

#define pfn\_valid(pfn)               ((pfn)>= ARCH\_PFN\_OFFSET && ((pfn) - ARCH\_PFN\_OFFSET) < max\_mapnr)

ARCH\_PFN\_OFFSET是一个平台的最小页面便宜，max\_mapnr是最大页面数。

255行调用page\_zone函数判断page所在的区域是不是zone。

page\_zone函数

         page\_zone函数在include/linux/mm.h中定义，代码如下：

static inline struct zone \*page\_zone(conststruct page \*page)

{

         return&NODE\_DATA(page\_to\_nid(page))->node\_zones\[page\_zonenum(page)\];

}

为了节省内存，在page结构的变量flags中用若干位来保存节点号，另外用若干位保存区域类型。这里调用page\_to\_nid函数获得节点号码，并用宏NODE\_DATA转换为节点结构，在节点结构里包含node\_zones变量是一个区域数组，再用page\_zonenum函数获得区域类型，这些函数都比较简单，这里不做分析。

##### buffered\_rmqueue函数

1445         if(prep\_new\_page(page, order, gfp\_flags))

1446                 goto again;

1445行调用prep\_new\_page函数为新分配的页做些早期准备工作，如果准备工作是否，则调到again标号出重新进行一次分配。

###### prep\_new\_page函数

prep\_new\_page函数在mm/page\_alloc.c中实现，代码如下：

817 static int prep\_new\_page(struct page \*page, int order, gfp\_tgfp\_flags)

 818 {

 819         int i;

 820

 821         for (i = 0; i < (1 << order);i++) {

 822                 struct page \*p = page + i;

 823                 if(unlikely(check\_new\_page(p)))

 824                         return 1;

 825         }

 826

821-825行一个循环，调用check\_new\_page函数对块中的每个页面进行检查，如果一个页面没有通过检查，则返回真，表示准备新页面出错。

###### check\_new\_page函数

check\_new\_page函数在mm/page\_alloc.c中实现，代码如下：

804 static inline int check\_new\_page(structpage \*page)

 805{

 806        if (unlikely(page\_mapcount(page) |

 807                 (page->mapping !=NULL)  |

 808                (atomic\_read(&page->\_count) != 0) |

 809                 (page->flags &PAGE\_FLAGS\_CHECK\_AT\_PREP) |

 810                (mem\_cgroup\_bad\_page\_check(page)))) {

 811                 bad\_page(page);

 812                 return 1;

 813         }

 814        return 0;

 815}

check\_new\_page函数对页面进行一些检查，确定页面是否被使用，如不被使用返回真值，表示检查的页面已经被使用。806行调用page\_mapcount函数计算页面映射计数，如果映射计数不为零表示页面还行还有使用则，807行判断页面的映射指针是否为空，不为空表示页面正作为匿名页面或文件缓存使用，808行读取引用计数，如果引用计算不为零，表示页面还被应用，809判断页面的标志位有没有被设置，810判断页面是不是被mem cgroup使用，如果以上条件的或成立，则调用bad\_page打印一些调试信息，并且重置映射计数。

###### prep\_new\_page函数

827        set\_page\_private(page, 0);

 828         set\_page\_refcounted(page);

 829

 830         arch\_alloc\_page(page, order);

 831         kernel\_map\_pages(page, 1 <<order, 1);

 832

 833         if (gfp\_flags & \_\_GFP\_ZERO)

 834                 prep\_zero\_page(page, order, gfp\_flags);

 835

 836         if (order && (gfp\_flags &\_\_GFP\_COMP))

 837                 prep\_compound\_page(page,order);

 838

 839         return 0;

 840 }

827行设置page页面的private成员为0

828行把页面的引用计算设置为1，刚分配的页面只有一个使用则

830行arch\_alloc\_page是个空函数，什么也不做

833-834行如果分配标记设置了\_\_GFP\_ZERO，把页面清零。

836-837行，如果设置了大页面标记（\_\_GFP\_COMP），并且分配阶不为0（因为大页面比较要用第零个页面之后的页面来保存一些数据，索引最少要保护两个页面，也就是分配阶不能为零。

###### prep\_compound\_page函数：

prep\_compound\_page函数在mm/page\_alloc.c中实现，代码如下：

343 void prep\_compound\_page(struct page \*page, unsigned long order)

 344 {

 345         int i;

 346         int nr\_pages = 1 << order;

 347

 348         set\_compound\_page\_dtor(page,free\_compound\_page);

 349         set\_compound\_order(page, order);

 350         \_\_SetPageHead(page);

 351         for (i = 1; i < nr\_pages; i++) {

 352                 struct page \*p = page + i;

 353                 \_\_SetPageTail(p);

 354                 set\_page\_count(p, 0);

 355                 p->first\_page = page;

 356         }

 357 }

         大页面在第零个页面只后，也就是第一个页面的lru结构的next成员保存一个函数地址，在释放页面的时候会调用，在第一个页面的lru结构的prev结构保存了分配阶，在释放的时候用来获得阶。
    
         353行对第一个页面到最后一个页面设置PG\_head\_tail\_mask标记，表示是大页面页面，但不是大页面的第零个页面。
    
         354行设置引用计数为0
    
         355 行把大页面的后面页面的first\_page指针指向第零个页面。

##### buffered\_rmqueue函数

1447         return page;

1448

1449 failed:

1450        local\_irq\_restore(flags);

1451         return NULL;

1452 }

buffered\_rmqueue函数的最后一部分代码比较简单，返回分配结果，failed标记后开中断，返回空指针。

下面分析\_\_rmqueue函数。

##### \_\_rmqueue函数

1038 static struct page \*\_\_rmqueue(struct zone \*zone, unsigned intorder,

1039                                                int migratetype)

1040 {

1041         struct page\*page;

1042

1043 retry\_reserve:

1044         page =\_\_rmqueue\_smallest(zone, order, migratetype);

1045

1046         if(unlikely(!page) && migratetype != MIGRATE\_RESERVE) {

1047                 page =\_\_rmqueue\_fallback(zone, order, migratetype);

1048

1049                 /\*

1050                  \* UseMIGRATE\_RESERVE rather than fail an allocation. goto

1051                  \* isused because \_\_rmqueue\_smallest is an inline function

1052                  \* and wewant just one call site

1053                  \*/

1054                 if(!page) {

1055                        migratetype = MIGRATE\_RESERVE;

1056                        goto retry\_reserve;

1057                 }

1058         }

1059

1060        trace\_mm\_page\_alloc\_zone\_locked(page, order, migratetype);

1061         return page;

1062 }

\_\_rmqueue函数有两个主要执行流程，两个流程的主要区别是是分配的时候是否要进行内存迁移。1044行调用\_\_rmqueue\_smallest函数进行分配，\_\_rmqueue\_smallest函数分配的时候不进行内存迁移。

###### \_\_rmqueue\_smallest函数

         \_\_rmqueue\_smallest函数在mm/page\_alloc.c中实现，代码如下：

846 static inline

 847 struct page\*\_\_rmqueue\_smallest(struct zone \*zone, unsigned int order,

 848                                                int migratetype)

 849 {

 850         unsigned int current\_order;

 851         struct free\_area \* area;

 852         struct page \*page;

 853

 854         /\* Find a page of the appropriate sizein the preferred list \*/

 855         for (current\_order = order;current\_order < MAX\_ORDER; ++current\_order) {

 856                 area =&(zone->free\_area\[current\_order\]);

 857                 if(list\_empty(&area->free\_list\[migratetype\]))

 858                         continue;

 859

 860                 page =list\_entry(area->free\_list\[migratetype\].next,

 861                                                        struct page, lru);

 862                 list\_del(&page->lru);

 863                 rmv\_page\_order(page);

 864                 area->nr\_free--;

 865                 expand(zone, page, order,current\_order, area, migratetype);

 866                 return page;

 867         }

 868

 869         return NULL;

 870 }

         855行从申请分配的阶开始直到系统最大的允许的阶，进行一个循环扫描。

每个区域包含一个free\_area成员是free\_area结构数组，856行以order为数组下标，获得一个free\_area结构指针。

free\_area结构包含一个成员free\_list，是一个链表数组，每种迁移类型对应一个链表，链表中的项是页面块。860行以迁移类型为下标，area结构的成员free\_list的链表中获得next项的page结构指针。

862行把获得的块从page摘除，863行清除分配阶。Page结构包含一个成员private，在块的第一个页，伙伴系统用来保存分配阶。rmv\_page\_order函数做了两项工作，把映射计数置位，在把分配阶设置为0，也就是吧page的成员private设置为0.

         因为分配了一块，864行对空闲块计数减1
    
         865行调用expand函数对分配到的块进行分割，在分析完\_\_rmqueue\_smallest函数后再对expand函数进行分析。
    
         866行后面的代码只是返回分配结果。

在上面我们提出来块的概念，所谓块是在一个区域中连续的若干页，包含2^order个页面。在每个系统里，order会有个最大值MAX\_ORDER -1。如果一个块A的阶是order（order<MAX\_ORDER -1），第一个页面的页帧为pfn，则另一个阶也是order，并且首页帧是(pfn^(1<<order))的块B是块A的伙伴。

伙伴分配系统分配的时候并不一定会刚好分配到我们所需要的页的阶大小，可能会分配到比我们所需要的阶大的块，这时分配到的页面是我们所申请分配大小的2^n（n是自然数）次大小。这时候我们需要把页展开，所谓展开把一个块分成两个伙伴块，把其中的一块归还到相应的空闲链表，如果剩余的一块已经是我们申请分配的大小，则展开已经完成，否则循环进行这样过程。这个循环总会结束，因为每次循环剩余块的大小就会减半。

展开这个过程是由expand函数完成的。

##### expand函数

         expand函数在mm/page\_alloc.c中实现，代码如下：
    
         767static inline void expand(struct zone \*zone, struct page \*page,

 768        int low, int high, struct free\_area \*area,

 769        int migratetype)

 770{

 771        unsigned long size = 1 << high;

 772

 773        while (high > low) {

 774                 area--;

 775                 high--;

 776                 size >>= 1;

 777                 VM\_BUG\_ON(bad\_range(zone,&page\[size\]));

 778

 779#ifdef CONFIG\_DEBUG\_PAGEALLOC

 780                 if (high <debug\_guardpage\_minorder()) {

 781                         /\*

 782                          \* Mark as guard pages(or page), that will allow to

 783                          \* merge back toallocator when buddy will be freed.

 784                          \* Corresponding pagetable entries will not be touched,

 785                          \* pages will stay notpresent in virtual address space

 786                          \*/

 787                        INIT\_LIST\_HEAD(&page\[size\].lru);

 788                        set\_page\_guard\_flag(&page\[size\]);

 789                        set\_page\_private(&page\[size\], high);

 790                         /\* Guard pages are notavailable for any usage \*/

 791                        \_\_mod\_zone\_page\_state(zone, NR\_FREE\_PAGES, -(1 << high));

 792                         continue;

 793                 }

 794#endif

 795                 list\_add(&page\[size\].lru,&area->free\_list\[migratetype\]);

 796                 area->nr\_free++;

 797                set\_page\_order(&page\[size\], high);

 798        }

 799}

         771行获得块包含的页面数。
    
         773行是一个循环，high是被展开的页面的阶，low是要展开成的页面的阶，high>low这个循环就继续。

area传进来时候是指向high的空闲区域结构指针，每次进入循环都代码吧一个块分成两个伙伴块，所以447行area要自减，775行high也要自减，776包含页面数也要右移一位，相当于除以2。777行bad\_range前面已经分析过。

779-794是调试代码，这里不分析。

795行，把分开的两个伙伴块帧帧大的一块归还到空闲区域中，796行增加空闲区域的空闲块计算，797行设置阶，每个块，阶是保存在第零个页的page结构的成员private中。

##### \_\_rmqueue\_fallback函数

         在一个区域（zone）中包含若干空闲区域（free\_area），每个空闲区域，包含若干迁移类型。每个迁移类型都有一个空闲链表，链表里面的每项都管理空闲块。伙伴系统内存分配的最终就是从一个空闲链表里面摘除一项并返回摘除项对应的物理内存地址给使用者。
    
         在分配的时候对一个迁移类型，我们并不总是在这个迁移类型的空闲链表里去分配内存，我们也可以到其他迁移类型去进行分配，除非一个迁移类型（MIGRATE\_RESERVE）不允许把内存分配给其他迁移类型。\_\_rmqueue\_fallback在分配内存的时候使用这个方法。
    
         \_\_rmqueue\_fallback函数中mm/page\_alloc.c中实现，代码如下：

965 staticinline struct page \*

 966 \_\_rmqueue\_fallback(struct zone \*zone, intorder, int start\_migratetype)

 967 {

 968        struct free\_area \* area;

 969        int current\_order;

 970        struct page \*page;

 971        int migratetype, i;

 972

 973        /\* Find the largest possible block of pages in the other list \*/

 974        for (current\_order = MAX\_ORDER-1; current\_order >= order;

 975                                                --current\_order) {

 976                 for (i = 0; i <MIGRATE\_TYPES - 1; i++) {

 977                         migratetype =fallbacks\[start\_migratetype\]\[i\];

 978

 979                         /\* MIGRATE\_RESERVEhandled later if necessary \*/

 980                         if (migratetype ==MIGRATE\_RESERVE)

 981                                 continue;

 982

 983                         area =&(zone->free\_area\[current\_order\]);

 984                         if(list\_empty(&area->free\_list\[migratetype\]))

 985                                 continue;

          986
    
         987                         page =list\_entry(area->free\_list\[migratetype\].next,
    
         988                                         structpage, lru);
    
         989                         area->nr\_free--;

974行的循环是从最大分配阶开始循环的，这是为了避免在新的迁移类型中引入碎片。为什么？现在假如A类型内存不足，向B类型求援，假设只从B中分配一块最适合的小块，OK，那么过会儿又请求分配A类型内存，又得向B类型求援，这样来来回回从B类型中一点一点的分配内存将会导致B类型的内存四处都散布碎片。

976行循环MIGRATE\_TYPES – 1次

在系统中定义了一个二维迁移数组，为每个迁移类型指定了从其他迁移类型去借内存的循环顺序。977行获得本次循环的迁移类型，就说这次循环从那个迁移类型去“借”内存。

如果迁移类型为MIGRATE\_RESERVE，这不能从这个迁移类型借内存。

983行获得空闲区域指针，984-985对判断空闲区域当前扫描的迁移类型链表是否为空。为空扫描继续一个循环。

987行依据空闲区和迁移类型的链表获得page结构地址。

987行自减空闲区域空闲块计数

990

 991                         /\*

 992                          \* If breaking a largeblock of pages, move all free

 993                          \* pages to thepreferred allocation list. If falling

 994                          \* back for areclaimable kernel allocation, be more

 995                          \* aggressive abouttaking ownership of free pages

 996                          \*/

 997                         if(unlikely(current\_order >= (pageblock\_order >> 1)) ||

 998                                        start\_migratetype == MIGRATE\_RECLAIMABLE ||

 999                                        page\_group\_by\_mobility\_disabled){

1000                                 unsigned longpages;

1001                                 pages =move\_freepages\_block(zone, page,

1002                                                                 start\_migratetype);

1003

1004                                 /\* Claim thewhole block if over half of it is free \*/

1005                                 if (pages>= (1 << (pageblock\_order-1)) ||

1006                                                 page\_group\_by\_mobility\_disabled)

1007                                        set\_pageblock\_migratetype(page,

1008                                                                start\_migratetype);

1009

1010                                 migratetype =start\_migratetype;

1011                         }

1012

1013                         /\* Remove the pagefrom the freelists \*/

1014                        list\_del(&page->lru);

1015                         rmv\_page\_order(page);

1016

1017                         /\* Take ownership fororders >= pageblock\_order \*/

1018                         if (current\_order>= pageblock\_order)

1019                                change\_pageblock\_range(page, current\_order,

1020                                                         start\_migratetype);

1021

1022                         expand(zone, page,order, current\_order, area, migratetype);

1023

1024                        trace\_mm\_page\_alloc\_extfrag(page, order, current\_order,

1025                                 start\_migratetype, migratetype);

1026

1027                         return page;

1028                 }

1029         }

在搬迁工作中，如果分配到的是一个较大块，则意味着被借的迁移类型内存比较充足。另外有一种迁移类型（MIGRATE\_RECLAIMABLE），在别去其他迁移类型借的会多借一些过来。在系统中有关变量（page\_group\_by\_mobility\_disabled），当这个变量为真值时，没次进行迁移都会把迁移类型设置成MIGRATE\_RECLAIMABLE，也就是不可迁移。

997-999是判断分配的页够不够大，迁移的源类型是不是MIGRATE\_RECLAIMABLE，以及变量page\_group\_by\_mobility\_disabled的值。

1001行调用move\_freepages\_block函数进行空闲链表迁移。

###### move\_freepages\_block函数

move\_freepages\_block函数在mm/page\_alloc.c中实现，代码如下：

932 static int move\_freepages\_block(structzone \*zone, struct page \*page,

 933                                 intmigratetype)

 934{

 935        unsigned long start\_pfn, end\_pfn;

 936        struct page \*start\_page, \*end\_page;

 937

 938        start\_pfn = page\_to\_pfn(page);

 939        start\_pfn = start\_pfn & ~(pageblock\_nr\_pages-1);

 940        start\_page = pfn\_to\_page(start\_pfn);

 941        end\_page = start\_page + pageblock\_nr\_pages - 1;

 942        end\_pfn = start\_pfn + pageblock\_nr\_pages - 1;

 943

 944        /\* Do not cross zone boundaries \*/

 945        if (start\_pfn < zone->zone\_start\_pfn)

 946                 start\_page = page;

 947        if (end\_pfn >= zone->zone\_start\_pfn + zone->spanned\_pages)

 948                 return 0;

 949

 950        return move\_freepages(zone, start\_page, end\_page, migratetype);

 951}

         move\_freepages\_block函数对一个范围内的页面进行迁移。这样范围的第一个页面由参数page传进来，页面数由宏pageblock\_nr\_pages定义。pageblock\_nr\_pages的值分两种情况，在配置在大页选项的情况下pageblock\_nr\_pages等于一个大页包含的页面数，否则等于最大空闲页包含的页面数。
    
         这个函数只是求出第一个要迁移的页面page结构指针，最后一个要迁移的页面page结构指针，调用move\_freepages函数进行迁移。

######        move\_freepages函数

         move\_freepages把一个范围的页面迁移到一个区域的迁移类型空闲链表中。参数zone是要进行迁移到区域，迁移只能在一起区域内进行，不能把一个区域的页面迁移到另一个区域。start\_page 是要迁移到第一个页面结构地址，end\_page是最后一个要迁移到页面结构地址。migratetype 是要迁移到的迁移类型。move\_freepages在mm/page\_alloc.c中实现，代码如下：
    
         889static int move\_freepages(struct zone \*zone,

 890                           struct page\*start\_page, struct page \*end\_page,

 891                           int migratetype)

 892{

 893        struct page \*page;

 894         unsigned long order;

 895        int pages\_moved = 0;

 896

 897#ifndef CONFIG\_HOLES\_IN\_ZONE

 898        /\*

 899         \* page\_zone is not safe to call in this context when

 900         \* CONFIG\_HOLES\_IN\_ZONE is set. This bug check is probably redundant

 901         \* anyway as we check zone boundaries in move\_freepages\_block().

 902         \* Remove at a later date when no bug reports exist related to

 903         \* grouping pages by mobility

 904         \*/

 905        BUG\_ON(page\_zone(start\_page) != page\_zone(end\_page));

 906#endif

 907

 908        for (page = start\_page; page <= end\_page;) {

 909                 /\* Make sure we are notinadvertently changing nodes \*/

 910                 VM\_BUG\_ON(page\_to\_nid(page) !=zone\_to\_nid(zone));

 911

 912                 if(!pfn\_valid\_within(page\_to\_pfn(page))) {

 913                         page++;

 914                         continue;

 915                 }

 916

 917                 if (!PageBuddy(page)) {

 918                         page++;

 919                         continue;

 920                 }

 921

 922                 order = page\_order(page);

 923                 list\_move(&page->lru,

 924                          &zone->free\_area\[order\].free\_list\[migratetype\]);

 925                 page += 1 << order;

 926                 pages\_moved += 1 <<order;

 927        }

 928

 929        return pages\_moved;

 930}

         895行定义实际迁移的页面数量并初始化为0。
    
         908行从第一个页面开始进行循环，值到最后一个页面。
    
         910行对节点号进行检查。
    
         912行对页帧号进行检查。
    
         917行检查页面是不是buddy系统的页面。
    
         922行获得阶。
    
         923-924行进行空闲链表迁移。
    
         925行对增加page地址。
    
         926行对迁移页面数加1 << order
    
         929返回迁移的页面数量。

### 4 伙伴系统内存释放

####       free\_pages函数

伙伴系统释放页面的主函数是free\_pages，在mm/page\_alloc.c中实现，代码如下：

2517 void free\_pages(unsigned long addr,unsigned int order)

2518 {

2519        if (addr != 0) {

2520                VM\_BUG\_ON(!virt\_addr\_valid((void \*)addr));

2521                \_\_free\_pages(virt\_to\_page((void \*)addr), order);

2522        }

2523 }

         free\_pages函数只是在地址不为零的情况下简单调用virt\_to\_page函数从物理地址得到该地址的管理结构page的地址，然后调用\_\_free\_pages函数释放内存。

####       virt\_to\_page宏

         virt\_to\_page宏在arch/x86/include/asm/page.h中定义：
    
         #definevirt\_to\_page(kaddr)    pfn\_to\_page(\_\_pa(kaddr) >> PAGE\_SHIFT)
    
                  \_\_pa是一个宏，在arch/x86/include/asm/page.h中定义：
    
                  36 #define \_\_pa(x)         \_\_phys\_addr((unsigned long)(x))
    
                  \_\_phys\_addr是一个宏，在arch/x86/include/asm/page\_32.h中定义
    
                  16 #define \_\_phys\_addr(x)          \_\_phys\_addr\_nodebug (x)
    
                  \_\_phys\_addr\_nodebug也是一个宏，在arch/x86/include/asm/page\_32.h中定义
    
                  #define \_\_phys\_addr\_nodebug(x)   ((x) - PAGE\_OFFSET)
    
                  在x86上，PAGE\_OFFSET的值是0
    
                   一层一层拨开后\_\_pa(x)也就是返回x的值
    
                  pfn\_to\_page也是一个宏，在include/asm-generic/memory\_model.h中定义
    
                  30 #define \_\_pfn\_to\_page(pfn)      (mem\_map + ((pfn) - ARCH\_PFN\_OFFSET))
    
                  ARCH\_PFN\_OFFSET也是个宏，在include/asm-generic/memory\_model.h中定义
    
                  8 #ifndef ARCH\_PFN\_OFFSET
    
                9#define ARCH\_PFN\_OFFSET         (0UL)
    
                 10#endif
    
                  pfn\_to\_page只是在全局变量mem\_map加上页帧号获得该页帧的page管理结构。
    
         virt\_to\_page通过宏\_\_pa返回物理地址，物理地址右移PAGE\_SHIFT位后得到页帧，页帧号加上全局变量mem\_map就得到地址所对应的page管理结构地址。

#### \_\_free\_pages函数

         2505void \_\_free\_pages(struct page \*page, unsigned int order)

2506 {

2507        if (put\_page\_testzero(page)) {

2508                 if (order == 0)

2509                        free\_hot\_cold\_page(page, 0);

2510                 else

2511                         \_\_free\_pages\_ok(page,order);

2512        }

2513 }

         \_\_free\_pages函数2507行调用函数put\_page\_testzero在include/linux/mm.h中定义

275 static inline intput\_page\_testzero(struct page \*page)

 276{

 277        VM\_BUG\_ON(atomic\_read(&page->\_count) == 0);

 278        return atomic\_dec\_and\_test(&page->\_count);

 279}

         atomic\_dec\_and\_test函数对原子类型的变量原子地减1，并判断结果是否为0，如果为0，返回真，否则返回假。
    
         put\_page\_testzero函数的就是自减引用计算，并返回结构。

\_\_free\_pages函数2507行自减页面引用计数并返回当然自减后测试结果，如果自减后的引用计数为0，表面没有使用则，则分两种情况释放页面，阶为零的情况调用free\_hot\_cold\_page函数释放页面，否则调用\_\_free\_pages\_ok释放页面。         free\_hot\_cold\_page函数中伙伴系统的内存迁移一节一节分析过。

#### free\_pages\_prepare函数

         free\_pages\_prepare函数主要做释放内存的前期工作，在mm/page\_alloc.c中实现，代码如下：
    
         690static bool free\_pages\_prepare(struct page \*page, unsigned int order)

 691{

 692        int i;

 693        int bad = 0;

 694

 695        trace\_mm\_page\_free(page, order);

 696        kmemcheck\_free\_shadow(page, order);

 697

 698        if (PageAnon(page))

 699                 page->mapping = NULL;

 700        for (i = 0; i < (1 << order); i++)

 701                 bad += free\_pages\_check(page +i);

 702        if (bad)

 703                 return false;

 704

 705        if (!PageHighMem(page)) {

 706                debug\_check\_no\_locks\_freed(page\_address(page),PAGE\_SIZE<<order);

 707                debug\_check\_no\_obj\_freed(page\_address(page),

 708                                           PAGE\_SIZE << order);

 709        }

 710        arch\_free\_page(page, order);

 711        kernel\_map\_pages(page, 1 << order, 0);

 712

 713        return true;

 714}

         695行是调试代码，696行是关于kmemcheck内存调试的代码。
    
         701行调用free\_pages\_check函数对要释放的页面做检查，free\_pages\_check在前面已经分析过
    
         710行的arch\_free\_page函数是空函数
    
         如果不打开内存调试宏kernel\_map\_pages也是个空函数。

#### \_\_free\_pages\_ok函数

         \_\_free\_pages\_ok在mm/page\_alloc.c中实现，代码如下：

716 static void \_\_free\_pages\_ok(struct page\*page, unsigned int order)

 717{

 718        unsigned long flags;

 719        int wasMlocked = \_\_TestClearPageMlocked(page);

 720

 721        if (!free\_pages\_prepare(page, order))

 722                 return;

 723

 724        local\_irq\_save(flags);

 725        if (unlikely(wasMlocked))

 726                 free\_page\_mlock(page);

 727        \_\_count\_vm\_events(PGFREE, 1 << order);

 728        free\_one\_page(page\_zone(page), page, order,

 729                                        get\_pageblock\_migratetype(page));

 730        local\_irq\_restore(flags);

 731}

         \_\_free\_pages\_ok函数721行调用free\_pages\_prepare函数做释放前的准备工作
    
         724行关中断

725-726行并对mlock页调用free\_page\_mlock函数。

727行增加释放页面的统计计数。

728-729行调用free\_one\_page函数释放内存。获得迁移类型的函数get\_pageblock\_migratetype在伙伴系统的迁移类型一节已经分析过。

730恢复中断。

#### free\_one\_page函数

free\_one\_page函数在mm/page\_alloc.c中实现，代码如下：

         678static void free\_one\_page(struct zone \*zone, struct page \*page, int order,

 679                                 intmigratetype)

 680{

 681        spin\_lock(&zone->lock);

 682        zone->all\_unreclaimable = 0;

 683        zone->pages\_scanned = 0;

 684

 685        \_\_free\_one\_page(page, zone, order, migratetype);

 686        \_\_mod\_zone\_page\_state(zone, NR\_FREE\_PAGES, 1 << order);

 687        spin\_unlock(&zone->lock);

 688}

681获得回环锁

         682-683行是因为向伙伴系统释放了内存，通知内存交换系统这样区域已经可以进行回收和需要再次扫描。
    
         685行调用\_\_free\_one\_page释放函数，因为进入free\_one\_page函数的时候已经关中断，681行也获得了回旋锁，所以\_\_free\_one\_page是指原子上下文中执行，不会不中断，页不会被其他cpu打扰。
    
         686行增加区域的空闲页计数。
    
         687释放回旋锁。

#### \_\_free\_one\_page函数

         伙伴系统的内存释放工作是在\_\_free\_one\_page函数中完成的，进入\_\_free\_one\_page函数的时候已经关了中断，并且也已经获取了区域的自锁锁，这样保证\_\_free\_one\_page执行是原子的。
    
         \_\_free\_one\_page函数参数page是要释放的块的首页的page结构地址；zone是要把空闲块释放到的区域结构指针；order是要释放的块的阶，migratetype是要释放的块的迁移类型。\_\_free\_one\_page函数在mm/page\_alloc.c中实现，代码如下：

524 static inline void \_\_free\_one\_page(structpage \*page,

 525                 struct zone \*zone, unsignedint order,

 526                 int migratetype)

 527{

 528        unsigned long page\_idx;

 529        unsigned long combined\_idx;

 530        unsigned long uninitialized\_var(buddy\_idx);

 531        struct page \*buddy;

 532

 533        if (unlikely(PageCompound(page)))

 534                 if(unlikely(destroy\_compound\_page(page, order)))

 535                         return;

 536

 537        VM\_BUG\_ON(migratetype == -1);

 538

 539         page\_idx = page\_to\_pfn(page) & ((1<< MAX\_ORDER) - 1);

 540

 541        VM\_BUG\_ON(page\_idx & ((1 << order) - 1));

 542        VM\_BUG\_ON(bad\_range(zone, page));

 543

 544        while (order < MAX\_ORDER-1) {

 545                 buddy\_idx = \_\_find\_buddy\_index(page\_idx,order);

 546                 buddy = page + (buddy\_idx -page\_idx);

 547                 if (!page\_is\_buddy(page,buddy, order))

 548                         break;

 549                 /\*

 550                  \* Our buddy is free or it isCONFIG\_DEBUG\_PAGEALLOC guard page,

 551                  \* merge with it and move upone order.

 552                  \*/

 553                 if (page\_is\_guard(buddy)) {

 554                        clear\_page\_guard\_flag(buddy);

 555                        set\_page\_private(page, 0);

 556                        \_\_mod\_zone\_page\_state(zone, NR\_FREE\_PAGES, 1 << order);

 557                 } else {

 558                        list\_del(&buddy->lru);

 559                        zone->free\_area\[order\].nr\_free--;

 560                         rmv\_page\_order(buddy);

 561                 }

 562                 combined\_idx = buddy\_idx &page\_idx;

 563                 page = page + (combined\_idx -page\_idx);

 564                 page\_idx = combined\_idx;

 565                 order++;

 566        }

 567        set\_page\_order(page, order);

568

 569        /\*

 570         \* If this is not the largest possible page, check if the buddy

 571         \* of the next-highest order is free. If it is, it's possible

 572         \* that pages are being freed that will coalesce soon. In case,

 573         \* that is happening, add the free page to the tail of the list

 574         \* so it's less likely to be used soon and more likely to be merged

 575         \* as a higher order page

 576         \*/

 577        if ((order < MAX\_ORDER-2) &&pfn\_valid\_within(page\_to\_pfn(buddy))) {

 578                 struct page \*higher\_page,\*higher\_buddy;

 579                 combined\_idx = buddy\_idx &page\_idx;

 580                higher\_page = page +(combined\_idx - page\_idx);

 581                 buddy\_idx =\_\_find\_buddy\_index(combined\_idx, order + 1);

 582                 higher\_buddy = page +(buddy\_idx - combined\_idx);

 583                 if (page\_is\_buddy(higher\_page,higher\_buddy, order + 1)) {

 584                        list\_add\_tail(&page->lru,

 585                                &zone->free\_area\[order\].free\_list\[migratetype\]);

 586                         goto out;

 587                 }

 588        }

 589

 590         list\_add(&page->lru,&zone->free\_area\[order\].free\_list\[migratetype\]);

 591out:

 592        zone->free\_area\[order\].nr\_free++;

 593}

         对伙伴系统的每个空闲块，都有一个对应阶order，阶的最大值是MAX\_ORDER-1。对阶不是最大阶的块，都有一个伙伴块，伙伴块的阶也是order。如果把阶是最大值MAX\_ORDER-1，起始页帧号是2^ （MAX\_ORDER-1）的倍数的块叫做最大块。则伙伴块都在同一最大块中，也就是说伙伴块的页帧号大于MAX\_ORDER-1的位是相同的。在迁移类型一节中我们知道，同一最大块的迁移类型是相同的。
    
         伙伴系统内存释放就是要把伙伴块合并成一个更大的块，直到块已经是最大块或者当前要释放的块的伙伴块不是空闲块为止。这样可以防止内存碎片。
    
         对一个页的页帧号，我们把0到MAX\_ORDER-1位的部分叫内部页帧号，两个伙伴块的页帧号不同只在内存页帧号。
    
         局部变量page\_idx是当前要释放的块的内部页帧号，page是其首页的page管理结构地址。buddy\_idx是当前要释放的页的伙伴的页帧号，buddy是其首页的page管理结构地址。
    
         539行获得当前要释放块的首页内部帧号。
    
         545行获取伙伴块的首页帧号。一个阶为的order块的首页帧号和伙伴块的首页帧号只是在order为不同而已，\_\_find\_buddy\_index函数把当前页号的onder位做异或就得到了伙伴块的首页帧号。
    
         546获得伙伴块的首页的page结构地址。
    
         547-548判断两个块是不是伙伴块。不是则退出循环。
    
         553-557是用于调试的代码。
    
         558-560行把伙伴块从所在的空闲链表中摘除，减少所在的空闲区域的空闲块计数，并清除阶性息，因为合并后的首页可能不再是这一页。
    
         562获得是当前块和伙伴或的内部页帧号中较小的一个，也就是合并后的块的首页帧号。

563-564从新设置当前块首页的page管理结构地址和当前块的首页内部页帧号。

567保存阶，块的阶保存在块的首页的page管理结构的成员private中。

577-588如果合并后的当前块还不是最大块，则把当前块和当前块的伙伴块作为一个更大的块与上一阶阶的伙伴块尝试进行一次合并。如果合并是允许的，把当前块释放到空闲链表的末尾，使其更加晚的分配出去，因为分配的时候总是先分配链表头的块。因为这意味着如果当前块的伙伴块被释放的话，就可以合并成更大的块。使可能合成更大块的块更晚的分配出去，有防止内存碎片的作用。

590把合并后的块链接到空闲链表中。

592自增空闲区域空闲块计数。



## 参考

[linux 3.4.10 内核内存管理源代码分析3:伙伴系统内存分配-CSDN博客](https://blog.csdn.net/ancjf/article/details/8952012?spm=1001.2014.3001.5506)