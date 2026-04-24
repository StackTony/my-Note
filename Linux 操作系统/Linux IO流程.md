---
tags:
  - IO
---

#### Linux的IO栈：
<https://zhuanlan.zhihu.com/p/39721251>
![[Pasted image 20260424165505.png]]


确认下发到磁盘的io是透传还是写回模式
```
cat /sys/class/scsi_disk/0\:2\:0\:0/cache_type
```
当选用write through方式时，系统的写磁盘操作并不利用阵列卡的Cache，而是直接与磁盘进行数据的交互。 而write Back方式则利用阵列Cache作为系统与磁盘间的二传手，系统先将数据交给Cache，然后再由Cache将数据传给磁盘。


简易流程
Linux本地IO下发过程：
[https://blog.csdn.net/invaders/article/details/126449378?utm_medium=distribute.pc_relevant.none-task-blog-2~default~baidujs_baidulandingword~default-1-126449378-blog-121689114.235^v36^pc_relevant_anti_vip_base&spm=1001.2101.3001.4242.2&utm_relevant_index=4](https://blog.csdn.net/invaders/article/details/126449378?utm_medium=distribute.pc_relevant.none-task-blog-2~default~baidujs_baidulandingword~default-1-126449378-blog-121689114.235%5ev36%5epc_relevant_anti_vip_base&spm=1001.2101.3001.4242.2&utm_relevant_index=4)

1）通用BLOCK层，把请求排队
2）通用BLOCK层，调度层
3）SCSI层提交给驱动层等待返回
4）返回处理成功or超时


详细流程：
linux文件写入流程-从vfs层通用block层到scsi磁盘驱动
<https://gmd20.github.io/blog/linux%E6%96%87%E4%BB%B6%E5%86%99%E5%85%A5%E6%B5%81%E7%A8%8B-%E4%BB%8Evfs%E5%B1%82%E9%80%9A%E7%94%A8block%E5%B1%82%E5%88%B0scsi%E7%A3%81%E7%9B%98%E9%A9%B1%E5%8A%A8/>、



#### IO调度算法：
<https://cloud.tencent.com/developer/article/1615744>
