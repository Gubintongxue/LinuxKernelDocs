好的那首先欢迎各位同志们啊
来到铃声学院
我是威克
今天晚上我们主要是给大家分享的是LINUX内核
内存管理的一大奖
里面分为四个点进行学习
第一个就是我们所讲的内存泄露与战役出
第二个呢是虚拟地址布局及内存映射
第三是物理内存组织模型
其引导内核分配器
第四是火旺分配器和块分配器
从四个点进行分析
那首先我们来看第一个问题啊
那第一个问题呢
就是我们所讲LINUX的一个什么内存管理的架构
在这个过程当中
它有一个什么呢
专门有一个什么内存管理的一个子系统
内存管理系子系统的一个架构啊
我们看一下这个架构啊
怎么回事呢
它可以分为什么啊
它可以分为这个用户空间
还有一个是什么呢
内核空间和什么和这个硬件啊
硬件这个层面主要分为三个部分
那现在我们首先通过一个图形进行全部的整体
理解一下
你看啊
这有点慢是吧
刷新一下
等待一下可以了
那现在在做啊
各位同志们
大家所看到的一个图形
它就是什么呢
就是内存管理系统的一个价格
那么这个架构现在我就将这个图形啊
这个架构图直接放到我们的笔记当中
那么后面来的各位同志们啊
就直接找谁呢
找这个一老师来领取这个资料就可以了
你看第一个就出来了
但是现在呢我们从这个图形我们可以看到啊
你看上面是我们的用户空间
那中间这一块是内核空间
那再往下面就是硬件层面
就是机器硬件了
那首先我们看的是第一个问题是吧
首先我们从上面开始讲啊
上面是什么呢
就是用户空间
那这个用户空间它是怎么一回事呢
你看到的比如应用程序一
应用程序二
应用程序三
一直到应用程序N是吧
所以在这里就告诉你了
这个应用程序啊
它使用什么
直接使用这个MELOO这个函数是吧
和什么呢
和这一个MC函数和什么
直接来申请这个内存空间是吧
嗯那么它使用什么呢
你看使用这个frail
对不对啊
使用这个frail干什么呢
使用frame这个函数来释放什么
释放你所分配的内存空间
哎这个我们完全就可以搞定了
是不是
所以这个非常简单啊
这第一个我们就解决了第二个什么呢
还有一个点这个MELOC函数和FRAR
它是GLABEC的一个分配器
这个我没搞清楚了
对不对
所以接下来他就从这个地方下手了
什么情况下手呢
就是我们经常写代码分申请空间
MELOC和什么呢
和这个frail这个函数它是什么
是我们所讲g level c库的一个什么内存分配器
那这个内存分配机制是PMELOO
你看到没有注意这个什么呢
MACO和frail这个应用程序调用的接口
下面是对应什么
用PTMACO啊
所提供这个分配器
它所提供嗯所提供的一个什么接口
就这个意思
那么我们所使用什么呢
你所使用的PPT Mac使用什么
使用这个系统调用
那么系统就装了一个什么
你看下面有个SYSBRK
你看到没有啊
所以他装了个BRK
下滑BRK或者可以用什么
可以用这个map啊
这两个可以用这个map
看到没有啊
直接像什么
你看啊
系统调用它系统应用层调用这个使用macle frail
MACO下面对应系统里面就有个什么
有个这个bro和这个IMAP来进行分
像什么呢
向内核申请内申请空间
就这个意思
直接向内核向内核以什么呢
业为单位嗯
干什么呢
申请这个内存空间
就这个意思好
那么得到运行空间之后呢
然后呢他就是什么呢
然后就划分成什么
划分成小块
就是划分成什么呢
小内存块啊
直接分配给什么
分配给这个应用程序
看他的思路就是这个来是吧
那这就是我们理解这个啊
那么接着什么呢
接着就往后面了
还有什么用户空间
它这个内存分配器就除了什么呢
除了刚刚我们讲这个g label c这个库以外
还有了是不是
所以我们针对什么
针对这个用户空间
用户空间的内存分配器啊
除什么呢
除这个哎
除这个g label c是吧
除g label c这个库
这个PTMACC除他以外还有什么呢
还有这个谷歌啦
谷歌是谷歌公司所提供的一个什么TCMELOE
对不对
和什么呢
和free b嗯
SD对不对
那BSD它的一个什么j Mac
他就是专门每个有每个的一套啊
各有各的一套
各有各的一套方案
那么各位大家要理解就行了
好那现在我问各位同志们啊
那现在我所讲的每一句话所讲的内容
大家能不能听得清楚
如果你听得清楚啊
请给我打一个啊
六是吧
听得清楚1.6就说如果听不清
到时候我录制这个视频之后再返回发给你们
你们就听不清这么麻烦啊
都听不到是吧
听清楚就非常好啊
非常感谢各位同志们是吧
好那现在我们就继续了啊
嗯好现在是这样的
那么这个什么这个用户空间我们就解决了是吧
那接下来你看啊
用户空间我们全部都解释完毕了
那下面是一个什么是一个内核空间
你看内核空间里面有什么虚拟内存管理啊
啊内存控制组啊
内存碎片整理啊
液回收啊
内存耗尽杀手啊
页表管理啊
就T2B嘛
还有什么啊
这个页分配器啊
你比这个引导内存分配器
有块分配器
不联系页分配器
每处理器内存分配器还有个什么呢
还有那个页错误异常处理
对不对等等啊
这里我就不在了解了
核心点我们全部搞定了
那现在我们来看第二点是吧
我们把这个字啊稍微把它变大一点
这也没办法变大一点
字体这个字体没有大小算没打小算是吧
那我们就接着来第二个了
到时候再打开文档时候给他告诉大家啊
第二个我们看内核空间
内核空间
那首先第一个问题我们要掌握的是什么呢
是内核空间的一个什么基本功能
那内核空间的这个基本功能是哪些呢
很简单
你比如刚刚讲了这个虚拟内存管理啊
虚拟内存管理它主要是什么呢
负责从这个进程的一个虚拟地址空间
该怎么来分配这个虚拟页啊
那么就会调用这个函数
没有
他有个SYS下划线不可
这里面本来有个下划线
大家注意
因为这个字体比较大
他就没看到下划线是吧
其实他是这样的
可以看这个图形的
这里面有个下划线
划线没那个下划线
那我把它变小一点
你可以看到
你看系统内核里面前面加了个S5S
你看没看
你看S5S下划线就加在前面了啊
那现在我们就回到我们的笔记里面来是吧
那么通过什么呢
就是用这个S5S下划线B2K对不对
用它这个函数来干什么呢
直接用它扩大或者是什么呢
来收缩这个堆
因为你分配空间嘛
再通过这个system里面什么IMAP用来什么
用来在这个内存映射区域啊
内存啊
这个映射区域来分配什么呢
分配这个虚拟
那么干什么呢
用这个S5S下划线
你看啊这个mu map用这个函数来什么
直接通过它来释放这个血液
他就来做这个事情
但是我们还要知道因为什么呢
因为LINUX内核对不对
内核它使用什么
使用一个延迟啊
你要注意啊
他使用这个延迟分配这个什么呢啊
物理内存的一个策略问题
是不是比如这个进程他第一次来访问什么呢
来访问这个虚拟页的时候
他会怎么做
这个时候我们就要触发了
是不是触发什么触发一错一长
对不对
那这个页错误异常处理
从下他直接从什么呢
直接从业分配器申请什么呢
申请这个申请这个物理页
那么我们在什么呢
你直接在这个进程啊
在进程的一个页表当中
直接把虚拟什么虚拟页直接映射到什么
映射到对应的一个物理
你看诶他是不是就解开了
是不是
这是一个点
我们就摸清楚了
是不是
那我摸清这个关系之后
你看啊
嗯下面干什么呢
下面一个就是说你比如那个液分配器
那么他负责什么呢
负责分配物理也要搞清楚啊
液分配器是负责分配物理页
那么当前注意啊
当前我们所使用这个液分配器
它是一个什么
是一个非常出名的叫做什么叫做火伴分配器
就这个意思
那我们这个就摸清楚了
是不是哎那我们摸清楚了就没问题了
那就往下走了
是不是好
往下面我们看啊
他有个什么问题呢
啊那接下来就说你这个内核空间
它提供把这个页啊划分成小块内存啊
这个块分配器来块进行分配
它主要是做这个事情
就是内核空间
它所提供什么呢
提供把这个页啊进行什么
划分成小内存块来分配的什么呢
分配的快分配器进行处理
那么提供分配的这提供分配的这个内存呢
它这个接口是什么
接口是K这个什么呢
MELOG和释放啊
嗯和释放什么呢
和释放内存的接口叫什么呢
CFO就这个意思
那么它能够支持什么呢
它能够支持三种啊
一般是支持三种分配器
那后面我会讲被DOM去讲啊
比如这个slab分配器算法怎么做是吧
嗯除了slab之后呢
还有什么呢
还有snob这个分配器嘛
啊除了snob呢
还有一个什么呢
和这个logo是吧
这个分配器主要是这三个是吧
好那我们搞清这三个关系之后
但是你还知道它在什么呢
就是在我们系统初始化
就是内核初始化的过程当中
那么这个页分配器呢还没有什么呢
如果页分配器还没有准备好
那这个时候需要什么
需要使用一个
需要使用一个什么呢
需要使用一个临时的这个临时的引导啊
这个内存分配器来分配什么呢
来分配这个内存
你看他来做的这个事情
整个思路啊
我们现在全部给打开了
是不是哎打开这个思路说
那接下来我们就下一个了
对不对
那这是第一个功能
就是内核的基本功能
另外一个内核它还有个什么
还专门有一个内核空间的一个什么扩展功能
他做什么呢
就是相当于不联系液分配器
它所提供的什么
所提供分配内存的接口就是什么
VMACO又有这个函数啊
和什么呢
和释放什么
释放这个内存的接口
释放内存接口叫做什么呀
We fell
你看它是一个对应关系的啊
哎那么在内存中在内存碎片化的时候嗯
那么就会申请什么
申请一个连续的一个物理业的成功率啊
它是很低的
那为什么呢
我们就往后面看
是因为什么我们可以申请什么
我们可以申请这个不联系的这个什么物理啊
映射到什么呢
映射到连续的虚拟页嗯
G就是我们所讲的虚拟地址连续
而且它这个物理地址呢它是什么
物理地址是不连续的
你看他那个石头将这个来A那我搞清楚之后呢
还有一个点就是说它有一个什么
就是每一个处理器内存分配器是用来什么
为每处理器分配内存空间
它主要是做这些事情
那我们对啊
这些问题我们搞清楚就很简单
就相当于就相当于每处理器内存分配器
买处理器
这个注意啊
内存分配器诶
它是用来什么呢
用来为每一个处理器变量跟什么分配内存
他做了这个事情
那么还有一个什么呢
我们注意专门有一个什么
比如联系内存分配器主要是干什么的啊
就是我们所讲的这个连续内存分配器
它用来给什么呢
用来给这个驱动程序啊
直接预留一个什么呢
一留一段这个预留一段连续的这个内存
对不对
那么当什么呢
当我们所讲那个驱动程序啊
不用的时候启动程序啊
他不用的时候
那怎么办呢
我们就可以给什么给进程去使用
这是一个点是吧
那么当时呢当驱动程序对不对
他需要使用的这个需要使用这个内存的时候呢
哎那我们就把这个什么
把进程刚刚所占用的这个内存通过什么呢
通过回收的方式
或者是通过这个迁移的方式
来把那个内存呢让出来啊
run出来之后给谁呢
给这个驱动程序run出来给啊
嗯嗯run出来给这个给这个嗯
给驱动程序OK给驱动程序使用它就这个意思
但是我们回头过来看啊
这里面还有个内存控制组
你发现没有内存控制组啊
那么这里所讲的个什么呢
内存控制组
那么它是用来什么
专门用来控制什么控制进程所占用的内存资源
就这个意思
那么就说当内存了碎片化的时候
那我们找不到什么呢
你找不到连续的这个物理页是吧
内存碎片整理
就是通过迁移的方式得到连续的这个物理页
那么在内存不足的时候
那么液回收他负责回收物理液
对没有后备存储设备知识的这个域名页
他把什么把数据放出到交换区
然后呢释放物理
我然后就释放物理
对他有这些后备存储设备知识的
这个什么文件页的话
那么就把这个数据啊写回到存储设备
然后就释放这个物理液了
他就这个意思
那如果这个液在回收失败了
他就使用最后一招就是什么呢
就是我们所讲的选择进程杀掉
就是内存耗尽杀手啊
这个第二个就是内核空间
你看这个中间这个模块我们就全部一开
全部解决了是吧
好现在我们就来看第几个
来看第三个了是吧
好第三个就是我们所讲的硬件硬件这个层面啊
好下面看硬件层面啊
嗯硬件层面是什么意思啊
这个不难啊
其实我们都知道嗯
硬件层面就下面各位来看没有硬件
硬件这个层面就是机器硬件里面有什么
有处理器和物理内存
看到没有
那现在我们就分头下面来啊
这里是什么呢
你比如这个硬件层面
你比这个处理器处理器啊
它就包含一个什么
包含一个哎称为什么呢
称为内存管理单元
内存管理单元这个部件是不是嗯
他主要负责什么
负责直接把这个什么把这个虚拟地址啊
把虚拟地址转换啊
转换成什么呢
转换成这个物理地址
他就这个意思
那现在我们来看把这个弄一下啊
OK嗯搞清楚了是吧
那怎么转换呢
那么这个内存就是我们所讲内存管理单元啊
叫做内存管理单元
那么他专门有个叫什么叫做MMU
要注意啊
那它就是什么呢
就是memory这个单词想要上什么呢
MAGIMENT有一个unit这三个单词进行组合而成
就这个意思
但是呢我们要知道这个内存呢
你这个内存管理单元它就包含一个什么
包含一个这个页表缓存
对不对
页表缓存的对页表缓存的一个嗯嗯嗯
页表缓存的一个部件是不是那页表反向的部件
那我们叫做什么叫做这个TLB是吧
TLB那英文简写翻译嘛
就是trust类型是吧啊
lock excite还有个什么Buff
他就这个意思啊
就这个意思
那么这个搞清之后呢
你再往后面是因为什么这个TLB啊
这个页表缓存它干什么
它是用来保存啊
就是用来保存我们最近啊
所使用过的这个什么啊
使用过的这个页表映射
对不对
那主要呢它是避免什么
避免你每次直接把这个虚拟地址干什么转换啊
他直接把那个虚拟地址转换成这个物理地址
都需要什么
都需要查询这个内存当中的一个什么也表
就是避免这个问题
那为什么这么做
原因非常简单
他提供告诉我们
就是说它主要是为了什么呢
他为了解决处理器的一个什么执行速度啦啊
这处理器的执行速度和什么
和内存的一个访问速度不匹配的那个什么问题
为什么呢
所以它在处理器和内存之间嗯
就增加了一个什么
增加了这个缓存
对不对
那返程它通常就可以分为什么呢
比如一级缓存和什么呢
和二级缓存就这个意思
那为了什么呢
啊为了支持这个运行的取值令和什么呢
和取数据
所以他这个一级缓存它又分为什么
分为数据缓存和指定方程
就按这个思路来
完了之后呢
那你看啊一级缓存
二级缓存是吧
除此以外啊
这个我们要摸清楚了
是不是啊
那我们搞清这个关系之后
后面还有一个什么
你看后面就是我们对应的代码点了
是不是第八点跟咱们把这个相对来讲啊啊啊
啊点慢是不是换换就这样吧
嗯现在我们来看第二大点
那第二大点我们注意啊
就是虚拟地址空间的一个布局来看笔记啊
笔记等下课时会发给大家
是不是大家直接什么
直接找这个一老师来领取这个笔记就可以了啊
因为今天晚上我们所讲这个内容吧
它主要是什么呢
我是针对这个就是说你具备一定的开发经验
然后再研究这个LINUX内核
是不是为了我们更好的作用层开发
和自己的技术提升啊
那么如果啊告诉大家非常简单
因为我们做什么
我们大家啊都是做这个什么
如果你做这个后台
是不是你做后台的这个服务器研发
是不是你掌握这个什么呢
就是相当于我们掌握这个内核之后
那下一步那我们再进一步什么
进一步研究这个pass是不是这样就更加方便了
因为这个是其内核这一块啊
基础上架构的就是我们目前所推出的非常火
说如果我们不具备内核的基础来学这个东西
基本上难难上加难
就很难看得懂
很难看得懂
因为里面东西很深很深
OK好
那现在我们来看一下
是不是那么研究技术面分析比较多
应用安全这一块它也是从内核下手的
那么现在我们来看第一个第一个是什么呢
我们针对这个虚拟要注意啊
现在我们来看一下这个虚拟虚拟什么呢
第一个要掌握的一个点
就是虚拟地址空间的一个什么
虚拟地址空间的划分对啊
到目前为止
这些应用程序啊
没有那么大的一个什么内存需求要注意啊
为什么呢
因为现在内存太大了
是不是因为什么呢
就是因为目前啊按照目前情况来看
是目前这个应用程序他没有什么呢
没有那么大的一个什么内存的一个需求
所以我们经常讲这个arm64
是不是arm64这个处理器
它不支持完全的什么呢
64位的一个虚拟地址
那么实际上它所支持的情况下
来我们来看一下啊
那情况呢它是这样的
你比如这个我们看第一个啊
第一个是什么
就是虚拟地址了
虚拟地址的一个最大最大宽度
最大宽度它是多少位啊
是48位
是不是48位就很大了
是不是啊
它不是一般的大啊
大到48位了
我们看一下是吧
你看那48位啊
48位
那就相当于什么呢
你比如这个内核啊
我把这个图搞出来
要么就画好了
看到没有
在这
那没有是吧
我们把这个标出来啊
没关系
是这样的
你比如它有内核地址空间
有不规范地址空间不允许使用
还有什么呢
还有就是用户地址空间
它从内部开始
从这里开始看到没有
他就0XX0X后面是什么呢
16个F是吧
到什么呢
直接到这个项链上来
立方其实是0X4F后面有12个零
看
他就这个意思啊
接着到下面这个不规范地址
这个用户空间它是怎么计算的
用空间从另方开始卡
哎这个用户空间它是什么呢
是0X嗯
X后面是几个F呢
他就这个意思是吧
从这开始啊
然后到什么呢
到后面0000
后面是16个零了
这地方就是六个零
嗯就这么简单啊
所以现在我将这个图形
直接把它放到我们的笔记当中
这样的话我们就便于学习
要注意这一点是
这图片放的也很简单是吧
那所以现在我们所看到
那现在你看到什么这个图形
这个内核你看啊
内核内核内核虚拟地址在什么
在我们这个64位啊
地址空间当中的那个什么
直接在这个顶部了
注意啊
内核在顶部是吧
内核在顶部啊
那么它的高16位
注意啊
算最上面嘛
高16位
它前是什么
前是一那范围
刚才我们讲了
他直接从什么
直接从这开始了
就是X是吧
四个F后面是嗯对吧
嗯到哪里呢
到这个0X0X是什么呢
后面是16个F
你看到下面是吧
一是六个F啊
大家注意了
1234
就这么简单是吧
这是这是我们所讲的啊
看清楚啊
这个是什么呢
这个是内核的啊
内核的一个什么地址空间在上面
他从0X啊
这个四个F后面12个零到什么
来到0X后面是前10FF是吧
16个F就是啊
我们所讲的内核内核地址空间啊
哎那么除了内核地址空间之后呢
还有什么
还有下面呢
下面是用户地址空间
这不很容易嘛
对不对
好
那么下面就是一个用户虚拟地址空间
用户虚拟地址在什么
它在64位的这个地址空间
当中的一个什么底部
对不对
在最底下面啊
就相当于什么高16位是吧
嗯高16位
他钱是什么
钱是零
对不对
那么这个范围那就根据这个来试验呢
范围是从几开始
是从这个零看啊
范围是从0X有没有
16个零啊对啊
这就出来了
然后逗号后面有个什么到0X
嗯就这个意思重现出来
这个表示什么呢
就全部出来了
但是要注意了
那么这里所讲的啊
上面是内核地址空间
下面是用户地址空间
那么在整个虚拟地址空间布局的话
最顶部在底部的话呢
它就是内核几次空间
最下面就是用户地址空间
那么他们的地址空间的范围
我们已经全部表示出来了
但是我们还高16位
是不是那么高
16位
他钱是什么呢
嗯他高16位啊
它前是一或者是什么呢
或者他前世的这个地址
那就称为什么称为规范的
这个什么称为规范地址
那么两者之间注意啊
两者之间是不规范的
这个地址不规范地址
那么就是告诉大家就是不允许什么呢
就是不允许使用了
他这个意思啊
这是非常简单啊
那么还有一个点是什么呢
还有一点就是说
你比如如果我们在arm这个什么呢
它大的虚拟地址
并且支持什么64KB
那么这个虚拟地最大最大宽度是52位
是不是后面大家可以想
那么可以什么可以为虚拟地址啊
配置比什么呢
最大的一个什么宽度
小的一个什么啊
带这个带宽
这是都可以做得到的
那还有什么呢
还有就是说我们要知道的一个问题是什么
就相信这个问题啊
来我们看下面啊
好这里是什么呢
这个地方到这里来之后就好办了
是不是这里来是什么呢
就是相当于就是说我们在编译
是不是别人在编译啊
你在编译什么
我们在编这个arm64啊
这个架构这个架构在这个什么呢
LINUX内核的时候
那我们可以什么呢
我们可以选择这个虚拟地址的一个宽度
那兄弟的宽度
你比如第一个啊
比如A是吧
你比如A
A的话呢
你看如果如果我们写这个叶的长度是4KB是吧
那么它默认默认在一个什么虚拟地址
它的宽度模拟虚拟地址宽度是多少呢
是39位
就这个意思是不是后面就意思类推了是吧
A是吧
下面是B
那B的话呢如果选择这个液态长度啊
如果不是4K的
他一是多少K16K
那么默认这个虚拟地址的宽度就达到什么
达到了47了
OK是吧
那如果我们选中的是下面这种成就是哪一个呢
如果选择E的长度是多少K呢
嗯如果你选择的是64K
那么默认的一个虚拟地址宽度啊
对方达到什么
到了42
因为这个比较大是吧
还有啊
如果我们选择什么呢
如果你选择这个48位的
你也可以注意啊
我们也可以选择
嗯可以选择可以选择什么呢
可以选择这个48位的一个什么虚拟地址
但是我们必须要明白一点
什么意思呢
就是相当于在那个什么arm64架构啊
在arm60这个架构的LINUX内核当中
内核当中对不对
那么内核虚拟地址和什么呢
和这个应付的一个什么虚拟地址
它的什么
它这个宽度是相同的
但是什么意思呢
就是说所有进程注意啊
所有进程它共享什么呢
共享内核的一个虚拟地址空间
那每一个进程每个进程它有一个什么
有个独立的用户虚拟地址空间是吧
那么同一个线程
同一个线程组的用户线程它共享什么
共享用户啊
虚拟地址空间看到没有
那内核线程呢它没有什么呢
没有用户虚拟地址空间还有这个意思
所以整个思路我们现在就全部解搞清楚了
是不是啊
那搞清楚更好
那么接下来就来看下一个是不是
这是第一大点啊
虚拟地址空间的一个布局
好那现在我们来看下一个下一个是什么
就是第二个了
是不是第二个什么呢
就是用户虚拟地址空间布局是怎么回事
那就需要另外一个方式进行思考对吧
下面就是第二个
用户虚拟地址空间的一个什么布局
OK那么他这个布局
你比这个进程进程的一个什么用户地址空间
它的一个什么起始起始地址是为减为零
但是长度它是什么呢
长度是task下划线size是不是由于什么呢
每一种的这个处理器架构对不对
他定义自己的什么呢
自己的这个红用自己的宏啊
task下划线有一个什么size
就这个意思是吧
比如我们刚刚讲了这2M64架构定的
红砖架构定的行
怎么做呢
他这个宏是吧
task size是吧
他怎么做
看下面这个方式好
那它这个方式是这样的
就是说比如第一个是吧
你比如他是32位
那如果是32位用户空间程序
你32个应用空间程序
那这个值的task size是不是它的值是多少
是下划线有什么
有一个三二看到没有
我的32G就相当于0X后面是1123
就这么多题要注意啊
它就相当等于几呢
是100
121234诶
这地方在哪里看啊
这个词我们可以找到啊
在内核源码里面找就不记得了
等一下啊
不要着急
打开内核源码好
我们在内核源码里面有一个arm64
嗯下面有一个include sm下面的memory
嗯双击
看到没有if看到没有井号if ef这个红啊
这个条件编译啊
我们看看这个条件编译就值了
好那么这个条件编译的话啊
要注意了
这个条件编译
如果task size si
那么它就直接是什么呀
1001几个零
把它复制出来
好放大了是吧
看大小就出来了
是不是
那就解决这个问题
是不是一下子就搞定了啊
好那我搞定这个问题
这是基于什么基
相当于这个地方就32位嘛
32位就相当于什么
就等于多少
大概就是这么多
就是有4GB啊
那如果如果你是64位
那如果你是64位的那个什么呢
用户空间程序
那怎么办呢
那么task size的值啊
这个地方他就不这么做了
他就直接改变为这个多少呀
后面就是六十四六十四嗯
它就相当于相当于银行啊
这么算的啊
如果是64位好看
64位
64位
就相当于我们说这个VBA的32次方嗯
这个子64past size
图上设计操作是吧
那这样来看没有VA下划线
就是二的左移嘛
对不对
他用他左移左移
左移的话就这么上来
一的左移
那就相当于二的
嗯VBS44方这个字节就这么算了
那叫什么呢
这个VBS它是什么
是编内核的时候
所选择的一个什么虚拟地址位数
他做这个事情
这个我们全部搞清了是吧
好搞清之后呢
那么这个内核的页码啊
现在翻出来看一下
我把它放到这里
大家可以看一下
在这里
把它移过来啊
干掉
就可以了
就是说现在刚刚你所看到这个布局
是不是他全部就在这里
这个图他这个大小这个图啊
它是这样的
看到没有
你看这个图
你就知道是不是
32位和64位的用户空间程序
对不对
他怎么进行计算
他这个图就完全可以明白
但是你还知道一个点
是不是就像什么呢
你比如这个进程进程的进程的这个什么呢
用户啊
进程用户虚拟地址空间它就包含什么呀
包含如下这个区域
那么它的哪些区域呢
些什么代码段啊
是不是还有一些什么数据段啊
很多啊
我们看举个例子
你比这个代码段
代码大师是吧
和代码还有什么
还有数据和什么和没有初始化的那个什么数据
还有一个什么动态库的这个代码端
动态库代码段还有什么呢
还有这个数据单和什么
和没有初始化的这个什么数据端是吧
动态库还有什么
还有存放动态啊
存放动态生成的一个什么数据的堆
由此以外你比如乘换什么呢
存换局部变量和什么呢
和实现函数调用的一个账
还有什么
还有存放在这样的一个什么
在底部的一个环境变量和什么呢
和一些参数字符串
除此以外还有什么呢
还有就是我们可以把什么
你可以把文件区间
把文件器件呢它映射到什么呢
映射到虚拟地址空间的内存映射区域
整个思路啊我们全部搞解决了
这是其中之一啊
搞定了是好
那这些问题啊
哎我们全部搞定之后
那下一个是不是
那下一个是什么
下一个我们针对这些有内存呢
他怎么去描述一个什么呢
进程用户信息地址空间
它怎么就模式它是这样的
要注意啊
就是相当于什么
相当于我们在什么
比如我们在这个嗯就是内核了是吧
内核啊
它直接使用什么
使用一个内存描述符
对不对
是类似描述
专门有个什么mm下划线有一个structure这个描述啊
这个描述符啊
mm下划线来描述什么
描述进程的用户虚拟地址空间
他就这个意思
那么这个内存描述符它的一个什么呢
主要成员就是主要成员
主要成员我给大家列一下
是不是主要成员就是内核的这个什么掩码如下
内核源码看一下怎么做
看我们找几个核心产业
大家注意啊
搞几个核心点成员
我们来看一下啊
这样大家看起来方便又简简单
是不是这个也不难啊
没问题了是吧
这些生成的文档大家可以看
这个就是一个笔记
非常标准的笔记
好我们再往后面
那往后面他怎么生成呢
导到这来内核源码啊
在哪里呢
在这个include下面有一个LINUX
LINUX下面有个mm下划线TP
mm这个地方双击
双击之后
你找到一个刚才我们讲了康JF有个mm下划线
一个STRUCT
看到没有就找到答
看到没有
点它就可以了
看到没有
这就出来了
那么这个就是专门用来描述什么内存
多内存描述符
其实它就是什么
就是描述这个进程
他这个用户啊
虚拟地址空间
那就这么一个思路
现在啊我给大家解释一下
算OK什么意思呢
就是相当于啊就是内核
它使用什么来使用这个内存描述符
看到没有
mm下划线这个structure啊
专门描述呢描述进程的用户虚拟地址空间
看到没有
那么主要成员就在下面了
我就不再列了
你看首先我们来看第一个啊
你比如第一个这个嗯嗯比如那个什么呢
比如这个啊
你看有一个下面一个VM小于A的
它也是套用一个结构体来定义这个mm
其实这里是什么
这里所告诉的就是虚拟内存区域的一个链表
就他这个成员啊
他这个成员就是专门就是虚拟内存区域的一个
什么链表区
内存区域列表还有一个什么
你比如再搞一个专门有一个mm下划线
还有个什么阿B没有超出几个核心成员
他用什么RB下划线rot
其实它是个红黑树是吧
那么就说它是什么呢
是虚拟内存区域的一个什么呢
红黑树进行了存储
怎么解的呢
下面是v v m a catch SQL
是不是就是每线程啊
他那个方程就在这里嗯
除此以外
那后面还有很多啊
我们这还有两个核心点
看看有这个再找一个啊
嗯这个点看到没有这个地方
那么这个PGD下划线T是吧
你像这个成员这个成员是专门要讲的
为什么呢
因为这个指针它是指向什么
指向一个叶啊
全集的一个目录
就是我们所讲的D级列表
这个比较难啊
野马分析了
再往下面走
虽嗯完了之后呢
它有一个什么
你看上面有个task下划线size
那这个task下划线size是用来干什么的呢
这个成员他是这样的
它主要是用来实现什么呢
就是我们所讲的用户啊
用户虚拟地址空间的一个什么程度
后面你比什么比这个还有一个map map bass
是不是a map bass
这里面它主要是什么
主要是内存映射区域的一个什么提示
就这么简单
它整体思路就这样了
所以我们要了解这些啊
还有一个切换上下文呢
还有还有一个成语
还有一个成员是非常重要的
就是mm下划线user对这个地方嗯
M下线user是什么意思呢
就是共享共享同一个用户嗯
这个什么虚拟地址空间的
一个虚拟地址空间的进程的一个什么数量
也就是什么
也就是线程组它所包含的一个什么进程来看
这个事情是再往后面还有一个什么M下划线
CTRL有两个M啊
那么这个control是什么意思嗯
多少是有录屏吗
有录屏啊
今天晚上
前程我全部都给大家把这个视频录下来
是不是
那么你直接找这个一老师领取就可以了啊
知道吧
你直接找一老师
这个老师看到没有找他领取啊
看到没有加上它再领取
今天晚上那个笔记啊
包括这个视频
是不是就是今晚的这个什么课堂笔记啊
课堂笔记和什么呢
这个嗯
录播视频找一老师啊
免费领取
OK是这里我就不再多列举了啊
那现在我们就开始看下一个点是吧
那么这个地方啊我们刚才讲了
是不是就到这了
每个点都很重要
大家注意点太重要了
是不是因为一个进程产生之后
还有内存了一串的关联啊
mm下划线CTRL是不是他表示什么
表示这个内存描述符
内存描述符的一个什么引用技术就这么简单
除此以外后面有一个什么
有些成员啊就是用来架构定内存管理的啦
就是盐后面会讲一些叫什么审计
用到的叫做找回你啊
别着急
这个mm下划线看到没有
count下划线T买这个地方
这里面它是什么
它是处理器架构
特定的一个什么特定的内存管理
上下文就这个意思了是吧
全部清掉好
那么这个解决完之后
那基本上等于取出来了
那这里面还有像这些这些也是常用啊
你看start code是吧
end code没有start date和end date这个词分别代表什么
意思是非常简单
你看啊
那前面两个
study code和decode
那这个是表示什么呢
就表示这个代码段好
代码段的一个什么
代码段的起始地址和这个什么和这个结束地址
后面那个date就是什么
就是数据大的一个起始地和结束地址
思路就这样啊
这两个人一起注册了是吧
那么还有一个什么
还有一个往后面啊
下面再来
Ok
你看start这个什么bk
是不是还是type sb k是什么意思呢
你看啊前面的两个是不是堆满
前面那两个是堆
就是堆的一个什么
堆的一个起始地址和结束地址是
后面有一个start tag
那个是什么藏的一个起始地址
就这个意思啊
这个就出来了啊
嗯再往后面那么有什么
你看ARGARGN没有EV是吧
这两个现在来看一下
前面那个前面那个是什么参数
要注意啊
那这个就是参数参数
什么参数字符串的一个起始地址和结束地址
看到没有
后面那个是什么
你看啊
EV是vim
它就是环境
就是环境变量的一个什么起始地址和结束地址
你看整体啊
这个mm下划线这个结构的这个整体掩码
核心部分我钱都给他注释了
是不是
但是你要搞清楚啊
这几个点了啊
还有哪两个点呢
这个进程描述符当中和内存描述符的核心
它之间是有区别的
那现在我把这个掩码放这了啊
好我们来看一下
KCTRLC是吧
好直接放在这里是吧
出来了好
那么放到这来之后呢
我们再往后面看啊
到这来是吧
下面是什么
下面你要知道一点
就是这个进程的
就是我们经常变成这个进程描述符
也专门有一个后面会讲是吧
后面会讲叫做task下划线这个structure嗯
这个进程描述符当中和内存的一个什么
就是这个内存描述他的一些什么
往相关联的一个什么啊
如下看一下哪些成员呢
你看啊
比如说进程描述符
是不是他这个成员怎么去关联
比如进程描述符
它这个成员是在哪里呢
后面我们有个备注数据
有个mm你看看
找一下
叫做
STRUCT啊
哎
你输入一下输一下
mm下划线
tract空格信号
mm这个
嗯这个成员看到没有点慢啊
那么你像这个进程描述符这个成员
那么它是用来做什么的呢
要注意啊
它是什么
就是进程进程的这个mm它指向一个什么
指向一个内存描述符
要注意啊
那么内核这个线程他没有什么没有应付啊
虚拟地址空间
所以他这个mm它是一个什么
是一个空指针
这是我们要知道的是吧
那另外一个呢还有个成语叫active是吧
嗯一个active
装了一个程序看啊
task下划线标指在
要连续看
Ctrl f
这叫做嗯STRUCTORMM下划线啊
就这个了
看到没有
就这个成员看到没有
就在这里了
就在这个进程这个描述符
看到没有唉
进程要输出量
进程描述符当中有两个成员
一个成员是mm
刚才已经讲了
另外一个是下面这个
看到没有
下面那个没搞清楚啊
下面一个是吧
后面那个成员注意啊
嗯可以的
这个干什么
你像这个这个成员啊
这个成年这个进程的这个啊
比如嗯active mm和什么呢
和后面这个mm它总是什么
总是指向同一个什么呢
同一个内存描述符
为什么这么讲
看这里啊
你看这个
不管你是不管你是mm还是TM
这两个指针都是指向这个内存描述符的
这个结构体的
他就这个意思
然后接着什么接着看到这边来啊
接着就到这来了
就是说明什么
相当于你比这个内核线程
内核线程的这个嗯active mm是吧
他在怎么在没有运行的时候
它是一个什么这个空指针嗯
在运行啊
在运行时呢
它就指向什么
指向从上一个什么上一个进程借用的内存嗯
内存描述他刚刚这个思路来
所以我们不能搞错是吧
看到了你用这个进行描述就可以了是吧
但是我们要知道
那么如果进程不属于线程组
那么这个进程描述和内存描述的关系对不对
你比如说他两个指针
他怎么去处理关系
怎么做
现在我们往下看啊
你只要看我这个笔记
你学习这个内核源码分析是非常简单
下面来分析一下
那么如果什么呢
如果进程它是不属于什么
不属于这个线程组
那么这个时候呢它进程描述符啊
嗯进程描述符和内存的这个描述符
它们的关系像来看哪些关系呢
你看啊是这样的
画一个图
他怎么就表示出来
嗯这个进程和内存之间的关系
里面啊
等一下
来往这边这个是进程描述符
就是我们所讲的task下划线那个
可以啊
里面有几个成员
那先找到task struct看啊
这我们以这个为例啊
刚刚我们讲了这个进程条舒服
那么party starter进程描述符在内核里面的描述源码
看到没有
比如他第一个成员
第一个成立
好接着第二个成立
还有后面
跳蚤是吧
可以了啊
对面很多啊
好那么这两个指针看到没有
指向内存里面去啊
这两个里面啊
嗯这里没有用没有用到的啊
改一下颜色没用的
用红色表示
然后他指向什么呢
它指向这个内存
要说服了他
刚才是进程是吧
现在它指向内存描述符
内存描述符有对应关系啦
它不是task啦
它是mm下划线
看它就这么表示的啊
但是这里面还有两个两个成员
那就回到这来看memory里面啊
这个struct mm下划线
看到这来看没有哎
看到了在那个auto user user
啊这个成员还有一个mm control
那么他怎么进行操作
相当于什么呢
相当于他这两个指针对不对
它分别指向他的
指向这个结构体啊
你要注意啊
我们把这个移下来一点吧
这个不好放
就这么画吧
线出来了
是不是
好差不多
好搞定了
那么在搞定之后它有个什么呢
就是有指向的时候
它这个值啊
user这个值他一定是什么思维
看到没有
后面这个CTRL它的值也得到一
后面还有的成员就不列了啊
那么这个表示它是什么呢
就是那个进程描述符啊
进程的进程描述符和内存描述的关系
就这个意思就是说进程不属于什么呢
进程如果进程不属不属于线程组
那么他们之间关系的就是不同一个线程组
你俩之间的关系就这么个意思
我直接把它放在这里啊
这就是这个关系
那么怎么一回事呢
接着你这个思路又来了是吧
他怎么来呢
是这样的
比如说进程进程这个描述符成员
比如mm和什么呢
和这个active下划线
mm他都什么都指向同一个内存描述符
看到没有
都指向同一个内存描述符
那么内存描述符合它的一个什么成员
内存描述成员mm下划线
这个users是users
他就得到什么
就为了看到没有
那这个成员呢mm下划线啊
control是不是它为几
唯一就是其中就是嗯这个这个关系啊
就是你进程不属于同一个线程组的这种关系
我们就解决了
那么还有一种就是说如果什么呢
你如果两个进程是两个
现在只有一个进程啊
对不对
那现在我们如果是两个进程怎么办
两个进程
那么如果两个进程两个进程呢
嗯它属于什么呢
属于同一个线程组
同一个线程组
那么怎么呢
那么这个进程描述符是吧
和什么呢
和内存
这个内存描述符的关系
他就在下面是怎么做的
你看我们用这个图来画一下就出来了
同样的关系我们来看一下啊
他怎么就相当于两个都指向一个地方
就这个意思就是两个进程都指向同一个地方
好看
这是这是什么
这是那个进程一啊
就是进程
P1了
来进程1CN加C
嗯这个是进程二
他怎么去指向
道理是一样的
他也指向这一块啊
你要搞清楚
这里也是一样
啊差不多把这个颜色改一下啊
你比如这个
线
哎
现在我们用
红色是吧
这样的话你看起来就那个啊看的清楚一些是吧
下面这个也是一样
放一根线啊
嗯全部搞定了是吧
OK就是指向整个啊
这个内存描述他的思路就是这样的
所以说在这里它又引发出另外一个问题了是吧
进程一和进程二了
你看把它复制过来啊
那么所以这时候父子就不一样了
你看啊
这时候来一个mm下划线CTRL是吧
哎这个值仍然为一
但是它的优势就不是一了
你两个进程
两个进程就变成二了
三个啊
这个读数变为三
它就这么个关系
你看这个思路他就这样了
咱们把它换成看看
没有关系
我们要理解
要理解好
那我们理解这个关系之后
那后面就往后面走了啊
还没什么意思呢啊
就相当于每一个进程的这个什么呀
进程描述符这个重点啊
MMMM和什么呢
和这个嗯mm和这个active是下划线
mm那都什么都指向同一个内存描述符
同一个内存这个描述是吧
那这个内存描述符呢
内存描述符这个成员是吧
mm下划线users就为二了
是不是
那么他这个成员呢这个成员mm下划线是吧
这个control它就为几呢
它就为一
就相当于等于一吗
道理就出来了好那么当然注意了
由于什么呢
看啊
内核线程的进程描述和内存之间的关系是套路
听起来注意啊
就是你内核线程的进程描述符
跟内存描述符的关系
他怎么指看着啊
现在来看这个是内核线程
内核线程的进程描述符和内存描述符
它的一个什么关系啊
下来看一下
那他怎么做呢
我画个图啊
比如说进程描述是吧
直接列就很简单了
你看
有第一种情况
就是相当于在设计的时候假设啊
这个这个是什么呢
就是内核线程没有运行运行
如果运行会借助内内内内存描述的这个意思
那么内存线程内核线程没有运行的时候
对不对
没有运行的时候
这两个成员这个成年它就等于空
没有下面这个它同样的道理
它也是等于什么
等于空来就来干这个事情
来拉过来一点
可以了是吧
嗯解决了算注意啊
现在所画的一个图形
它是一个什么
是一个进程描述符对吧
就是说它内核线程没有运行的时候
它是这个样子都为空好
那么如果它运行了内核线程运行怎么办
那内核线程运行了
那你这个时候这个地方就变为空了
激活了是不是激活了你这个成员就不能为空嗯
然后他就会借助什么
借助内存描述符的地方
他会借助内存条束缚
看到没有这个指针
它就会什么指向内存条束缚
就这个意思
为什么呢
这个地方他这样的是借用借用内存
嗯描述算这个是进程描述
就是一运行跟没有运行是有区别的
这边它是什么呢
内核线程运行的时候
唉他来干的事情
这个是没有硬性的
要注意啊
没有E显示这个这个情况
没有硬性
看到没有
这是没有硬性好
另外一个呢它是硬性的
硬性的话看这user它只有一个啊
一个进程
对不对
这里面CTRL是两个
就可以了
所以现在我们所看到的这两个图形是吧
就很容易理解是吧
这什么意思呢
内核线程没有用户虚拟地址空间
不就是当什么呢
这样的就是内核线程它没有什么内核线程
没有用户啊
虚拟地址空间
当内核线程呢嗯它没有运行啊
没有运行的时候
对不对
那这个进程描述符的成员就是mm和哪个嘞
和这个active active下划线mm它都是什么
都是一个空指针
看到没有
这是一种情况
那么他一屏的时候又不一样
一行是下面这种情况
看到没有
就是你这个内核线程是吧
都是空R
那么当这个内核线程它运行的时候
运行的时候呢
这个时候他会借用什么
借用一个进程进程的一个什么内存描述符
进程的内存描述就是你在背什么
在被什么再被这个借用进程的用户啊
虚拟地址空间
近地空间的一个什么呢
上方运行要搞清楚是吧
但是这个进程描述符嗯
进程描述符的这个成员是active下划线mm是吧
它指向什么呢
指向这个借用的内存描述符
那假设假设什么呢
假设你被借用啊
假设你被借用的这个内存描述符它会怎么样呢
这个内存描述符它所属什么
所属的进程不属于这个什么线程组
那么这个内核描述符
他这个纯圆mm下划线user它不变
它仍然是什么不变
它仍然为什么仍然为一
就这个意思
那纯岩这个mm下划线CTRL是这个统计
它就为几啊
就是加一之后它就围起来为二
他这个思路啊
整体操作他就这样做了
OK你看这个不难是吧
我们就搞清楚了
好搞清楚之后呢
再往后面啊
我们注意什么意思呢
比如我们为了什么
为了使这个缓冲区溢出攻击更加困难
那会怎么做呢
就说内核支持为内存映射区
包括站和D选择随机的一个起始地址
进程是什么呢
进程是否使用这个虚拟地址空间的一个什么呢
随机化功能
那么由于什么呢啊由以下两个因素决定啊
一个是进程描述库成员
还有一个是全局变量
是不是啊
这两个进行处理啊
比如这个账啊
通常是什么呢
自顶向下增长
这个我们后面会讲啊
大家不要着急是吧
除此以外
你还必须要知道什么
你比这个内存描述是吧
你看这个内存描述符啊
那这个内存描述符就是内存映射
你看他怎么做
这是第一个吧
第一
三个是第三个
第三个第三个现在我们看一下第三你比如什么
比如这个内存内存映射区域
它的一个什么起始地址
是内存描述符的这个成员啊
是m map下划线base
刚才看到了
对不对
那么用什么用用这个图啊
使用这个图形
看到没有
使用图形表示啊
你比如这个用户虚拟地址空间和什么呢
用虚拟地址空间唉
它有什么呢
有两种布局
对不对
有两种布局
哪两种布局呢
你比如它区别在于什么
区别是内存映射嗯
内存映射器的一个起始地址就是起始位置了
起始位置的一个什么
其实位置和这个增长的方向不同嗯
那基本思路就是这样的
你比如说传统方向对吧
比如传统传统这个布局啊
你除传统布局以外还有什么呢
还有一种是新的布局
但传统布局新的布局怎么回事
我们来看一下这两个是有区别的
好我们来研究一下啊
一种是传统武器和新的补给
这个画一个图形
通过一个图形
我们就更加容易理解
嗯在线画一个图形
你看这边这个不够了
好单位花钱算下来一点了
下来一点的话
我们看这里啊
首先你看这个传统布局内存映射啊
自底向上进行增长
画一个多一点啊
就这个意思有第一块
然后呢
第二块
第三块
可以了啊
大小也不能太大
他等下预算不够了是吧
复制一个过来
你比如说这个是赞啊
这个是赞
字比较小
改大一点
20号
这个是三
这么下面呢它是一个什么
是一个内存的映射区
是不是内存
OK如果各位同志们啊
大家后面来没有跟上
是不是
如果在学习过程中有任何问题
大家可以什么可以加秋香老师啊
来领取今天晚上的课程资料
就是207032995已经发了
公平了是吧
接着我们再来下一个了
下一个下一个这里啊来一个这个是什么呢
这个就是我们写一个
对啊
这不是堆堆
注意下面就是你没有什么
没有初始化的一个什么数据啊
没有初始化数据段啊
还有一个什么还有一个就是代码当
数据大
数据大才是代码段
代码端
可以了
再往上再往下面再来一个
可以把它标一下
标一下中间那个是空是没有的
标一下
这样就容易理解
我涂一下颜色啊
用这个来表示
内存映射区
我用红色表示
下面这堆未初始化数据
还有个代码单
代码段
飞sir表示
可以啊
那这个全部解决之后
有一个点什么
比这个站里面是不是还有一个空虚
看下C
他怎么增长呢
他是这么长
你就站从内存区里面直线
这个弯起来了
可以了
这么增长啊
这么增长是吧
这里他这样增长
之间的关系这样真的啊
那个堆
嗯等一下啊
多的增长
可以从这里发出来
啊这个就可以了
看到没有看它基本思路就这样了啊
这个时候对一个灰色来表示嗯
全部都解决了这一种方式啊
注意这种方式它是一个什么呢
是一个传统的传统
他表示什么呢
就是传统布局
传统布局就是什么
就是这个内存映射
内存映射区域它是什么呢
是知底知底向上进行增长
他思路
这个要搞清楚
另外一个呢要注意啊
他怎么表示你比对方他从零开始了是吧嗯嗯
好引到这边来是吧
到这个什么内存映射区里面
内存映射区域这个词它是怎么指向的啊
红色这个地方
相当于什么呢
比这个mm它所指向的一个什么呢
他说MF没有一个
它等于什么
等于这个task
刚刚讲了
task下面像什么呢
UNMAP下划线
一个base直接加起来
直接加一个随机值
嗯他就会得到这个词
OK就是这一块啊啊除此一块之后
那么还有一个什么
还有个上面比这个赞
是不是你小子
这个蛋啊
那这个站的话呢它是什么呢
task size最大值从这开始嗯
task下划线有一个什么size减一是吧
到这个站里面来了
在这里的话就是stop stack tag
下划线那个什么top直接减掉什么
减掉这个随机
就这么一回事了啊
整体思路啊
他就这样走
看到没有
就搞定了
现在你所看到这个它是一个什么
是一个传统的布局
那么如果是新式布局就不一样
传统布局它是什么呢
是支底啊
自底向上进行增长
这是传统
那么复制过来啊
我直接复制过来啊
复制过来之后粘贴到右边来好
这个就是另外一种方式了
就是新式布局
就相当于什么相当于新布局
新布局
那么这个哎
啊这个心不仅是这样
就说你比如他这个不是这样长的
它占比
这个就不要了
他从这边往下增长
看到没有啊
这个往下挪一点
嗯OK那这个的话这个线啊
再来一遍
可以了
那么现在大家所看到的堆是不是
那么它这是一种什么
不是传统啊
这是一种新的布局
那么这种新的布局
从这个结构图我们可以看出来
是不是什么结构啊
这个内存映射区域它是指什么来自顶看到没
置顶向下进行什么自顶向下进行增长嗯
那左边这个是什么
左边这个它是知底看到底向上增长
对不对
自底向上增长啊
这个是自底向下
要搞清楚自底向上哎
那所以现在我们所看到两个图形
就很容易分清楚了
很容易就把它分清楚出来了啊
看来把它弄到这来啊
嗯解决
好搞定了啊
现在看这个笔记就出来了一个窗口是吧
一个是什么呢
一个是新布局的啊
好传统
看能不能一块截过来
可以啊
解决了
传统和新布局
全部解决
那么这个传统和新布局
那么现在出来之后
我们就很容易的把它分析出来了
那为什么呢
你看啊传统看清楚这个传统传统布局
你比如内存映射
内存映射区域
它是自底是不是自底向上进行增长
那么其实它其实是什么呀
其实这个呢是task下划线
UNMP的下划线BS
但是每一种处理器架构都要定这个红
那么arm64这个架构呢只用task除以四就行了
那默认它启用内存映射区域
它是随机变化的
就是随机化
需要把一个什么
需要把一些起始地址啊加上一个随机值才可以
是不是
那么就是说传统布局的缺点存在哪个地方
是不是这个注意看了没
传统
传统布局
传统布局的一个缺点
它是什么
这个堆堆在一个什么最大长度啊
他受到了什么
受到了一个限制
就这个意思就相当于什么呢
相当于你在32位系统当中
它的影响是比较大的
但是你在什么呢
你在64位
那你在64位的这个系统当中啊
就完全没有问题了
很大了
是不是就不用操心这个问题
OK那就这个啊
那么还有一种是新布局了
新布局内存预设
它是资本项下嘛
那么其实是他是泰克啊
对对对对
一样啊
起始是tag开始发发反了哈啊它的一个起始嗯
他加上什么这个地方他不怎么怎么取
就是这个pop
它是一个stack
下划线那个top干什么呢
减伤这个站的一个最大长度
看到没有
然后再减上这个间隙
继续啊
来继续再减上这个随机值
好他那个处理方式就是这样的啊
这个我们解决了
嗯搞定点这个图画
就把它放到这来算了
分开啊
分开搞过来
嗯这个是
传统布局缺点
下面是新布局
好那这个新布局的话再带一点啊
可以啊
那这个新布局我们先搞定了
是不是
那新布局的话要注意啊
他这个什么呢
比如当进程调用完装载EF文件的时候
它会导入
是不是将会创建一个进程的用户虚拟地址等等
是不是还按这个思路来
但是它真正是什么呢
你给它装载EF文件的时候
他怎么去按照这个步骤啊
啊一步一步的把它加进来
好
我们来看一下
比如什么呢
你比如他各种处理器
是不是你比如各种注意各种处理器
它是怎么去执行这个架构啊
我要分析一下
可能还比较难是吧
好我们来看一下
就是各种处理器架构
他这个自定义啊
自定的函数呢
它的内核里面要注意啊
在什么呢
在这个
大家看一下啊
写在内核源码里面
找到这个你看arm看到没有
ARCHM64里面有个内存吗
mm mm下面还有个mm
看到没有
IMAP点C了
map点C里面装一个函数啊
CTRLF有一个什么ARCH里面的那个什么pk
就这啊就这个函数是吧
其实现在我们所看到的这个函数来拷过来
这点东西买这个行
他干什么呢
它主要是负责啊
负责选择这个内存映射区域的一个什么布局
那这个布局这个联盟我们来看一下啊
那首先我们来看第一个问题是他从什么呢
从这开始看啊
整个结构是不是
那整个结构的话呢
你所看到的啊
你比如他从这个地方开始是吧
嗯从这个地方开始是吧
异地分开始到后面再来
那么它主要是做什么呢
啊前面是初始化嘛
它主要是干了个事情
就是相当于什么这个地方
它相当于我们要执行
就是如果判断处理掉这个函数
就是如果什么呢
如果我们给这个进程
如果给进程描述符这个重点
是pre personal personality来设置一个什么呀
他设置一个标志位
这个标志位呢
就是我们这个ADDR门槛标志为
当DDR下划线一个嗯
看到没有
compact下划线
Line out
他表示什么
表示我们使用的是传统的那个什么虚拟地址
空间布局
那么然后呢或者是说你这个用户账对不对
他可以什么可以无限的增长
嗯或者你可以通过什么通过文件专门一个PLC
我们讲过SYSVM虚拟
用这个来表示是VA这个什么lg out是吧
进行了指定
那么一般情况下对不对
他来执行一些相关的函数呢
装一个RCH这个什么来进行处理
比如如何进行
如何进行什么来进行这个字啊
比如自顶向下布局
是不是调用哪些函数
在内核里面专门有些同意跟他都有关系
OK这个一点码我就直接放到这
所以你根据这个来你就可以搞个第二是不是好
那么这些我们解决之后
后面我们还注意一个问题
就专门针对什么是本节课的三大点
第三个点是什么
就是内核地址的一个什么空间部
内核地址空间布局
你给我以arm64为主
arm64啊
他这个处理器啊架构的一个什么呢
架构的一个内核地址空间布局
他怎么做
如下图所示啊
这个图我已经给大家画好了
比较大啊
这个图在这里看到没有
就这个结构可能太大了
换小一点
把它关小一点就小一点是吧
好就到这了够了
双击
那现在我们说大家啊
就是现在我们大家所看到这个图形就很清楚了
那么这个这个这个图形告诉我们什么呢
告诉我们这个整个架构的地址分配
那你比如啊比如比如这个线性映射区
是不是你看线性映射区域这个范围呢
它是从什么呢
叫pg下划线offset
对不对
到什么到二的
你看他左一嘛就是二的64次方减一
就是64啊
二的64次方减一
那么其实那起始位置就page offset
对不对
那么它的一个什么呢
就等于什么
等于这个0X后面16个F没有16个F啊
直接走就行了
那么长度是内核虚拟地址空间的一半嗯
线性映射区域长度是内核虚拟空间地址的一半
就称为线性映射器的原因是因为什么
原因是因为这个虚拟地址啊
和物理地址它是一个什么关系啊
它是一个线性关系
大家要搞清楚啊
哎就这么一回事了
是不是好
那我们一旦搞清这个啊
搞清楚这个关系之后呢
那后面还有一个什么
还有一个点是针对什么针对的
第二个就是你这个虚拟地址了
那你这个虚拟地址它是等于物理地址啊
对不对
加上一些什么patch of s
那么其中我们刚讲了个什么
比这个page set是不是
那这个是什么意思啊
这个就是内存的一个起始物理地址
还有你比这个v m e m map是不是m m e m map呢
是不是啊
那这个是我后面要表示一个什么
它这个范围它长度是整个一个什么线性期长度
除上这个页的长度就可以了
包括配置这个结构体的长度上下限
还有往下面你还有什么
还有两造间隙
我就不管了
还有个PCI杠啊
PCIIO区占16兆
那这个是什么意思呢
它那个接触力就是相当于外围组件的一些互联
你比如PCIO的一个设备
就PC设备的IO地址空间
那就从这一段进行取的好
那么从这一段取之后啊
后面怎么做呢
后面你比如后面还有一个固定的映射区是吧
那这个固定的映射区域它是什么呢
就是相当于什么呢
就是固定的地址是编译的时候啊
它是一个特殊的虚拟地址
编译的时候它是一个常量啊
在内核初始化的时候
它就映射到物理地址
他就这个意思
后面你比如还有一些什么
还有内核模块的内核镜像呢
是不是内核镜像用VMC嘛
这个我就不多讲了是吧
还有一个内核模块
内核模块它的长度是128兆
内核模块
它是内核模块使用的一个虚拟地址空间
还有比后面那个看影子区是吧
那这个影子七是什么意思啊
哈哈注意啊
这个影子区域的一个起始地址
是内核虚拟地址空间的一个什么起始地址了
就不写出来了啊
长度是内核空间的地址1/8
那么内核地址它是这个消毒剂
就是COCL
它是一个什么呢
它是一个动态内存检测错误工具
他为什么呢
发现释放之后啊
使用啊
或者是一些业界访问啊
等等这些切线问题来提供一个什么快速定位
综合的一个解决方案
那么通过这个图形
就可以可以导出这些相关关系的
是不是
那我们搞清这个之后
你看我们后面还有第四个了
对不对啊
第四个叫什么内存序啊
内存映射啊
物理地址内存后面我就给大家写出来
把笔记换掉了
那么其实我们在学习过程当中
我们还有什么注意啊
像今天晚上我们所讲的内容是吧
是专门针对啊
各位同志们啊
就是说具备一定的开发能力
想从事内核开发的相关的这个什么岗位的啊
或者是说以后你学了个什么啊
库巴斯啊
前提要具备的内核功底是吧
做内核开发的
做应用程序开发的
应用程序开发的一个工程师啊
以及什么呢
你做这个啊
网络安全驱动这一块



## 参考

[Linux内核内存管理 1_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1xi421o73f?p=1&vd_source=4a018e55005b433dc9d72a8e969a1c5f)

字幕提取

[B站字幕转换 | 牧尘的NAS小站 (dreamlyn.cn)](https://www.dreamlyn.cn/html/bsrt.html)

F12-ctrl+r-网络-过滤（ai_subtitle，json）