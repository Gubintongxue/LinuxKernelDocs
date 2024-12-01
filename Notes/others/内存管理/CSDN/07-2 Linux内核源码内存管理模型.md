# <Linux内核源码>内存管理模型

## [<Linux内核源码>内存管理模型](http://www.cnblogs.com/guguli/p/4489272.html)

**题外语：本人对linux内核的了解尚浅，如果有差池欢迎指正，也欢迎提问交流！**

 http://www.cnblogs.com/guguli/p/4489272.html

**首先要理解一下每一个进程是如何维护自己独立的寻址空间的，我的电脑里呢是8G内存空间。了解过的朋友应该都知道这是虚拟内存技术解决的这个问题，然而再linux中具体是怎样的模型解决的操作系统的这个设计需求的呢，让我们从linux源码的片段开始看吧！（以下内核源码均来自fedora21 64位系统的fc-3.19.3版本内核）**

**<include/linux/mm\_type.h>中对于物理页面的定义struct page，也就是我们常说的页表，关于这里的结构体的每个变量/位的操作函数大部分在<include/linux/mm.h>中。**

![](image/cdec0645add3fc3c328197dda5c76203.gif) ![](image/81178cc93a2a3bb5048d90d76e7ec935.gif)

![复制代码](image/69c5a8ac3fa60e0848d784a6dd461da6.gif)

  1 struct page {
  2     /\* First double word block \*/
  3     unsigned long flags;        /\* Atomic flags, some possibly  4 \* updated asynchronously \*/
  5     union {
  6         struct address\_space \*mapping;    /\* If low bit clear, points to  7                          \* inode address\_space, or NULL.
  8                          \* If page mapped as anonymous
  9                          \* memory, low bit is set, and
 10                          \* it points to anon\_vma object:
 11                          \* see PAGE\_MAPPING\_ANON below.
 12                          \*/
 13         void \*s\_mem;            /\* slab first object \*/
 14     };
 15 
 16     /\* Second double word \*/
 17     struct {
 18         union {
 19             pgoff\_t index;        /\* Our offset within mapping. \*/
 20             void \*freelist;        /\* sl\[aou\]b first free object \*/
 21             bool pfmemalloc;    /\* If set by the page allocator, 22                          \* ALLOC\_NO\_WATERMARKS was set
 23                          \* and the low watermark was not
 24                          \* met implying that the system
 25                          \* is under some pressure. The
 26                          \* caller should try ensure
 27                          \* this page is only used to
 28                          \* free other pages.
 29                          \*/
 30         };
 31 
 32         union {
 33 #if defined(CONFIG\_HAVE\_CMPXCHG\_DOUBLE) && \\
 34     defined(CONFIG\_HAVE\_ALIGNED\_STRUCT\_PAGE)
 35             /\* Used for cmpxchg\_double in slub \*/
 36             unsigned long counters;
 37 #else
 38             /\*
 39              \* Keep \_count separate from slub cmpxchg\_double data.
 40              \* As the rest of the double word is protected by
 41              \* slab\_lock but \_count is not.
 42              \*/
 43             unsigned counters;
 44 #endif
 45 
 46             struct {
 47 
 48                 union {
 49                     /\*
 50                      \* Count of ptes mapped in
 51                      \* mms, to show when page is
 52                      \* mapped & limit reverse map
 53                      \* searches.
 54                      \*
 55                      \* Used also for tail pages
 56                      \* refcounting instead of
 57                      \* \_count. Tail pages cannot
 58                      \* be mapped and keeping the
 59                      \* tail page \_count zero at
 60                      \* all times guarantees
 61                      \* get\_page\_unless\_zero() will
 62                      \* never succeed on tail
 63                      \* pages.
 64                      \*/
 65                     atomic\_t \_mapcount;
 66 
 67                     struct { /\* SLUB \*/
 68                         unsigned inuse:16;
 69                         unsigned objects:15;
 70                         unsigned frozen:1;
 71                     };
 72                     int units;    /\* SLOB \*/
 73                 };
 74                 atomic\_t \_count;        /\* Usage count, see below. \*/
 75             };
 76             unsigned int active;    /\* SLAB \*/
 77         };
 78     };
 79 
 80     /\* Third double word block \*/
 81     union {
 82         struct list\_head lru;    /\* Pageout list, eg. active\_list 83                      \* protected by zone->lru\_lock !
 84                      \* Can be used as a generic list
 85                      \* by the page owner.
 86                      \*/
 87         struct {        /\* slub per cpu partial pages \*/
 88             struct page \*next;    /\* Next partial slab \*/
 89 #ifdef CONFIG\_64BIT
 90             int pages;    /\* Nr of partial slabs left \*/
 91             int pobjects;    /\* Approximate # of objects \*/
 92 #else
 93             short int pages;
 94             short int pobjects;
 95 #endif
 96         };
 97 
 98         struct slab \*slab\_page; /\* slab fields \*/
 99         struct rcu\_head rcu\_head;    /\* Used by SLAB
100 \* when destroying via RCU
101                          \*/
102 #if defined(CONFIG\_TRANSPARENT\_HUGEPAGE) && USE\_SPLIT\_PMD\_PTLOCKS
103         pgtable\_t pmd\_huge\_pte; /\* protected by page->ptl \*/
104 #endif
105     };
106 
107     /\* Remainder is not double word aligned \*/
108     union {
109         unsigned long private;        /\* Mapping-private opaque data:
110 \* usually used for buffer\_heads
111 \* if PagePrivate set; used for
112 \* swp\_entry\_t if PageSwapCache;
113 \* indicates order in the buddy
114 \* system if PG\_buddy is set.
115                          \*/
116 #if USE\_SPLIT\_PTE\_PTLOCKS
117 #if ALLOC\_SPLIT\_PTLOCKS
118         spinlock\_t \*ptl;
119 #else
120         spinlock\_t ptl;
121 #endif
122 #endif
123         struct kmem\_cache \*slab\_cache;    /\* SL\[AU\]B: Pointer to slab \*/
124         struct page \*first\_page;    /\* Compound tail pages \*/
125     };
126 
127 #ifdef CONFIG\_MEMCG
128     struct mem\_cgroup \*mem\_cgroup;
129 #endif
130 
131     /\*
132 \* On machines where all RAM is mapped into kernel address space,
133 \* we can simply calculate the virtual address. On machines with
134 \* highmem some memory is mapped into kernel virtual memory
135 \* dynamically, so we need a place to store that address.
136 \* Note that this field could be 16 bits on x86 ... ;)
137 \*
138 \* Architectures with slow multiplication can define
139 \* WANT\_PAGE\_VIRTUAL in asm/page.h
140      \*/
141 #if defined(WANT\_PAGE\_VIRTUAL)
142     void \*virtual;            /\* Kernel virtual address (NULL if
143 not kmapped, ie. highmem) \*/
144 #endif /\* WANT\_PAGE\_VIRTUAL \*/
145 
146 #ifdef CONFIG\_KMEMCHECK
147     /\*
148 \* kmemcheck wants to track the status of each byte in a page; this
149 \* is a pointer to such a status block. NULL if not tracked.
150      \*/
151     void \*shadow;
152 #endif
153 
154 #ifdef LAST\_CPUPID\_NOT\_IN\_PAGE\_FLAGS
155     int \_last\_cpupid;
156 #endif
157 }

