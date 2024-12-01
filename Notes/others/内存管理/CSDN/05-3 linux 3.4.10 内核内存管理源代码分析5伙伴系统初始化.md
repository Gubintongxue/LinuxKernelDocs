# linux 3.4.10 内核内存管理源代码分析5:伙伴系统初始化

### 5 伙伴系统初始化

         计算机在启动时都是先加电,然后进行硬件检测并加载引导程序。

引导程序把[Linux](https://so.csdn.net/so/search?q=Linux&spm=1001.2101.3001.7020)系统内核装载到内存，加载内核后引导程序跳转到

arch/x86/boot/compressed/head\_32.S的startup\_32标号处执行。

         在arch/x86/boot/compressed/head\_32.S中会调用arch/x86/boot/main.c中的main函数。
    
         main函数执行完后会跳转到arch/x86/kernel/head\_32.S的标号startup\_32处执行。
    
         在arch/x86/kernel/head\_32.S中会调用arch/x86/kernel/head32.c中的i386\_start\_kernel
    
         i386\_start\_kernel调用init/main.c中的start\_kernel函数，start\_kernel是用来启动内核的主函数。
    
         start\_kernel函数会调用arch/x86/kernel/setup.c中的setup\_arch函数
    
         setup\_arch函数会调用arch/x86/mm/init\_32.c中的paging\_init函数
    
         初始化后释放初始化内存分配器的内存到伙伴系统的流程是：

start\_kernel ()at init/main.c:524

         mm\_init() at init/main.c:458
    
         mem\_init() at arch/x86/mm/init\_32.c:752
    
         free\_all\_bootmem() at mm/nobootmem.c:168
    
         free\_low\_memory\_core\_early() at mm/nobootmem.c:130
    
         \_\_free\_memory\_core() at mm/nobootmem.c:118
    
         \_\_free\_pages\_memory() at mm/nobootmem.c:99
    
         \_\_free\_pages\_bootmem() at mm/page\_alloc.c:749
    
         \_\_free\_pages() at mm/page\_alloc.c:2506
    
         最终由伙伴系统的内存释放函数\_\_free\_pages来吧初始化内存分配器的内存释放到伙伴系统。
    
         伙伴系统的初始化实质是从初始化内存分配器接管内存管理的权限。而伙伴初始化也分成两个步骤，第一步是伙伴系统各种结构和管理数据的初始化，第二个步骤是把初始化内存分配器中的空闲内存释放到伙伴系统，之后就可以正式使用伙伴系统分配内存了。第一个步骤关键由zone\_sizes\_init和build\_all\_zonelists函数完成。第二个步骤为的执行流程我们在上面已经列出，具体代码将在初始内存分配器的实现代码。
    
       我们知道在numa系统中，包含若干节点，而每个节点包含若干区域，每个区域包含若干空闲区域，每个空闲区域包含若干迁移类型，对每个迁移类型，都有一个空闲链表。空闲链表链接的是空闲块。
    
       在初始化过程中关键的是节点和区域的初始化，因为空闲区域的初始化只是对包含的空闲链表数组的每个链表初始化为空链表。并对空闲块计数初始化为0而已。
    
       节点初始化重要的部分是找到第一个可用的页的页帧，节点包含也页面数和节点的可用页面数，还有初始化节点中page结构数组。
    
       对区域的初始化的关键也查找也是第一个可用的页的页帧，以及区域页面数和可用页面数的初始化。实际节点的可用页面数就是节点的所有区域可用页面数的和，节点的页面数是节点的所有区域包含的页面数的和。

为了在后面的分析避免嵌套过深，下面先介绍一个函数，包括计算区域页面数和可用页面数的函数和计算节点范围的函数：

#### \===============

####       zone\_spanned\_pages\_in\_node函数

         zone\_spanned\_pages\_in\_node函数计算区域的的包含的页面数，包含中间可能存在的空洞。计算区域总页面数要考虑两个因数：

1:在系统中包含一个数组arch\_zone\_lowest\_possible\_pfn，保存了每种类型的区域可能的最小的页帧号，另外一个数组arch\_zone\_highest\_possible\_pfn，保存了每种类型的区域可能的最大的页帧号。

2:另外有一种区域类型是ZONE\_MOVABLE，这是系统为了防止内存碎片退出的一种区域类型，其他类型的区域不能包含ZONE\_MOVABLE区域的页面。一个节点中的区域是按顺序存放的，ZONE\_MOVABLE存放在节点的最高端。

zone\_spanned\_pages\_in\_node在mm/page\_alloc.c中实现代码如下：

4090 staticunsigned long \_\_meminit zone\_spanned\_pages\_in\_node(int nid,

4091                                        unsigned long zone\_type,

4092                                        unsigned long \*ignored)

4093 {

4094         unsigned long node\_start\_pfn,node\_end\_pfn;

4095         unsigned long zone\_start\_pfn,zone\_end\_pfn;

4096

4097         /\* Get the start and end of the nodeand zone \*/

4098         get\_pfn\_range\_for\_nid(nid,&node\_start\_pfn, &node\_end\_pfn);

4099         zone\_start\_pfn = arch\_zone\_lowest\_possible\_pfn\[zone\_type\];

4100         zone\_end\_pfn =arch\_zone\_highest\_possible\_pfn\[zone\_type\];

4101        adjust\_zone\_range\_for\_zone\_movable(nid, zone\_type,

4102                                node\_start\_pfn, node\_end\_pfn,

4103                                 &zone\_start\_pfn,&zone\_end\_pfn);

4104

4105         /\* Check that this node has pageswithin the zone's required range \*/

4106         if (zone\_end\_pfn < node\_start\_pfn|| zone\_start\_pfn > node\_end\_pfn)

4107                 return 0;

4108

4109         /\* Move the zone boundaries inside thenode if necessary \*/

4110         zone\_end\_pfn = min(zone\_end\_pfn,node\_end\_pfn);

4111         zone\_start\_pfn = max(zone\_start\_pfn,node\_start\_pfn);

4112

4113         /\* Return the spanned pages \*/

4114         return zone\_end\_pfn - zone\_start\_pfn;

4115 }

4098行调用get\_pfn\_range\_for\_nid函数遍历初始化内存分配器的每个空闲段，取得最小的空闲页帧和最大的空闲页帧。

4098-4099行获得系统允许的区域最大页帧和最小页帧。

在区域中可能于ZONE\_MOVABLE类型区域有重合，4101行调用adjust\_zone\_range\_for\_zone\_movable函数去掉与区域ZONE\_MOVABLE类型重合的部分。

4106-4107行如果区域不在所在的节点的页范围内，返回0.

4110-4111行区域的页面范围只能在所在节点的范围内。

#### adjust\_zone\_range\_for\_zone\_movable函数

         adjust\_zone\_range\_for\_zone\_movable函数是用来保留ZONE\_MOVABLE类型区域的页面的。在mm/page\_alloc.c中实现代码如下：
    
         4060static void \_\_meminit adjust\_zone\_range\_for\_zone\_movable(int nid,

4061                                        unsigned long zone\_type,

4062                                        unsigned long node\_start\_pfn,

4063                                        unsigned long node\_end\_pfn,

4064                                         unsigned long \*zone\_start\_pfn,

4065                                        unsigned long \*zone\_end\_pfn)

4066 {

4067        /\* Only adjust if ZONE\_MOVABLE is on this node \*/

4068        if (zone\_movable\_pfn\[nid\]) {

4069                 /\* Size ZONE\_MOVABLE \*/

4070                 if (zone\_type == ZONE\_MOVABLE){

4071                         \*zone\_start\_pfn =zone\_movable\_pfn\[nid\];

4072                         \*zone\_end\_pfn =min(node\_end\_pfn,

4073                                arch\_zone\_highest\_possible\_pfn\[movable\_zone\]);

4074

4075                 /\* Adjust for ZONE\_MOVABLEstarting within this range \*/

4076                 } else if (\*zone\_start\_pfn< zone\_movable\_pfn\[nid\] &&

4077                                 \*zone\_end\_pfn> zone\_movable\_pfn\[nid\]) {

4078                         \*zone\_end\_pfn =zone\_movable\_pfn\[nid\];

4079

4080                 /\* Check if this whole rangeis within ZONE\_MOVABLE \*/

4081                 } else if (\*zone\_start\_pfn>= zone\_movable\_pfn\[nid\])

4082                         \*zone\_start\_pfn = \*zone\_end\_pfn;

4083        }

4084 }

         4068行只有在zone\_movable\_pfn\[nid\]数组中的先不为0，才考虑ZONE\_MOVABLE类型区域。
    
         4070-4073行处理的是zone\_typ等于ZONE\_MOVABLE的情况。ZONE\_MOVABLE区域范围的求法是：在系统中有个zone\_movable\_pfn数组，以节点号为下标可以确定每个节点的ZONE\_MOVABLE类型区域的首页帧号，另外有个变量movable\_zone，用来保存一个区域类型，表示ZONE\_MOVABLE类型区域的最大页帧号和movable\_zone类型的最大页帧号相等。
    
         全局变量movable\_zone表示一个区域类型，表示ZONE\_MOVABLE类型区域的最大页帧号和movable\_zone类型的最大页帧号相等，也就数说ZONE\_MOVAB区域的最大页帧号和同一节点的其他类型的一个区域的最大页帧号相等，这样如果zone\_type不等于ZONE\_MOVABLE，想像ZONE\_MOVABLE区域从最大页帧向下扩展，则会出现三种情况：ZONE\_MOVABLE在zone\_typ区域内，zone\_typ区域在ZONE\_MOVABLE区域内，两个区域不相交。4076-4078处理的是ZONE\_MOVABLE在区域zone\_type内的情景，4081-4082行处理的是zone\_typ区域在ZONE\_MOVABLE区域内的情景。

#### absent\_pages\_in\_range函数

#### \_\_absent\_pages\_in\_range函数

         absent\_pages\_in\_range函数计算区间不可用的页面数。absent\_pages\_in\_range是调用\_\_absent\_pages\_in\_range来实现的， \_\_absent\_pages\_in\_range在mm/page\_alloc.c中实现代码如下：
    
         4121unsigned long \_\_meminit \_\_absent\_pages\_in\_range(int nid,

4122                                 unsigned longrange\_start\_pfn,

4123                                 unsigned longrange\_end\_pfn)

4124 {

4125        unsigned long nr\_absent = range\_end\_pfn - range\_start\_pfn;

4126        unsigned long start\_pfn, end\_pfn;

4127        int i;

4128

4129        for\_each\_mem\_pfn\_range(i, nid, &start\_pfn, &end\_pfn, NULL) {

4130                 start\_pfn = clamp(start\_pfn,range\_start\_pfn, range\_end\_pfn);

4131                 end\_pfn = clamp(end\_pfn,range\_start\_pfn, range\_end\_pfn);

4132                 nr\_absent -= end\_pfn -start\_pfn;

4133        }

4134        return nr\_absent;

4135 }

         用区间的总页面数减去在区间中所有空闲的页面数，就获得了区间中不可用的页面数。4125求的区间的页面数，4129-4133遍历每个在初始化分配器中的区段，减去区段在区间中的页面数。clamp函数返回三个参数的中间值。

#### get\_pfn\_range\_for\_nid函数

get\_pfn\_range\_for\_nid函数获得节点的页帧范围，在mm/page\_alloc.c中实现，代码如下：

4011 void \_\_meminitget\_pfn\_range\_for\_nid(unsigned int nid,

4012                         unsigned long\*start\_pfn, unsigned long \*end\_pfn)

4013 {

4014        unsigned long this\_start\_pfn, this\_end\_pfn;

4015        int i;

4016

4017        \*start\_pfn = -1UL;

4018        \*end\_pfn = 0;

4019

4020        for\_each\_mem\_pfn\_range(i, nid, &this\_start\_pfn, &this\_end\_pfn,NULL) {

4021                 \*start\_pfn = min(\*start\_pfn,this\_start\_pfn);

4022                 \*end\_pfn = max(\*end\_pfn,this\_end\_pfn);

4023        }

4024

4025        if (\*start\_pfn == -1UL)

4026                \*start\_pfn = 0;

4027 }

         get\_pfn\_range\_for\_nid函数遍历memblock分配器在节点的每个空闲区域，获得最大和最小页帧号。

#### zone\_absent\_pages\_in\_node函数

zone\_absent\_pages\_in\_node函数获得区域的不可用的页面数

4151 static unsigned long \_\_meminitzone\_absent\_pages\_in\_node(int nid,

4152                                         unsigned long zone\_type,

4153                                        unsigned long \*ignored)

4154 {

4155        unsigned long zone\_low = arch\_zone\_lowest\_possible\_pfn\[zone\_type\];

4156        unsigned long zone\_high = arch\_zone\_highest\_possible\_pfn\[zone\_type\];

4157        unsigned long node\_start\_pfn, node\_end\_pfn;

4158        unsigned long zone\_start\_pfn, zone\_end\_pfn;

4159

4160        get\_pfn\_range\_for\_nid(nid, &node\_start\_pfn, &node\_end\_pfn);

4161        zone\_start\_pfn = clamp(node\_start\_pfn, zone\_low, zone\_high);

4162        zone\_end\_pfn = clamp(node\_end\_pfn, zone\_low, zone\_high);

4163

4164        adjust\_zone\_range\_for\_zone\_movable(nid, zone\_type,

4165                         node\_start\_pfn,node\_end\_pfn,

4166                         &zone\_start\_pfn,&zone\_end\_pfn);

4167        return \_\_absent\_pages\_in\_range(nid, zone\_start\_pfn, zone\_end\_pfn);

4168 }

         4155-4156行从arch\_zone\_lowest\_possible\_pfn数组获得区域最小页帧号，从arch\_zone\_highest\_possible\_pfn数组获得区域最大页帧号。
    
         4160-4162行把区域范围限制到节点范围中。
    
         4167调用\_\_absent\_pages\_in\_range求得区域范围中不可用的页面数。

#### \=========================================

#### zone\_sizes\_init函数

         伙伴系统的初始化主要是指zone\_sizes\_init函数中完成的，调用zone\_sizes\_init函数的流程是：

start\_kernel () at init/main.c:496

setup\_arch () atarch/x86/kernel/setup.c:972

paging\_init () at arch/x86/mm/init\_32.c:700

zone\_sizes\_init () atarch/x86/mm/init.c:398

         zone\_sizes\_init函数在arch/x86/mm/init.c中实现，代码如下：

397 void \_\_init zone\_sizes\_init(void)

398 {

399        unsigned long max\_zone\_pfns\[MAX\_NR\_ZONES\];

400

401        memset(max\_zone\_pfns, 0, sizeof(max\_zone\_pfns));

402

403 #ifdef CONFIG\_ZONE\_DMA

404        max\_zone\_pfns\[ZONE\_DMA\]         =MAX\_DMA\_PFN;

405 #endif

406 #ifdef CONFIG\_ZONE\_DMA32

407        max\_zone\_pfns\[ZONE\_DMA32\]       =MAX\_DMA32\_PFN;

408 #endif

409        max\_zone\_pfns\[ZONE\_NORMAL\]      =max\_low\_pfn;

410 #ifdef CONFIG\_HIGHMEM

411        max\_zone\_pfns\[ZONE\_HIGHMEM\]     =max\_pfn;

412 #endif

413

414        free\_area\_init\_nodes(max\_zone\_pfns);

415 }

         max\_zone\_pfns是一数组定义了每个区域类型范围。max\_low\_pfn和max\_pfn在前面已经确定。

#### free\_area\_init\_nodes函数

         free\_area\_init\_nodes在mm/page\_alloc.c中实现，代码如下：

4734 void \_\_initfree\_area\_init\_nodes(unsigned long \*max\_zone\_pfn)

4735 {

4736        unsigned long start\_pfn, end\_pfn;

4737        int i, nid;

4738

4739        /\* Record where the zone boundaries are \*/

4740        memset(arch\_zone\_lowest\_possible\_pfn, 0,

4741                                sizeof(arch\_zone\_lowest\_possible\_pfn));

4742        memset(arch\_zone\_highest\_possible\_pfn, 0,

4743                                sizeof(arch\_zone\_highest\_possible\_pfn));

4744        arch\_zone\_lowest\_possible\_pfn\[0\] = find\_min\_pfn\_with\_active\_regions();

4745        arch\_zone\_highest\_possible\_pfn\[0\] = max\_zone\_pfn\[0\];

4746        for (i = 1; i < MAX\_NR\_ZONES; i++) {

4747                 if (i == ZONE\_MOVABLE)

4748                         continue;

4749                arch\_zone\_lowest\_possible\_pfn\[i\] =

4750                         arch\_zone\_highest\_possible\_pfn\[i-1\];

4751                arch\_zone\_highest\_possible\_pfn\[i\] =

4752                         max(max\_zone\_pfn\[i\],arch\_zone\_lowest\_possible\_pfn\[i\]);

4753        }

4754        arch\_zone\_lowest\_possible\_pfn\[ZONE\_MOVABLE\] = 0;

4755        arch\_zone\_highest\_possible\_pfn\[ZONE\_MOVABLE\] = 0;

4756

4757        /\* Find the PFNs that ZONE\_MOVABLE begins at in each node \*/

4758        memset(zone\_movable\_pfn, 0, sizeof(zone\_movable\_pfn));

4759        find\_zone\_movable\_pfns\_for\_nodes();

4760

4761        /\* Print out the zone ranges \*/

4762        printk("Zone PFN ranges:\\n");

4763        for (i = 0; i < MAX\_NR\_ZONES; i++) {

4764                 if (i == ZONE\_MOVABLE)

4765                         continue;

4766                 printk("  %-8s ", zone\_names\[i\]);

4767                 if(arch\_zone\_lowest\_possible\_pfn\[i\] ==

4768                                arch\_zone\_highest\_possible\_pfn\[i\])

4769                        printk("empty\\n");

4770                 else

4771                        printk("%0#10lx ->%0#10lx\\n",

4772                                arch\_zone\_lowest\_possible\_pfn\[i\],

4773                                arch\_zone\_highest\_possible\_pfn\[i\]);

4774        }

4775

4776        /\* Print out the PFNs ZONE\_MOVABLE begins at in each node \*/

4777        printk("Movable zone start PFN for each node\\n");

4778        for (i = 0; i < MAX\_NUMNODES; i++) {

4779                 if (zone\_movable\_pfn\[i\])

4780                         printk("  Node %d: %lu\\n", i, zone\_movable\_pfn\[i\]);

4781        }

4782

4783        /\* Print out the early\_node\_map\[\] \*/

4784        printk("Early memory PFN ranges\\n");

4785        for\_each\_mem\_pfn\_range(i, MAX\_NUMNODES, &start\_pfn, &end\_pfn,&nid)

4786                 printk("  %3d: %0#10lx -> %0#10lx\\n", nid,start\_pfn, end\_pfn);

4787

4788        /\* Initialise every node \*/

4789        mminit\_verify\_pageflags\_layout();

4790        setup\_nr\_node\_ids();

4791        for\_each\_online\_node(nid) {

4792                 pg\_data\_t \*pgdat = NODE\_DATA(nid);

4793                 free\_area\_init\_node(nid, NULL,

4794                                find\_min\_pfn\_for\_node(nid), NULL);

4795

4796                 /\* Any memory on that node \*/

4797                 if(pgdat->node\_present\_pages)

4798                        node\_set\_state(nid,N\_HIGH\_MEMORY);

4799                check\_for\_regular\_memory(pgdat);

4800        }

4801 }

         free\_area\_init\_nodes的代码比较长，但主要作了两项工作，确定节点的每个区域的上下界，然后对每个节点初始化。
    
         除ZONE\_MOVABLE区域类型外，区域范围的确定方法是用两个数组，arch\_zone\_lowest\_possible\_pfn确定区域的最小页帧号，arch\_zone\_highest\_possible\_pfn确定区域的最大页帧号，一个区域的页帧号pfn所允许的范围是arch\_zone\_lowest\_possible\_pfn<= pfn<arch\_zone\_highest\_possible\_pfn\[zone\_type\]。ZONE\_MOVABLE的区域的范围的确定方法涉及到一个数组zone\_movable\_pfn和一个变量movable\_zone，对一个节点号为nid的节点，ZONE\_MOVABLE的页帧号pfn的区域是：zone\_movable\_pfn\[nid\]<= pfn < arch\_zone\_highest\_possible\_pfn\[movable\_zone\]。
    
         4740-4753行求得从区域除ZONE\_MOVABLE区域类型外的区域范围，从区域范围的求法可以知道，区域在节点中是依次连续的。
    
         4754-4759行求ZONE\_MOVABLE区域的范围。其中关键是find\_zone\_movable\_pfns\_for\_nodes函数，分析本函数后分析find\_zone\_movable\_pfns\_for\_nodes函数。
    
         4762-4786行打印区域范围信息。
    
         4789行mminit\_verify\_pageflags\_layout函数验证位码信息并输出一些调式信息。
    
         4790行调用setup\_nr\_node\_ids函数设置节点总数。保存在变量nr\_node\_ids中。
    
         4791-4799一个循环，变量每个节点，4793行调用free\_area\_init\_node函数对每个节点初始化，free\_area\_init\_node函数中后面进行分析。4797-4799行主要是设置一些节点是否内存的状态信息。系统定义了一个枚举变量enum node\_states，用来记录一个节点是否能用（N\_POSSIBLE），是否在线（N\_ONLINE），是否具有普通内存区域（N\_NORMAL\_MEMORY），是否有普通内存或高端内存内存（N\_HIGH\_MEMORY），是否有连接有cpu（N\_CPU）。mm/page\_alloc.c中有个节点掩码数组node\_states\[\]对enum node\_states的每项都有个节点掩码，来记录节点的状态信息。

#### find\_zone\_movable\_pfns\_for\_nodes函数

         find\_zone\_movable\_pfns\_for\_nodes的工作是确定ZONE\_MOVABLE区域的范围。在mm/page\_alloc.c中实现，代码如下：

4567 static void \_\_initfind\_zone\_movable\_pfns\_for\_nodes(void)

4568 {

4569        int i, nid;

4570        unsigned long usable\_startpfn;

4571        unsigned long kernelcore\_node, kernelcore\_remaining;

4572        /\* save the state before borrow the nodemask \*/

4573        nodemask\_t saved\_node\_state = node\_states\[N\_HIGH\_MEMORY\];

4574        unsigned long totalpages = early\_calculate\_totalpages();

4575        int usable\_nodes = nodes\_weight(node\_states\[N\_HIGH\_MEMORY\]);

4576

4577        /\*

4578          \* If movablecore was specified,calculate what size of

4579          \* kernelcore that corresponds so thatmemory usable for

4580          \* any allocation type is evenly spread.If both kernelcore

4581          \* and movablecore are specified, thenthe value of kernelcore

4582          \* will be used forrequired\_kernelcore if it's greater than

4583          \* what movablecore would haveallowed.

4584          \*/

4585        if (required\_movablecore) {

4586                 unsigned long corepages;

4587

4588                 /\*

4589                  \* Round-up so thatZONE\_MOVABLE is at least as large as what

4590                  \* was requested by the user

4591                  \*/

4592                 required\_movablecore =

4593                        roundup(required\_movablecore, MAX\_ORDER\_NR\_PAGES);

4594                 corepages = totalpages -required\_movablecore;

4595

4596                 required\_kernelcore = max(required\_kernelcore,corepages);

4597        }

4598

4599        /\* If kernelcore was not specified, there is no ZONE\_MOVABLE \*/

4600        if (!required\_kernelcore)

4601                 goto out;

4602

4603        /\* usable\_startpfn is the lowest possible pfn ZONE\_MOVABLE can be at \*/

4604        find\_usable\_zone\_for\_movable();

4605        usable\_startpfn = arch\_zone\_lowest\_possible\_pfn\[movable\_zone\];

4606

4607 restart:

4608        /\* Spread kernelcore memory as evenly as possible throughout nodes \*/

4609        kernelcore\_node = required\_kernelcore / usable\_nodes;

4610        for\_each\_node\_state(nid, N\_HIGH\_MEMORY) {

4611                 unsigned long start\_pfn,end\_pfn;

4612

4613                 /\*

4614                  \* Recalculate kernelcore\_nodeif the division per node

4615                  \* now exceeds what isnecessary to satisfy the requested

4616                  \* amount of memory for thekernel

4617                  \*/

4618                 if (required\_kernelcore <kernelcore\_node)

4619                         kernelcore\_node =required\_kernelcore / usable\_nodes;

4620

4621                 /\*

4622                  \* As the map is walked, wetrack how much memory is usable

4623                  \* by the kernel usingkernelcore\_remaining. When it is

4624                  \* 0, the rest of the node isusable by ZONE\_MOVABLE

4625                  \*/

4626                 kernelcore\_remaining =kernelcore\_node;

4627

4628                 /\* Go through each range ofPFNs within this node \*/

4629                 for\_each\_mem\_pfn\_range(i, nid,&start\_pfn, &end\_pfn, NULL) {

4630                         unsigned longsize\_pages;

4631

4632                         start\_pfn =max(start\_pfn, zone\_movable\_pfn\[nid\]);

4633                         if (start\_pfn >=end\_pfn)

4634                                 continue;

4635

4636                         /\* Account for what isonly usable for kernelcore \*/

4637                         if (start\_pfn <usable\_startpfn) {

4638                                 unsigned longkernel\_pages;

4639                                 kernel\_pages =min(end\_pfn, usable\_startpfn)

4640                                                                - start\_pfn;

4641

4642                                kernelcore\_remaining -= min(kernel\_pages,

4643                                                        kernelcore\_remaining);

4644                                required\_kernelcore -= min(kernel\_pages,

4645                                                        required\_kernelcore);

4646

4647                                 /\* Continue ifrange is now fully accounted \*/

4648                                 if (end\_pfn<= usable\_startpfn) {

4649

4650                                         /\*

4651                                          \* Push zone\_movable\_pfn to the endso

4652                                          \*that if we have to rebalance

4653                                          \*kernelcore across nodes, we will

4654                                          \* notdouble account here

4655                                          \*/

4656                                        zone\_movable\_pfn\[nid\] = end\_pfn;

4657                                        continue;

4658                                 }

4659                                start\_pfn =usable\_startpfn;

4660                         }

4661

4662                         /\*

4663                          \* The usable PFNrange for ZONE\_MOVABLE is from

4664                          \*start\_pfn->end\_pfn. Calculate size\_pages as the

4665                          \* number of pagesused as kernelcore

4666                          \*/

4667                         size\_pages = end\_pfn -start\_pfn;

4668                         if (size\_pages >kernelcore\_remaining)

4669                                 size\_pages =kernelcore\_remaining;

4670                         zone\_movable\_pfn\[nid\]= start\_pfn + size\_pages;

4671

4672                         /\*

4673                          \* Some kernelcore hasbeen met, update counts and

4674                          \* break if thekernelcore for this node has been

4675                          \* satisified

4676                          \*/

4677                         required\_kernelcore -=min(required\_kernelcore,

4678                                                                size\_pages);

4679                         kernelcore\_remaining-= size\_pages;

4680                         if(!kernelcore\_remaining)

4681                                 break;

4682                 }

4683        }

4684

4685        /\*

4686          \* If there is stillrequired\_kernelcore, we do another pass with one

4687          \* less node in the count. This willpush zone\_movable\_pfn\[nid\] further

4688          \* along on the nodes that still havememory until kernelcore is

4689          \* satisified

4690          \*/

4691        usable\_nodes--;

4692        if (usable\_nodes && required\_kernelcore > usable\_nodes)

4693                 goto restart;

4694

4695        /\* Align start of ZONE\_MOVABLE on all nids to MAX\_ORDER\_NR\_PAGES \*/

4696        for (nid = 0; nid < MAX\_NUMNODES; nid++)

4697                 zone\_movable\_pfn\[nid\] =

4698                        roundup(zone\_movable\_pfn\[nid\], MAX\_ORDER\_NR\_PAGES);

4699

4700 out:

4701        /\* restore the node\_state \*/

4702        node\_states\[N\_HIGH\_MEMORY\] = saved\_node\_state;

4703 }

         这个函数的目的是计算zone\_movable\_pfn数组。在系统中有两个变量required\_movablecore和required\_kernelcore，这两个变量的值是通过命令行传进来的，变量required\_movablecore通知内核保留给ZONE\_MOVABLE区域的页面数，required\_kernelcore是需要保留的非ZONE\_MOVABLE区域的页面数。
    
         4585-4601行，由4585行和4600行知道，如果这两个数据都没有通过命令行设置，则直接跳到out标号，也就是ZONE\_MOVABLE区域为空。corepages变量由early\_calculate\_totalpages初始化，是空闲内存的总数，roundup(x, y)是一个宏，返回大于等于x的是y的倍数的第一个数。4592-4593行设置required\_movablecore是MAX\_ORDER\_NR\_PAGES的倍数，4596行如果设置为指定的required\_kernelcore和剩余的空闲的区域required\_movablecore页后的页面，其实也就是让页面优先用做非ZONE\_MOVABLE区域的页面数。

在4602行的后面required\_movablecore变量没有再出现，后面的代码主要做了两部分工作，先选定一个区域，选的方法是从高到低的第一个不空的非ZONE\_MOVABLE区域，然后在这个区域的低端往上收缩，保证非ZONE\_MOVABLE区域的页面数达到required\_kernelcore。

         4604行调用函数find\_usable\_zone\_for\_movable设置变量movable\_zone，movable\_zone被设置的值就是最高不空的非ZONE\_MOVABLE区域。
    
         4605行设置usable\_startpfn变量的值，usable\_startpfn也就是第一个能作为ZONE\_MOVABL区域的页帧的值。
    
         4609行设置kernelcore\_node变量的值，usable\_nodes一个商数，初始化为是具有ZONE\_MOVABLE区域的节点数，在第一次扫描中kernelcore\_node初始化为对每个节点是均匀保留非ZONE\_MOVABLE区域页面的，以后每次扫描会自减usable\_node。在计算zone\_movable\_pfn数组时，会对一个节点集合遍历，kernelcore\_node变量是每个节点应该保留给非ZONE\_MOVABLE区域的页面数。        
    
         4610行对在节点掩码node\_states\[N\_HIGH\_MEMORY\]中可用的每个节点进行扫描。
    
         4618-4619行如果required\_kernelcore < kernelcore\_node重新设置kernelcore\_node变量的值
    
         4626行kernelcore\_remaining变量是在本次对节点的扫描要变量的页面数，赋值为required\_kernelcore。
    
         4629行对每个初始化内存分配器中的空闲区域进行遍历。
    
         4632-4634行zone\_movable\_pfn\[nid\]是本次扫描节点ZONE\_MOVABLE区域的最小页帧号，如果end\_pfn <=zone\_movable\_pfn\[nid\]或者end\_pfn <=start\_pfn就是本次扫描的空闲区段不再ZONE\_MOVABLE区域范围内或者是空区段，继续扫描下一个区段。
    
         4637行，start\_pfn是本次扫描的空闲区段的首页帧，usable\_startpfn是ZONE\_MOVABLE区域锁允许的最小帧。start\_pfn< usable\_startpfn意味着start\_pfn -->usable\_startpfn的帧是属于非ZONE\_MOVABLE区域的。4638-4645在所要保留的页面数中减去这段包含的页面。
    
         4648行end\_pfn <= usable\_startpfn表示正空闲区段都属于非ZONE\_MOVABLE区域。4656行zone\_movable\_pfn\[nid\] = end\_pfn，如果保留给非ZONE\_MOVABLE区域的区域已经足够，用本次扫描的空闲区段尾做本节点的ZONE\_MOVABLE区域首页帧号。注意一点区段是包含首页帧号start\_pfn，不包含尾帧end\_pfn。
    
         代码执行到4659行表示end\_pfn >usable\_startpfn，执行start\_pfn = usable\_startpfn把usable\_startpfnàend\_pfn当成一个空闲区域执行后面的代码。
    
         4667-4681行，执行到这段代码，表示整个区段都在都是可以作为ZONE\_MOVABLE页面，这段代码中这个空闲区段中保留非ZONE\_MOVABLE区域页面。
    
         4691-4693行自减商数usable\_nodes，并测试usable\_nodes&& required\_kernelcore > usable\_nodes，这样可以比较无限循环，并在每个节点需要保留的非ZONE\_MOVABLE区域页的数量大于1时，重新扫描。
    
         4696-4698行对齐ZONE\_MOVABLE区域的首页帧。
    
         4702恢复node\_state数组。

#### free\_area\_init\_node函数

         free\_area\_init\_node函数初始化节点，在mm/page\_alloc.c中实现，代码如下：

4420 void \_\_paginginitfree\_area\_init\_node(int nid, unsigned long \*zones\_size,

4421                 unsigned long node\_start\_pfn,unsigned long \*zholes\_size)

4422 {

4423        pg\_data\_t \*pgdat = NODE\_DATA(nid);

4424

4425        pgdat->node\_id = nid;

4426        pgdat->node\_start\_pfn = node\_start\_pfn;

4427        calculate\_node\_totalpages(pgdat, zones\_size, zholes\_size);

4428

4429        alloc\_node\_mem\_map(pgdat);

4430 #ifdef CONFIG\_FLAT\_NODE\_MEM\_MAP

4431        printk(KERN\_DEBUG "free\_area\_init\_node: node %d, pgdat %08lx,node\_mem\_map %08lx\\n",

4432                 nid, (unsigned long)pgdat,

4433                 (unsignedlong)pgdat->node\_mem\_map);

4434 #endif

4435

4436        free\_area\_init\_core(pgdat, zones\_size, zholes\_size);

4437 }

         free\_area\_init\_node函数调用calculate\_node\_totalpages对节点长度和节点总可用页面数进行初始化。calculate\_node\_totalpages函数是通过调用zone\_spanned\_pages\_in\_node和

zone\_absent\_pages\_in\_node函数实现的，这两个函数上面已经分析过。

alloc\_node\_mem\_map是对节点的page管理数据初始化。其他的初始化工作在free\_area\_init\_core函数中完成。

#### alloc\_node\_mem\_map函数

         alloc\_node\_mem\_map函数分配节点的page管理数组的内存，在mm/page\_alloc.c中实现，代码如下：

4379 static void \_\_init\_refokalloc\_node\_mem\_map(struct pglist\_data \*pgdat)

4380 {

4381        /\* Skip empty nodes \*/

4382        if (!pgdat->node\_spanned\_pages)

4383                 return;

4384

4385 #ifdef CONFIG\_FLAT\_NODE\_MEM\_MAP

4386        /\* ia64 gets its own node\_mem\_map, before this, without bootmem \*/

4387        if (!pgdat->node\_mem\_map) {

4388                 unsigned long size, start,end;

4389                 struct page \*map;

4390

4391                 /\*

4392                  \* The zone's endpoints aren'trequired to be MAX\_ORDER

4393                  \* aligned but thenode\_mem\_map endpoints must be in order

4394                 \* for the buddyallocator to function correctly.

4395                  \*/

4396                 start =pgdat->node\_start\_pfn & ~(MAX\_ORDER\_NR\_PAGES - 1);

4397                 end = pgdat->node\_start\_pfn+ pgdat->node\_spanned\_pages;

4398                end = ALIGN(end,MAX\_ORDER\_NR\_PAGES);

4399                 size =  (end - start) \* sizeof(struct page);

4400                 map =alloc\_remap(pgdat->node\_id, size);

4401                 if (!map)

4402                         map = alloc\_bootmem\_node\_nopanic(pgdat,size);

4403                 pgdat->node\_mem\_map = map +(pgdat->node\_start\_pfn - start);

4404        }

4405 #ifndef CONFIG\_NEED\_MULTIPLE\_NODES

4406        /\*

4407          \* With no DISCONTIG, the globalmem\_map is just set as node 0's

4408          \*/

4409        if (pgdat == NODE\_DATA(0)) {

4410                 mem\_map =NODE\_DATA(0)->node\_mem\_map;

4411 #ifdef CONFIG\_HAVE\_MEMBLOCK\_NODE\_MAP

4412                 if (page\_to\_pfn(mem\_map) !=pgdat->node\_start\_pfn)

4413                         mem\_map -= (pgdat->node\_start\_pfn -ARCH\_PFN\_OFFSET);

4414 #endif /\*CONFIG\_HAVE\_MEMBLOCK\_NODE\_MAP \*/

4415        }

4416 #endif

4417 #endif /\* CONFIG\_FLAT\_NODE\_MEM\_MAP \*/

4418 }

         在节点结构pglist\_data中，成员node\_start\_pfn是节点的首页帧号，node\_spanned\_pages是包含中间不可用页面的节点的长度。node\_mem\_map指向节点page结构管理数组，并且指向节点首页的page结构。
    
         4388-4403行的代码执行逻辑是：计算一个页帧范围，这个范围是包含节点的所有页面的最小范围，并且起始页帧和尾页帧都是按最大块对齐的。然后按这个范围来分配存放page结构数组的内存。分配完后（4403行）让node\_mem\_map成员指向node\_start\_pfn页帧的page结构地址。
    
         对page数组的内存是调用alloc\_remap和alloc\_bootmem\_node\_nopanic进行分配的，这两个函数中初始化内存分频器章节中介绍。
    
         4410行，在较早的版本，page管理数组的首地址是存放在变量mem\_map中的，现在这个变量指向第零个节点的page管理数组
    
         4412-4413行对page管理结构地址到页帧的转换进行校正。

#### free\_area\_init\_core函数

         free\_area\_init\_core是伙伴系统初始化的核心函数，在mm/page\_alloc.c中实现，代码如下：

4291 static void \_\_paginginitfree\_area\_init\_core(struct pglist\_data \*pgdat,

4292                 unsigned long \*zones\_size,unsigned long \*zholes\_size)

4293 {

4294        enum zone\_type j;

4295        int nid = pgdat->node\_id;

4296        unsigned long zone\_start\_pfn = pgdat->node\_start\_pfn;

4297        int ret;

4298

4299        pgdat\_resize\_init(pgdat);

4300        pgdat->nr\_zones = 0;

4301        init\_waitqueue\_head(&pgdat->kswapd\_wait);

4302        pgdat->kswapd\_max\_order = 0;

4303        pgdat\_page\_cgroup\_init(pgdat);

4304

4305        for (j = 0; j < MAX\_NR\_ZONES; j++) {

4306                 struct zone \*zone =pgdat->node\_zones + j;

4307                 unsigned long size, realsize,memmap\_pages;

4308                 enum lru\_list lru;

4309

4310                 size = zone\_spanned\_pages\_in\_node(nid,j, zones\_size);

4311                 realsize = size -zone\_absent\_pages\_in\_node(nid, j,

4312                                                                zholes\_size);

4313

4314                 /\*

4315                  \* Adjust realsize so that itaccounts for how much memory

4316                  \* is used by this zone formemmap. This affects the watermark

4317                  \* and per-cpu initialisations

4318                  \*/

4319                 memmap\_pages =

4320                         PAGE\_ALIGN(size \* sizeof(structpage)) >> PAGE\_SHIFT;

4321                 if (realsize >=memmap\_pages) {

4322                         realsize -=memmap\_pages;

4323                         if (memmap\_pages)

4324                                 printk(KERN\_DEBUG

4325                                       "  %s zone: %lu pages usedfor memmap\\n",

4326                                       zone\_names\[j\], memmap\_pages);

4327                 } else

4328                         printk(KERN\_WARNING

4329                                 "  %s zone: %lu pages exceeds realsize%lu\\n",

4330                                 zone\_names\[j\],memmap\_pages, realsize);

4331

4332                 /\* Account for reserved pages\*/

4333                 if (j == 0 && realsize> dma\_reserve) {

4334                         realsize -=dma\_reserve;

4335                         printk(KERN\_DEBUG"  %s zone: %lu pagesreserved\\n",

4336                                        zone\_names\[0\], dma\_reserve);

4337                 }

4338

4339                 if (!is\_highmem\_idx(j))

4340                         nr\_kernel\_pages +=realsize;

4341                 nr\_all\_pages += realsize;

4342

4343                 zone->spanned\_pages = size;

4344                 zone->present\_pages = realsize;

4345 #ifdef CONFIG\_NUMA

4346                 zone->node = nid;

4347                 zone->min\_unmapped\_pages =(realsize\*sysctl\_min\_unmapped\_ratio)

4348                                                / 100;

4349                 zone->min\_slab\_pages =(realsize \* sysctl\_min\_slab\_ratio) / 100;

4350 #endif

4351                 zone->name = zone\_names\[j\];

4352                spin\_lock\_init(&zone->lock);

4353                spin\_lock\_init(&zone->lru\_lock);

4354                 zone\_seqlock\_init(zone);

4355                 zone->zone\_pgdat = pgdat;

4356

4357                 zone\_pcp\_init(zone);

4358                 for\_each\_lru(lru)

4359                        INIT\_LIST\_HEAD(&zone->lruvec.lists\[lru\]);

4360                 zone->reclaim\_stat.recent\_rotated\[0\]= 0;

4361                zone->reclaim\_stat.recent\_rotated\[1\] = 0;

4362                zone->reclaim\_stat.recent\_scanned\[0\] = 0;

4363                zone->reclaim\_stat.recent\_scanned\[1\] = 0;

4364                 zap\_zone\_vm\_stats(zone);

4365                zone->flags = 0;

4366                 if (!size)

4367                         continue;

4368

4369                set\_pageblock\_order(pageblock\_default\_order());

4370                 setup\_usemap(pgdat, zone,size);

4371                 ret =init\_currently\_empty\_zone(zone, zone\_start\_pfn,

4372                                                size, MEMMAP\_EARLY);

4373                 BUG\_ON(ret);

4374                 memmap\_init(size, nid, j,zone\_start\_pfn);

4375                 zone\_start\_pfn += size;

4376        }

4377 }

         这个函数的代码比较长，但比较简单，就一些变量，锁和链表的初始化。对这个函数本身就不做分析了，而对函数中调用的memmap\_init做些介绍，memmap\_init是一个宏定义如下：

#define memmap\_init(size, nid, zone,start\_pfn) \\

         memmap\_init\_zone((size),(nid), (zone), (start\_pfn), MEMMAP\_EARLY)。

是对memmap\_init\_zone函数的调用。

#### memmap\_init\_zone函数

         memmap\_init\_zone对一个区域的page管理结构的初始化，在mm/page\_alloc.c中实现，代码如下：

3619 \* done. Non-atomic initialization, single-pass.

3620 \*/

3621 void \_\_meminitmemmap\_init\_zone(unsigned long size, int nid, unsigned long zone,

3622                unsigned long start\_pfn,enum memmap\_context context)

3623 {

3624        struct page \*page;

3625        unsigned long end\_pfn = start\_pfn + size;

3626        unsigned long pfn;

3627        struct zone \*z;

3628

3629        if (highest\_memmap\_pfn < end\_pfn - 1)

3630                 highest\_memmap\_pfn = end\_pfn -1;

3631

3632        z = &NODE\_DATA(nid)->node\_zones\[zone\];

3633        for (pfn = start\_pfn; pfn < end\_pfn; pfn++) {

3634                 /\*

3635                  \* There can be holes inboot-time mem\_map\[\]s

3636                  \* handed to thisfunction.  They do not

3637                  \* exist on hotplugged memory.

3638                  \*/

3639                 if (context == MEMMAP\_EARLY) {

3640                         if (!early\_pfn\_valid(pfn))

3641                                 continue;

3642                         if(!early\_pfn\_in\_nid(pfn, nid))

3643                                 continue;

3644                 }

3645                 page = pfn\_to\_page(pfn);

3646                 set\_page\_links(page, zone, nid, pfn);

3647                 mminit\_verify\_page\_links(page,zone, nid, pfn);

3648                 init\_page\_count(page);

3649                 reset\_page\_mapcount(page);

3650                 SetPageReserved(page);

3651                /\*

3652                  \* Mark the block movable sothat blocks are reserved for

3653                  \* movable at startup. Thiswill force kernel allocations

3654                  \* to reserve their blocksrather than leaking throughout

3655                  \* the address space duringboot when many long-lived

3656                  \* kernel allocations aremade. Later some blocks near

3657                  \* the start are markedMIGRATE\_RESERVE by

3658                  \* setup\_zone\_migrate\_reserve()

3659                  \*

3660                  \* bitmap is created forzone's valid pfn range. but memmap

3661                  \* can be created for invalidpages (for alignment)

3662                  \* check here not to callset\_pageblock\_migratetype() against

3663                  \* pfn out of zone.

3664                  \*/

3665                 if ((z->zone\_start\_pfn<= pfn)

3666                     && (pfn <z->zone\_start\_pfn + z->spanned\_pages)

3667                     && !(pfn &(pageblock\_nr\_pages - 1)))

3668                        set\_pageblock\_migratetype(page, MIGRATE\_MOVABLE);

3669

3670                INIT\_LIST\_HEAD(&page->lru);

3671 #ifdef WANT\_PAGE\_VIRTUAL

3672                 /\* The shift won't overflowbecause ZONE\_NORMAL is below 4G. \*/

3673                 if (!is\_highmem\_idx(zone))

3674                         set\_page\_address(page,\_\_va(pfn << PAGE\_SHIFT));

3675 #endif

3676        }

3677 }

         3629-3630行highest\_memmap\_pfn是存在page管理结构的最大的页帧号，如果本管理区的最大的存在page管理结构的最大的页帧号大于highest\_memmap\_pfn，就需要更新highest\_memmap\_pfn。
    
         3632行获得区域结构地址。
    
         3633对区域的所有页帧进行遍历。
    
         3640-3641行检查页帧号是否合法，也就是要小于系统最大的页帧号，大于系统允许的最小的页帧。
    
         3642-3643行检查页帧pfn是否属于节点nid。
    
         3645行获得pfn帧的page管理结构地址。
    
         3646行调用set\_page\_links函数设置页面的一些链接，主要包含页面所在节点，页面的区域类型，页面所在段。这样信息都是保存在page结构的成员flags中，每种信息占用一些位。3647行对设置的页面所在节点，页面的区域类型，页面所在段的信息进行验证，如果有错误输出一些调试信息。
    
         3648初始引用数信息，3649初始化映射数信息。
    
         3665-3668行，对每个最大块的首帧，调用set\_pageblock\_migratetype函数设置迁移类型信息，set\_pageblock\_migratetype函数在伙伴系统的内存迁移一节有分析。
    
         3647行设置页面映射的虚拟地址。

####       ====区域列表的初始化

#### build\_all\_zonelists函数

         区域列表的初始化由函数build\_all\_zonelists来完成，build\_all\_zonelists函数的进入路径是：
    
         start\_kernel() at init/main.c:504

build\_all\_zonelists() at mm/page\_alloc.c:3409

build\_all\_zonelists在mm/page\_alloc.c中实现，代码如下：

3408 void \_\_refbuild\_all\_zonelists(void \*data)

3409 {

3410         set\_zonelist\_order();

3411

3412         if (system\_state == SYSTEM\_BOOTING) {

3413                 \_\_build\_all\_zonelists(NULL);

3414                 mminit\_verify\_zonelist();

3415                cpuset\_init\_current\_mems\_allowed();

3416         } else {

3417                 /\* we have to stop all cpus toguarantee there is no user

3418                    of zonelist \*/

3419 #ifdefCONFIG\_MEMORY\_HOTPLUG

3420                 if (data)

3421                        setup\_zone\_pageset((struct zone \*)data);

3422 #endif

3423                stop\_machine(\_\_build\_all\_zonelists, NULL, NULL);

3424                 /\* cpuset refresh routineshould be here \*/

3425         }

3426         vm\_total\_pages =nr\_free\_pagecache\_pages();

3427         /\*

3428          \* Disable grouping by mobility if thenumber of pages in the

3429          \* system is too low to allow themechanism to work. It would be

3430          \* more accurate, but expensive tocheck per-zone. This check is

3431          \* made on memory-hotadd so a system canstart with mobility

3432          \* disabled and enable it later

3433          \*/

3434         if (vm\_total\_pages <(pageblock\_nr\_pages \* MIGRATE\_TYPES))

3435                page\_group\_by\_mobility\_disabled = 1;

3436         else

3437                 page\_group\_by\_mobility\_disabled= 0;

3438

3439         printk("Built %i zonelists in %sorder, mobility grouping %s.  "

3440                 "Total pages:%ld\\n",

3441                         nr\_online\_nodes,

3442                         zonelist\_order\_name\[current\_zonelist\_order\],

3443                        page\_group\_by\_mobility\_disabled ? "off" : "on",

3444                         vm\_total\_pages);

3445 #ifdefCONFIG\_NUMA

3446         printk("Policy zone: %s\\n",zone\_names\[policy\_zone\]);

3447 #endif

3448 }

在初始化过程中，函数会进入3413-3415行代码运行。

3413行区域列表的初始的主体工作是在\_\_build\_all\_zonelists中完成的。介绍完本函数后介绍\_\_build\_all\_zonelists函数。

3414行调用mminit\_verify\_zonelist函数做一些验证工作。

         在伙伴系统的内存分配一节中，我们把伙伴系统内存分为三个阶段，而第一阶段的主要任务是确定区域列表和节点掩码。在进程结构中有个成员mems\_allowed，是一个节点掩码，表示进程所允许分配内存的节点，只有一个节点包含在进程的mems\_allowed中，并且在内存策略也允许在这个节点进行分配时才会到这个节点进行内存分配。cpuset\_init\_current\_mems\_allowed设置进程的mems\_allowed成员包含所有节点。
    
         3626行，nr\_free\_pagecache\_pages返回的是对所有区域可用页面数减去高水位线后的的剩余页面数相加的值，这个值作为剩余可用页面数。
    
         3434行，如果剩余可用页面小于pageblock\_nr\_pages \* MIGRATE\_TYPES，也就是说如果不能满足每个迁移类型都包含一个迁移块。则禁用迁移类型，禁用迁移类型后所有页面的迁移都会迁移到MIGRATE\_UNMOVABLE迁移类型，也就是不可迁移类型。

#### \_\_build\_all\_zonelists函数

         \_\_build\_all\_zonelists在mm/page\_alloc.c中实现，代码如下：

3356 static\_\_init\_refok int \_\_build\_all\_zonelists(void \*data)

3357 {

3358         int nid;

3359         int cpu;

3360

3361 #ifdefCONFIG\_NUMA

3362         memset(node\_load, 0,sizeof(node\_load));

3363 #endif

3364         for\_each\_online\_node(nid) {

3365                 pg\_data\_t \*pgdat =NODE\_DATA(nid);

3366

3367                 build\_zonelists(pgdat);

3368                 build\_zonelist\_cache(pgdat);

3369         }

3370

3371         /\*

3372          \* Initialize the boot\_pagesets thatare going to be used

3373          \* for bootstrapping processors. Thereal pagesets for

3374          \* each zone will be allocated laterwhen the per cpu

3375          \* allocator is available.

3376          \*

3377          \* boot\_pagesets are used also forbootstrapping offline

3378          \* cpus if the system is alreadybooted because the pagesets

3379          \* are needed to initialize allocatorson a specific cpu too.

3380          \* F.e. the percpu allocator needs thepage allocator which

3381          \* needs the percpu allocator in orderto allocate its pagesets

3382          \* (a chicken-egg dilemma).

3383          \*/

3384         for\_each\_possible\_cpu(cpu) {

3385                setup\_pageset(&per\_cpu(boot\_pageset, cpu), 0);

3386

3387 #ifdefCONFIG\_HAVE\_MEMORYLESS\_NODES

3388                 /\*

3389                  \* We now know the "localmemory node" for each node--

3390                  \* i.e., the node of the firstzone in the generic zonelist.

3391                  \* Set up numa\_mem percpuvariable for on-line cpus.  During

3392                  \* boot, only the boot cpushould be on-line;  we'll init the

3393                  \* secondary cpus' numa\_mem as theycome on-line.  During

3394                  \* node/memory hotplug, we'llfixup all on-line cpus.

3395                  \*/

3396                 if (cpu\_online(cpu))

3397                         set\_cpu\_numa\_mem(cpu,local\_memory\_node(cpu\_to\_node(cpu)));

3398 #endif

3399         }

3400

3401         return 0;

3402 }

区域列表是区域的有序集合，设置区域列表的目的是为了从列表中选择一个区域，在区域中进行内存分配。

有几个因素会影响区域的选择：

1：一个是区域在区域列表中的顺序。

2：还有一个是分配标志位指定的最大区域类型，一些分配只能在低端内存中分配，如一些只支持低端内存访问的设备驱动程序。当选择一个区域时，要考虑区域的类型，只有区域类型小于等于标志位指定的最大区域类型，才选择这个区域。

3：在分配的时候，如果快速通道分配内存失败，在慢速通道中会记录区域内存不充足缓存信息，在内存的时候会检查内存内存是否充足的缓存信息，这会影响区域的选择。

4: 节点掩码也会影响区域的选择，只会选择在节点掩码集合中的区域。

考虑这几个因素，我们就可以解释区域列表的结构zonelist的定义了，为什么在列表中定义一个zoneref数组，而不直接定义一个zone的数组指针？zoneref结构包含一个zone结构指针zone和zone\_idx是区域的类型，考虑第二个因素，在我们扫描区域列表的一项，需要的区域类型直接可以从zoneref成员的zone\_idx得到。

而zonelist的成员zlcache是个zonelist\_cache结构。用来保存区域的内存是否充足信息，对区域列表中的每个区域，zonelist\_cache结构的成员fullzones，是个位图数组，和zonelist结构的zoneref数组是对应的，用来表示zoneref数组索引的项内存是否充足，z\_to\_n用来实现从数组索引到节点号的转换，在zlc\_zone\_worth\_trying函数中会用到这些参数。

zonelist的成员zlcache\_ptr指向实际可用的zonelist\_cache结构地址，zlcache\_ptr不总是指向zonelist的zonelist\_cache。

3364-3369行，遍历所有在线的节点，调用函数build\_zonelists初始化节点的区域列表，每个节点包含若干个区域列表。调用build\_zonelist\_cache初始化节点的内存是否充足缓存信息。

3384-3399行，编译所有可用的cpu，调用setup\_pageset初始化每cpu页缓存信息。3396-3397行对在线的cpu，调用set\_cpu\_numa\_mem设置cpu所在节点。

在后面只介绍build\_zonelists的指向流程，build\_zonelist\_cache和其他部分不分析了。

#### build\_zonelists函数

         build\_zonelists初始化一个节点的区域列表，在mm/page\_alloc.c中实现，代码如下：

3286 static void build\_zonelists(pg\_data\_t\*pgdat)

3287 {

3288        int node, local\_node;

3289        enum zone\_type j;

3290        struct zonelist \*zonelist;

3291

3292        local\_node =pgdat->node\_id;

3293

3294        zonelist = &pgdat->node\_zonelists\[0\];

3295        j = build\_zonelists\_node(pgdat, zonelist, 0, MAX\_NR\_ZONES - 1);

3296

3297        /\*

3298          \* Now we build the zonelist so thatit contains the zones

3299          \* of all the other nodes.

3300          \* We don't want to pressure aparticular node, so when

3301          \* building the zones for node N, wemake sure that the

3302          \* zones coming right after the localones are those from

3303          \* node N+1 (modulo N)

3304          \*/

3305        for (node = local\_node + 1; node < MAX\_NUMNODES; node++) {

3306                 if (!node\_online(node))

3307                         continue;

3308                 j = build\_zonelists\_node(NODE\_DATA(node),zonelist, j,

3309                                                        MAX\_NR\_ZONES - 1);

3310        }

3311        for (node = 0; node < local\_node; node++) {

3312                 if (!node\_online(node))

3313                         continue;

3314                 j =build\_zonelists\_node(NODE\_DATA(node), zonelist, j,

3315                                                        MAX\_NR\_ZONES - 1);

3316        }

3317

3318        zonelist->\_zonerefs\[j\].zone = NULL;

3319        zonelist->\_zonerefs\[j\].zone\_idx = 0;

3320 }

         build\_zonelists\_node函数把一个包含的区域编译到区域列表。
    
         这个函数的重点是区域列表初始化的顺序，local\_node是本节点的号码，从3295，3305，3311行我们可以知道，对在线的节点点，对节点的初始化顺序是local\_node, local\_node+1,…,MAX\_NR\_ZONES – 1,0,…, local\_node-1。
    
         3318-3319我们知道对最后一个区域索引项，索引的是空区域，而前面的每个区域索引项都指向非空区域，这样我们可以判断区域列表的结束。

#### build\_zonelists\_node函数

build\_zonelists\_node把一个节点的区域编译到区域列表，把节点pgdat中类型小于等于zone\_type的区域以nr\_zones项开始编译到区域列表zonelist。build\_zonelists\_node函数在mm/page\_alloc.c中实现，代码如下：

2860 static intbuild\_zonelists\_node(pg\_data\_t \*pgdat, struct zonelist \*zonelist,

2861                                 int nr\_zones,enum zone\_type zone\_type)

2862 {

2863        struct zone \*zone;

2864

2865        BUG\_ON(zone\_type >= MAX\_NR\_ZONES);

2866        zone\_type++;

2867

2868        do {

2869                 zone\_type--;

2870                 zone = pgdat->node\_zones +zone\_type;

2871                 if (populated\_zone(zone)) {

2872                         zoneref\_set\_zone(zone,

2873                                &zonelist->\_zonerefs\[nr\_zones++\]);

2874                        check\_highest\_zone(zone\_type);

2875                 }

2876

2877        } while (zone\_type);

2878        return nr\_zones;

2879 }

         区域被编译的顺序和区域类型是一致的，populated\_zone是判断区域是否具有可用页面，有可用页返回真，否则返回假。check\_highest\_zone更新policy\_zone变量，policy\_zone变量保存在系统中能用的非ZONE\_MOVABLE的最大的区域类型。