# Linux任督二脉之内存管理(一) PPT

内存的`zone`: `DMA`、`Normal`和`HIGHMEM`  
`Linux`内存管理`Buddy`算法  
连续内存分配器(`CMA`)  
看`/proc/buddyinfo`

为什么分`ZONE`  
![图片](image/640.webp)

![图片](image/640-17288389658491.webp)

![图片](image/640-17288389658492.webp)

这里谈分页机制

虚实转换

RWX权限

特权模式权限与非特权模式

![图片](image/640-17288389658493.jpeg)

这是一个背离RWX权限导致段错误的例子

  

![图片](image/640-17288389658504.jpeg)

背离特权模式保护的meltdown漏洞

  

![图片](image/640-17288389658505.jpeg)

  

为什么分ZONE

  

![图片](image/640-17288389658506.webp)

  

DMA

  

![图片](image/640-17288389658507.jpeg)

  

DMA zone应该多大

  

![图片](image/640-17288389658508.webp)

  

Buddy算法

  

![图片](image/640-17288389658509.jpeg)

![图片](image/640-172883896585010.webp)

![图片](image/640-172883896585011.webp)

  

CMA

  

![图片](image/640-172883896585012.webp)

![图片](image/640-172883896585013.webp)

![图片](image/640-172883896585014.jpeg)

![图片](image/640-172883896585015.webp)

![图片](image/640-172883896585016.webp)



## 参考

[Linux任督二脉之内存管理(一) PPT_linux任督二脉之内存管理 ppt-CSDN博客](https://blog.csdn.net/sunshineywz/article/details/106021469)

[Linux任督二脉之内存管理(一) PPT (qq.com)](https://mp.weixin.qq.com/s?__biz=Mzg2OTc0ODAzMw==&mid=2247501980&idx=1&sn=636fbb1c4c198a8e6177b03c6f71427f&source=41#wechat_redirect)