![复制代码](image/69c5a8ac3fa60e0848d784a6dd461da6.gif)

View Code

**在整个struct page的定义里面的注释对每个位都作了详尽的解释，但我还是觉得有几个重要的定义要重复一下：**

**（1）void\*virtual:页的虚拟地址（由于在64位系统之中C语言里的void\*指针的长度最长为64bit，寻址空间是2^64大远远超出了当前主流微机的硬件内存RAM的大小（8GB，16GB左右）这也就给虚拟空间寻址，交换技术提供了可能性）对virtual中的虚拟地址进行映射需要通过四级页表来进行。**

**（2）pgoff\_t index:这个变量和freelist被定义在同一个union中，index变量被内存管理子系统中的多个模块使用，比如高速缓存。**

**（3）unsigned long flags:flag变量很少有设成long的可见里面的信息量比较大，这里是用来存放页的状态，比如锁/未锁，换出（虚拟内存用），激活等等。**

**再继续说内存管理机制之前，有一点非常重要，就是linux中关于进程和内存之间的对应关系。**

**linux中的每一个进程维护一个PCB，而这个PCB就是/include/linux/sched.h中定义的task\_struct，在这个结构体的定义之中有定义变量：**

**struct mm\_struct \*mm, \*active\_mm;**

**这也就是进程和内存管理的桥梁之一，也是由此可见进程和内存块/页之间的关系是一对多的（考虑进程共享的内存的话是多对多），进程在装入内存的时候，操作系统的工作的实质是将task\_struct中的相关的内存数据映射到部分映射到物理内存之中，而对于并没有映射的页就采取交换技术来解决。和windows系统中的程序装入过程相比较，windows中的程序装入过程都是靠loader完成的，loader的工作就是针对PE格式的可执行文件通过二进制的分析（比如IDT，IAT等等）进行装入，很多情况下一个进程都会被装入到同一个虚拟地址之中0x40000000（90%都是装入这里）。而linux之中，我们的进程是根据调度算法来安排其在虚拟地址之中的分布情况，buddy算法可以将进程的使用的页尽可能整齐地装入（其实这里我有些不是很清楚的地方，linux如果这么动态分配内存那么该如何处理一些动态加载的库的问题，像windows中的dll文件都是通过计算偏移来重定位，而linux会怎么做呢？）进程在已经装入物理内存的页的基础之上开始执行指令，跳转到并未被装入物理内存的页的虚拟地址的时候，会触发一个缺页中断，缺页中断触发页的交换的过程，从而帮助程序继续执行，这也就是虚拟内存的过程。**

