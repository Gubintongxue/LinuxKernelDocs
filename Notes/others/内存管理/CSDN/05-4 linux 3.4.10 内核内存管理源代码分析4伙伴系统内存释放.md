# linux 3.4.10 内核内存管理源代码分析4:伙伴系统内存释放

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

[linux 3.4.10 内核内存管理源代码分析4:伙伴系统内存释放_sf2507芯片使用-CSDN博客](https://blog.csdn.net/ancjf/article/details/8957281)