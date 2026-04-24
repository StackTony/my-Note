---
tags:
  - virtio
---

virtio介绍
<https://www.cnblogs.com/LoyenWang/p/14399642.html>

virtio详讲+代码分析
<https://blog.csdn.net/xidianjiapei001/article/details/89293914>

kvm介绍：IO虚拟化
<https://www.cnblogs.com/sammyliu/p/4543657.html>

virtio概述
<https://xgwang0.github.io/2019/03/01/KVM-Qemu-VirtIO/>

virtio和vhost（virtio的一种后端实现）
<https://zhuanlan.zhihu.com/p/38370357>

ioeventfd机制
<https://www.cnblogs.com/haiyonghao/p/14440743.html>

（guest写pci配置空间陷出到kvm。kvm通过ioeventfd通知qemu）

<span style='color:#222222'>整个ioeventfd的逻辑流程如下：</span>
1.  <span style='color:#222222'>QEMU分配一个eventfd，并将该eventfd加入KVM维护的eventfd数组中</span>
2.  <span style='color:#222222'>QEMU向KVM发送更新eventfd数组内容的请求</span>
3.  <span style='color:#222222'>QEMU构造一个包含IO地址，IO地址范围等元素的ioeventfd结构，并向KVM发送注册ioeventfd请求</span>
4.  <span style='color:#222222'>KVM根据传入的ioeventfd参数内容确定该段IO地址所属的总线，并在该总线上注册一个ioeventfd虚拟设备，该**虚拟设备的write**方法也被注册</span>
5.  <span style='color:#222222'>Guest执行OUT类指令(包括MMIO Write操作)</span>
6.  <span style='color:#222222'>VMEXIT到KVM</span>
7.  <span style='color:#222222'>调用虚拟设备的write方法</span>
8.  <span style='color:#222222'>write方法中检查本次OUT类指令访问的IO地址和范围是否符合ioeventfd设置的要求</span>
9.  <span style='color:#222222'>如果符合则调用eventfd_signal触发一次POLLIN事件并返回Guest</span>
10. <span style='color:#222222'>QEMU监测到ioeventfd上出现了POLLIN，则调用相应的处理函数处理IO</span>