![](image/cdec0645add3fc3c328197dda5c76203.gif) ![](image/81178cc93a2a3bb5048d90d76e7ec935.gif)

![复制代码](image/69c5a8ac3fa60e0848d784a6dd461da6.gif)

  1 struct task\_struct {
  2     volatile long state;    /\* -1 unrunnable, 0 runnable, >0 stopped \*/
  3     void \*stack;
  4     atomic\_t usage;
  5     unsigned int flags;    /\* per process flags, defined below \*/
  6     unsigned int ptrace;
  7 
  8 #ifdef CONFIG\_SMP
  9     struct llist\_node wake\_entry;
 10     int on\_cpu;
 11     struct task\_struct \*last\_wakee;
 12     unsigned long wakee\_flips;
 13     unsigned long wakee\_flip\_decay\_ts;
 14 
 15     int wake\_cpu;
 16 #endif
 17     int on\_rq;
 18 
 19     int prio, static\_prio, normal\_prio;
 20     unsigned int rt\_priority;
 21     const struct sched\_class \*sched\_class;
 22     struct sched\_entity se;
 23     struct sched\_rt\_entity rt;
 24 #ifdef CONFIG\_CGROUP\_SCHED
 25     struct task\_group \*sched\_task\_group;
 26 #endif
 27     struct sched\_dl\_entity dl;
 28 
 29 #ifdef CONFIG\_PREEMPT\_NOTIFIERS
 30     /\* list of struct preempt\_notifier: \*/
 31     struct hlist\_head preempt\_notifiers;
 32 #endif
 33 
 34 #ifdef CONFIG\_BLK\_DEV\_IO\_TRACE
 35     unsigned int btrace\_seq;
 36 #endif
 37 
 38     unsigned int policy;
 39     int nr\_cpus\_allowed;
 40     cpumask\_t cpus\_allowed;
 41 
 42 #ifdef CONFIG\_PREEMPT\_RCU
 43     int rcu\_read\_lock\_nesting;
 44     union rcu\_special rcu\_read\_unlock\_special;
 45     struct list\_head rcu\_node\_entry;
 46 #endif /\* #ifdef CONFIG\_PREEMPT\_RCU \*/
 47 #ifdef CONFIG\_PREEMPT\_RCU
 48     struct rcu\_node \*rcu\_blocked\_node;
 49 #endif /\* #ifdef CONFIG\_PREEMPT\_RCU \*/
 50 #ifdef CONFIG\_TASKS\_RCU
 51     unsigned long rcu\_tasks\_nvcsw;
 52     bool rcu\_tasks\_holdout;
 53     struct list\_head rcu\_tasks\_holdout\_list;
 54     int rcu\_tasks\_idle\_cpu;
 55 #endif /\* #ifdef CONFIG\_TASKS\_RCU \*/
 56 
 57 #if defined(CONFIG\_SCHEDSTATS) || defined(CONFIG\_TASK\_DELAY\_ACCT)
 58     struct sched\_info sched\_info;
 59 #endif
 60 
 61     struct list\_head tasks;
 62 #ifdef CONFIG\_SMP
 63     struct plist\_node pushable\_tasks;
 64     struct rb\_node pushable\_dl\_tasks;
 65 #endif
 66 
 67     struct mm\_struct \*mm, \*active\_mm;
 68 #ifdef CONFIG\_COMPAT\_BRK
 69     unsigned brk\_randomized:1;
 70 #endif
 71     /\* per-thread vma caching \*/
 72     u32 vmacache\_seqnum;
 73     struct vm\_area\_struct \*vmacache\[VMACACHE\_SIZE\];
 74 #if defined(SPLIT\_RSS\_COUNTING)
 75     struct task\_rss\_stat    rss\_stat;
 76 #endif
 77 /\* task state \*/
 78     int exit\_state;
 79     int exit\_code, exit\_signal;
 80     int pdeath\_signal;  /\*  The signal sent when the parent dies  \*/
 81     unsigned int jobctl;    /\* JOBCTL\_\*, siglock protected \*/
 82 
 83     /\* Used for emulating ABI behavior of previous Linux versions \*/
 84     unsigned int personality;
 85 
 86     unsigned in\_execve:1;    /\* Tell the LSMs that the process is doing an 87 \* execve \*/
 88     unsigned in\_iowait:1;
 89 
 90     /\* Revert to default priority/policy when forking \*/
 91     unsigned sched\_reset\_on\_fork:1;
 92     unsigned sched\_contributes\_to\_load:1;
 93 
 94 #ifdef CONFIG\_MEMCG\_KMEM
 95     unsigned memcg\_kmem\_skip\_account:1;
 96 #endif
 97 
 98     unsigned long atomic\_flags; /\* Flags needing atomic access. \*/
 99 
100     pid\_t pid;
101     pid\_t tgid;
102 
103 #ifdef CONFIG\_CC\_STACKPROTECTOR
104     /\* Canary value for the -fstack-protector gcc feature \*/
105     unsigned long stack\_canary;
106 #endif
107     /\*
108 \* pointers to (original) parent process, youngest child, younger sibling,
109 \* older sibling, respectively.  (p->father can be replaced with
110 \* p->real\_parent->pid)
111      \*/
112     struct task\_struct \_\_rcu \*real\_parent; /\* real parent process \*/
113     struct task\_struct \_\_rcu \*parent; /\* recipient of SIGCHLD, wait4() reports \*/
114     /\*
115 \* children/sibling forms the list of my natural children
116      \*/
117     struct list\_head children;    /\* list of my children \*/
118     struct list\_head sibling;    /\* linkage in my parent's children list \*/
119     struct task\_struct \*group\_leader;    /\* threadgroup leader \*/
120 
121     /\*
122 \* ptraced is the list of tasks this task is using ptrace on.
123 \* This includes both natural children and PTRACE\_ATTACH targets.
124 \* p->ptrace\_entry is p's link on the p->parent->ptraced list.
125      \*/
126     struct list\_head ptraced;
127     struct list\_head ptrace\_entry;
128 
129     /\* PID/PID hash table linkage. \*/
130     struct pid\_link pids\[PIDTYPE\_MAX\];
131     struct list\_head thread\_group;
132     struct list\_head thread\_node;
133 
134     struct completion \*vfork\_done;        /\* for vfork() \*/
135     int \_\_user \*set\_child\_tid;        /\* CLONE\_CHILD\_SETTID \*/
136     int \_\_user \*clear\_child\_tid;        /\* CLONE\_CHILD\_CLEARTID \*/
137 
138     cputime\_t utime, stime, utimescaled, stimescaled;
139     cputime\_t gtime;
140 #ifndef CONFIG\_VIRT\_CPU\_ACCOUNTING\_NATIVE
141     struct cputime prev\_cputime;
142 #endif
143 #ifdef CONFIG\_VIRT\_CPU\_ACCOUNTING\_GEN
144     seqlock\_t vtime\_seqlock;
145     unsigned long long vtime\_snap;
146     enum {
147         VTIME\_SLEEPING = 0,
148         VTIME\_USER,
149         VTIME\_SYS,
150     } vtime\_snap\_whence;
151 #endif
152     unsigned long nvcsw, nivcsw; /\* context switch counts \*/
153     u64 start\_time;        /\* monotonic time in nsec \*/
154     u64 real\_start\_time;    /\* boot based time in nsec \*/
155 /\* mm fault and swap info: this can arguably be seen as either mm-specific or thread-specific \*/
156     unsigned long min\_flt, maj\_flt;
157 
158     struct task\_cputime cputime\_expires;
159     struct list\_head cpu\_timers\[3\];
160 
161 /\* process credentials \*/
162     const struct cred \_\_rcu \*real\_cred; /\* objective and real subjective task
163 \* credentials (COW) \*/
164     const struct cred \_\_rcu \*cred;    /\* effective (overridable) subjective task
165 \* credentials (COW) \*/
166     char comm\[TASK\_COMM\_LEN\]; /\* executable name excluding path
167 - access with \[gs\]et\_task\_comm (which lock
168 it with task\_lock())
169 - initialized normally by setup\_new\_exec \*/
170 /\* file system info \*/
171     int link\_count, total\_link\_count;
172 #ifdef CONFIG\_SYSVIPC
173 /\* ipc stuff \*/
174     struct sysv\_sem sysvsem;
175     struct sysv\_shm sysvshm;
176 #endif
177 #ifdef CONFIG\_DETECT\_HUNG\_TASK
178 /\* hung task detection \*/
179     unsigned long last\_switch\_count;
180 #endif
181 /\* CPU-specific state of this task \*/
182     struct thread\_struct thread;
183 /\* filesystem information \*/
184     struct fs\_struct \*fs;
185 /\* open file information \*/
186     struct files\_struct \*files;
187 /\* namespaces \*/
188     struct nsproxy \*nsproxy;
189 /\* signal handlers \*/
190     struct signal\_struct \*signal;
191     struct sighand\_struct \*sighand;
192 
193     sigset\_t blocked, real\_blocked;
194     sigset\_t saved\_sigmask;    /\* restored if set\_restore\_sigmask() was used \*/
195     struct sigpending pending;
196 
197     unsigned long sas\_ss\_sp;
198     size\_t sas\_ss\_size;
199     int (\*notifier)(void \*priv);
200     void \*notifier\_data;
201     sigset\_t \*notifier\_mask;
202     struct callback\_head \*task\_works;
203 
204     struct audit\_context \*audit\_context;
205 #ifdef CONFIG\_AUDITSYSCALL
206     kuid\_t loginuid;
207     unsigned int sessionid;
208 #endif
209     struct seccomp seccomp;
210 
211 /\* Thread group tracking \*/
212        u32 parent\_exec\_id;
213        u32 self\_exec\_id;
214 /\* Protection of (de-)allocation: mm, files, fs, tty, keyrings, mems\_allowed,
215 \* mempolicy \*/
216     spinlock\_t alloc\_lock;
217 
218     /\* Protection of the PI data structures: \*/
219     raw\_spinlock\_t pi\_lock;
220 
221 #ifdef CONFIG\_RT\_MUTEXES
222     /\* PI waiters blocked on a rt\_mutex held by this task \*/
223     struct rb\_root pi\_waiters;
224     struct rb\_node \*pi\_waiters\_leftmost;
225     /\* Deadlock detection and priority inheritance handling \*/
226     struct rt\_mutex\_waiter \*pi\_blocked\_on;
227 #endif
228 
229 #ifdef CONFIG\_DEBUG\_MUTEXES
230     /\* mutex deadlock detection \*/
231     struct mutex\_waiter \*blocked\_on;
232 #endif
233 #ifdef CONFIG\_TRACE\_IRQFLAGS
234     unsigned int irq\_events;
235     unsigned long hardirq\_enable\_ip;
236     unsigned long hardirq\_disable\_ip;
237     unsigned int hardirq\_enable\_event;
238     unsigned int hardirq\_disable\_event;
239     int hardirqs\_enabled;
240     int hardirq\_context;
241     unsigned long softirq\_disable\_ip;
242     unsigned long softirq\_enable\_ip;
243     unsigned int softirq\_disable\_event;
244     unsigned int softirq\_enable\_event;
245     int softirqs\_enabled;
246     int softirq\_context;
247 #endif
248 #ifdef CONFIG\_LOCKDEP
249 # define MAX\_LOCK\_DEPTH 48UL
250     u64 curr\_chain\_key;
251     int lockdep\_depth;
252     unsigned int lockdep\_recursion;
253     struct held\_lock held\_locks\[MAX\_LOCK\_DEPTH\];
254     gfp\_t lockdep\_reclaim\_gfp;
255 #endif
256 
257 /\* journalling filesystem info \*/
258     void \*journal\_info;
259 
260 /\* stacked block device info \*/
261     struct bio\_list \*bio\_list;
262 
263 #ifdef CONFIG\_BLOCK
264 /\* stack plugging \*/
265     struct blk\_plug \*plug;
266 #endif
267 
268 /\* VM state \*/
269     struct reclaim\_state \*reclaim\_state;
270 
271     struct backing\_dev\_info \*backing\_dev\_info;
272 
273     struct io\_context \*io\_context;
274 
275     unsigned long ptrace\_message;
276     siginfo\_t \*last\_siginfo; /\* For ptrace use.  \*/
277     struct task\_io\_accounting ioac;
278 #if defined(CONFIG\_TASK\_XACCT)
279     u64 acct\_rss\_mem1;    /\* accumulated rss usage \*/
280     u64 acct\_vm\_mem1;    /\* accumulated virtual memory usage \*/
281     cputime\_t acct\_timexpd;    /\* stime + utime since last update \*/
282 #endif
283 #ifdef CONFIG\_CPUSETS
284     nodemask\_t mems\_allowed;    /\* Protected by alloc\_lock \*/
285     seqcount\_t mems\_allowed\_seq;    /\* Seqence no to catch updates \*/
286     int cpuset\_mem\_spread\_rotor;
287     int cpuset\_slab\_spread\_rotor;
288 #endif
289 #ifdef CONFIG\_CGROUPS
290     /\* Control Group info protected by css\_set\_lock \*/
291     struct css\_set \_\_rcu \*cgroups;
292     /\* cg\_list protected by css\_set\_lock and tsk->alloc\_lock \*/
293     struct list\_head cg\_list;
294 #endif
295 #ifdef CONFIG\_FUTEX
296     struct robust\_list\_head \_\_user \*robust\_list;
297 #ifdef CONFIG\_COMPAT
298     struct compat\_robust\_list\_head \_\_user \*compat\_robust\_list;
299 #endif
300     struct list\_head pi\_state\_list;
301     struct futex\_pi\_state \*pi\_state\_cache;
302 #endif
303 #ifdef CONFIG\_PERF\_EVENTS
304     struct perf\_event\_context \*perf\_event\_ctxp\[perf\_nr\_task\_contexts\];
305     struct mutex perf\_event\_mutex;
306     struct list\_head perf\_event\_list;
307 #endif
308 #ifdef CONFIG\_DEBUG\_PREEMPT
309     unsigned long preempt\_disable\_ip;
310 #endif
311 #ifdef CONFIG\_NUMA
312     struct mempolicy \*mempolicy;    /\* Protected by alloc\_lock \*/
313     short il\_next;
314     short pref\_node\_fork;
315 #endif
316 #ifdef CONFIG\_NUMA\_BALANCING
317     int numa\_scan\_seq;
318     unsigned int numa\_scan\_period;
319     unsigned int numa\_scan\_period\_max;
320     int numa\_preferred\_nid;
321     unsigned long numa\_migrate\_retry;
322     u64 node\_stamp;            /\* migration stamp  \*/
323     u64 last\_task\_numa\_placement;
324     u64 last\_sum\_exec\_runtime;
325     struct callback\_head numa\_work;
326 
327     struct list\_head numa\_entry;
328     struct numa\_group \*numa\_group;
329 
330     /\*
331 \* numa\_faults is an array split into four regions:
332 \* faults\_memory, faults\_cpu, faults\_memory\_buffer, faults\_cpu\_buffer
333 \* in this precise order.
334 \*
335 \* faults\_memory: Exponential decaying average of faults on a per-node
336 \* basis. Scheduling placement decisions are made based on these
337 \* counts. The values remain static for the duration of a PTE scan.
338 \* faults\_cpu: Track the nodes the process was running on when a NUMA
339 \* hinting fault was incurred.
340 \* faults\_memory\_buffer and faults\_cpu\_buffer: Record faults per node
341 \* during the current scan window. When the scan completes, the counts
342 \* in faults\_memory and faults\_cpu decay and these values are copied.
343      \*/
344     unsigned long \*numa\_faults;
345     unsigned long total\_numa\_faults;
346 
347     /\*
348 \* numa\_faults\_locality tracks if faults recorded during the last
349 \* scan window were remote/local. The task scan period is adapted
350 \* based on the locality of the faults with different weights
351 \* depending on whether they were shared or private faults
352      \*/
353     unsigned long numa\_faults\_locality\[2\];
354 
355     unsigned long numa\_pages\_migrated;
356 #endif /\* CONFIG\_NUMA\_BALANCING \*/
357 
358     struct rcu\_head rcu;
359 
360     /\*
361 \* cache last used pipe for splice
362      \*/
363     struct pipe\_inode\_info \*splice\_pipe;
364 
365     struct page\_frag task\_frag;
366 
367 #ifdef    CONFIG\_TASK\_DELAY\_ACCT
368     struct task\_delay\_info \*delays;
369 #endif
370 #ifdef CONFIG\_FAULT\_INJECTION
371     int make\_it\_fail;
372 #endif
373     /\*
374 \* when (nr\_dirtied >= nr\_dirtied\_pause), it's time to call
375 \* balance\_dirty\_pages() for some dirty throttling pause
376      \*/
377     int nr\_dirtied;
378     int nr\_dirtied\_pause;
379     unsigned long dirty\_paused\_when; /\* start of a write-and-pause period \*/
380 
381 #ifdef CONFIG\_LATENCYTOP
382     int latency\_record\_count;
383     struct latency\_record latency\_record\[LT\_SAVECOUNT\];
384 #endif
385     /\*
386 \* time slack values; these are used to round up poll() and
387 \* select() etc timeout values. These are in nanoseconds.
388      \*/
389     unsigned long timer\_slack\_ns;
390     unsigned long default\_timer\_slack\_ns;
391 
392 #ifdef CONFIG\_FUNCTION\_GRAPH\_TRACER
393     /\* Index of current stored address in ret\_stack \*/
394     int curr\_ret\_stack;
395     /\* Stack of return addresses for return function tracing \*/
396     struct ftrace\_ret\_stack    \*ret\_stack;
397     /\* time stamp for last schedule \*/
398     unsigned long long ftrace\_timestamp;
399     /\*
400 \* Number of functions that haven't been traced
401 \* because of depth overrun.
402      \*/
403     atomic\_t trace\_overrun;
404     /\* Pause for the tracing \*/
405     atomic\_t tracing\_graph\_pause;
406 #endif
407 #ifdef CONFIG\_TRACING
408     /\* state flags for use by tracers \*/
409     unsigned long trace;
410     /\* bitmask and counter of trace recursion \*/
411     unsigned long trace\_recursion;
412 #endif /\* CONFIG\_TRACING \*/
413 #ifdef CONFIG\_MEMCG
414     struct memcg\_oom\_info {
415         struct mem\_cgroup \*memcg;
416         gfp\_t gfp\_mask;
417         int order;
418         unsigned int may\_oom:1;
419     } memcg\_oom;
420 #endif
421 #ifdef CONFIG\_UPROBES
422     struct uprobe\_task \*utask;
423 #endif
424 #if defined(CONFIG\_BCACHE) || defined(CONFIG\_BCACHE\_MODULE)
425     unsigned int    sequential\_io;
426     unsigned int    sequential\_io\_avg;
427 #endif
428 #ifdef CONFIG\_DEBUG\_ATOMIC\_SLEEP
429     unsigned long    task\_state\_change;
430 #endif
431 };

