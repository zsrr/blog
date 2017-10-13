---
title: VirtualBox为CentOS7配置网络并与主机通信
date: 2017-10-12 22:04:47
tags: [VirtualBox,CentOS]
categories: 虚拟机

---
再不学虚拟机，你将失去你的女神……

<!--more-->

配置过程很简单，长话短说，前提是VirtualBox安装好CentOS。关于VirtualBox提供的几种网络模式，参见：[VirtualBox虚拟机网络设置](https://www.douban.com/group/topic/15558388/)。笔者采用NAT + Host Adapter双网卡模式，前者管的是系统与外界的通信，后者是主机与系统的通信。

# 配置网卡
首先要为VirtualBox设置网卡，如图所示：
![](http://ok34fi9ya.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-10-12%20%E4%B8%8B%E5%8D%8810.13.53.png)
![](http://ok34fi9ya.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-10-12%20%E4%B8%8B%E5%8D%8810.14.42.png)
![](http://ok34fi9ya.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-10-12%20%E4%B8%8B%E5%8D%8810.15.19.png)
![](http://ok34fi9ya.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-10-12%20%E4%B8%8B%E5%8D%8810.17.22.png)

这样就配置完成了，接下来为CentOS配置网卡：

![](http://ok34fi9ya.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-10-12%20%E4%B8%8B%E5%8D%8810.19.13.png)
![](http://ok34fi9ya.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-10-12%20%E4%B8%8B%E5%8D%8810.19.59.png)

至此配置完毕，注意要记住为CentOS配置的两网卡的MAC地址，后面要用。
# 配置CentOS
此时CentOS还不能连接网络。首先要用管理员用户登录CentOS，先修改**/etc/sysconfig/network-scripts/ifcfg-enp0s3**文件，添加HWADDR, 其值为刚才为CentOS设置的NAT网卡的MAC值；再将ONBOOT改为yes, BOOTPROTO设置为dhcp, 更改完成后退出，完整的配置如图所示：
![](http://ok34fi9ya.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-10-12%20%E4%B8%8B%E5%8D%8810.29.32.png)

接下来利用指令``cp ifcfg-enp0s3 ifcfg-enp0s8``来复制一份ifcfg-enp0s3文件，并将其名改为ifcfg-enp0s8，对此文件更改如下：

- HWADDR改成为CentOS配置的网卡2的MAC地址。
- BOOTPROTO设置为static。
- NAME和DEVICE设置为enp0s8。
- UUID随便改一个，长度保持一样，只改最后两位好了。
- 增加IPADDR, 其值为刚才为VirtualBox配置的Host-Only网络网卡中的IPv4地址。
- 增加NETMASK, 其值为刚才为VirtualBox配置的Host-Only网络网卡中的IPv4网络掩码。

保存，退出。

利用``service network restart``指令重启网络，先试一下与外界的连通，``ping www.baidu.com``，在试一下与本地主机的连通，``ping 自己主机的ip地址``。

# 与主机进行通信
与主机通信最直接的途径就是ssh指令和scp指令，笔者的主机是mac，所以要先在系统偏好设置 ---> 共享中打开远程登录选项，再在虚拟机中利用ssh指令远程登录自己的主机。



