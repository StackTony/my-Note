---
tags:
  - virtio
---

从代码上看，virtio的代码主要分两个部分：QEMU和内核驱动程序。Virtio设备的模拟就是通过QEMU完成的，QEMU代码在虚拟机启动之前，创建虚拟设备。虚拟机启动后检测到设备，调用内核的virtio设备驱动程序来加载这个virtio设备。
对于KVM虚拟机，都是通过QEMU这个用户空间程序创建的，每个KVM虚拟机都是一个QEMU进程，虚拟机的virtio设备是QEMU进程模拟的，虚拟机的内存也是从QEMU进程的地址空间内分配的。
vring是由虚拟机virtio设备驱动创建的用于数据传输的共享内存，QEMU进程通过这块共享内存获取前端设备递交的IO请求。

Virtio整体框架
---
虚拟机IO请求的整个流程：（*<u>Virtio完成的就是guest和qemu的IO请求数据的交互，后续的qemu模拟设备的IO调度和虚拟中断是在Virtio之后发生的</u>*）

KVM内核态的网络转发示意图：（此图中虚拟机和物理机均使用的内核态转发）
来自 <https://cloud.tencent.com/developer/article/1540284>

虚拟机使用dpdk轮询方式后时延情况

![image4](../../resources/4fec4f51f5e24a94976703232874cfd9.png)
对比图（左侧为虚拟机内部不使用dpdk轮询方式，通过KVM注入中断通知虚拟机，走virtio_pci_set_guest_notifiers函数）

![image5](../../resources/6a36f7337a6748b7ac80380d99267ea8.png)

虚拟机内不使用dpdk，使用原始方式的时延情况
![image6](../../resources/8879c447d8c345a3903a70a510eec927.png)
此图中的模式为虚拟机内使用KVM内核态组网，主机侧使用DPDK轮询
![image7](../../resources/4a141ed03ae54a75b64ef6ec9d07e2ad.png)

![image8](../../resources/7043d9edc1e84a8889dfa1d1e6be6bbf.png)

<https://blog.csdn.net/weixin_28927927/article/details/113027310>

![image9](../../resources/eefe0b724da84dc2b8e4ef7b2d194bca.png)
来自 <https://www.cnblogs.com/LoyenWang/p/14399642.html>

virtio从前端到后端的virtio IO请求/处理的整体流程：
1、guest通过对已经注册了ioeventfd的一块区域进行写入操作导致guest/host切换，vcpu从guest mode陷出到KVM
2、vcpu线程在KVM模块中通过eventfd_signal来唤醒IO thread的poll，此时vcpu线程陷入返回到guest mode
3、IO thread被唤醒，在内核态调度执行返回用户态
4、从avail vring中取出请求下发
5、aio系统调用
6、通过host 的device driver将数据传到物理设备
7、硬件处理完IO请求之后通过物理中断唤醒IO主线程
8、IO主线程再次从内核被调度返回用户态
9、根据IO处理的结果在used vring中保存记录
10、向注册过irqfd的fd进行写入操作
11、KVM中irqfd的等待队列被唤醒，向目标vcpu添加一个request请求，然后判断vcpu是否在物理cpu上运行，如果是就让vcpu退出guest mode，方便中断注入
12、vcpu陷出到host mode之后，再次调用vcpu_enter_guest回到guest mode时，从request中读出注入中断的请求，写入vmcs的IRQ寄存器，来真正注入中断，vcpu回到guest mode之后会调用guest中注册的中断处理函数进行处理

![image10](../../resources/cff7f5b8c1964da6953436caeaa3ce38.png)

来自 <http://3ms.huawei.com/km/blogs/details/5468787>

![image11](../../resources/f15d30e8a5e64f2aa07e5f09c07355d0.png)

![image12](../../resources/89e0903950ba40dea9c92f811444b2b5.png)

HOST和客户机也正是通过VirtQueue来操作buffer。每个buffer包含一个VRing结构，对buffer的操作实际上是通过VRing来管理的。

两个生产者-消费者模型:
前端驱动可以看做请求的生产者和响应的消费者
后端驱动看做请求的消费者和响应的生产者
![image13](../../resources/ceafdbcd7d444c5baf75cf3d89a1c304.png)

![image14](../../resources/512bb7d51c24414a9a183b75b3631382.png)

![image15](../../resources/4fbc8ac6afc84b9fbf117d7a8a3729f8.png)

1.  虚拟机产生的IO请求会被前端的virtio设备接收，并存放在virtio设备散列表scatterlist里；
2.  Virtio设备的virtqueue提供add_buf将散列表中的数据映射至前后端数据共享区域Vring中；
3.  Virtqueue通过kick函数来通知前端qemu进程。Kick通过写pci配置空间的寄存器产生kvm_exit；
4.  Qemu端注册ioport_write/read函数监听PCI配置空间的改变，获取前端的通知消息；
5.  Qemu端维护的virtqueue队列从数据共享区vring中获取数据
6.  Qemu将数据封装成virtioreq;
7.  Qemu进程将请求发送至硬件层。

前后端主要通过PCI配置空间的寄存器完成前后端的通信，而IO请求的数据地址则存在vring中，并通过共享vring这个区域来实现IO请求数据的共享。
Virtio设备的驱动分为前端与后端：前端是虚拟机的设备驱动程序，后端是host上的QEMU用户态程序。为了实现虚拟机中的IO请求从前端设备驱动传递到后端QEMU进程中，Virtio框架提供了两个核心机制：<u>前后端消息通知机制</u>和<u>数据共享机制</u>。