![复制代码](image/69c5a8ac3fa60e0848d784a6dd461da6.gif)

View Code

**愚蠢的问题1：**

**MMU是由硬件实现的专门为解决虚拟地址和物理地址映射问题而设计的部件，那么为什么要在linux的源代码中体现呢？为什么在要在软件中再描述一次呢？**

**虚拟地址到物理地址的映射，（目前而讲）需要4级页表索引的访问来完成。在mm\_struct结构体中的定义之中有一个pdg\_t类型的指针名叫pgd（PageGlobalDirectory），由此出发继续向下级访问有pud（PageUpperDirectory）pmd（PageMiddleDirectory）pte（PageTableEntry），最后一级是具体的页表很遗憾的是，我暂时没有在3.19内核的源码中找到关于pte\_t的定义，但是根据书籍上的描述应该是一个指向struct page数组的指针。**

**于是我们可以这样总结，程序在执行的过程会有大量的跳转的过程，而每次的跳转需要一个操作数即地址，这个地址是一个虚拟地址，然后根据该虚拟地址进行MMU的操作，过程中得到一个页表，首先根据页表判断该页是否已经存在于物理内存中，如果不是的话则进行一次交换的操作，上文已经阐述过该过程，页交换完成之后，寻址过程就得以继续进行了，此时使用相同的虚拟地址访问到的是另一个物理页面，即交换进入的物理页面。**

**愚蠢的问题2：**

**虚拟内存的机制像是把物理内存和外部存储容量共同地址编码，这个共同的编码就是虚拟地址，所谓“编码”过程不一定是顺序一对一的，但是虚拟地址和页表的索引之间一定是个满射关系。**

**这是我最初对于虚拟内存机制的理解，表面看起来没有什么问题，可还是当考虑每个进程的寻址空间独立性的时候就会发现问题，相同的地址在两个进程中映射外部地址应该可以是不相同的，可是一旦将他们看作共同地址编码，就不会有相同的逻辑地址映射到不同的物理地址这回事了。**

**其实答案很简单一句话：每个进程维护一个页表 !**

**最后一张大图概括一下上文**



## 参考

[内存管理模型-CSDN博客](https://blog.csdn.net/zdy0_2004/article/details/45769861)