# 						Uboot启动流程

### 1.什么是uboot

##### 当我们厌倦了裸机程序，而想要采用操作系统的时候，uboot就是不得不引入的一段程序。所以，uboot就是一段引导程序，在加载系统内核之前，完成硬件初始化，内存映射，为后续内核的引导提供一个良好的环境。uboot是bootloader的一种，全称为universal boot loader



### （需要搞清楚的问题）：

### （1）知道uboot也有设备树吗？？？？

#### 		uboot，全名Universal Boot Loader，顾名思义 “通用的”Boot Loader，既然为通用的，那么就是说明它一定可以支持很多的芯片，很多的架构，比如 ARM,RISC-V,X86....，这些架构又可以衍生出很多的芯片厂商，ARM为例，支持三星的芯片，NXP,ST......，而这些芯片又可以衍生出很多的板卡制作厂商，然后就到了我们用户，因为uboot启动内核之前肯定是先要去初始化硬件，就用我现在使用的imx6ull mini为例，假如它的网卡驱动可能接到了pin1引脚，而imx6ull 阿尔法接到了pin2引脚，因为他们的硬件资源不同，所以就要使用不同的网卡配置文件，而uboot源码里就含有了这一系列的配置文件，使得代码及其庞大臃肿，故uboot也会使用设备树文件来对这些配置文件进行修改，从而引出了uboot的设备树。

（未完成）后续的启动流程；什么是XIP设备，什么是非XIP设备；

## （2）知道uboot的设备树可以合并到kernel吗？？？？

## （3）知道uboot bootargs如何传递给内核吗？？？？

## （4）知道kernel的dts里面的chosen有效还是uboot的bootargs有效吗？？？？

## （5）如果均有效如何合并的？？？？

## （6）如果定义两个重复的键 不同的值 谁的优先级高 谁覆盖谁？？？？

## （顺便也把存储设备分区（基本都是gpt分区）搞清楚）

## （7）fat分区 ext分区和gpt分区关系 谁是大哥？？？？

## （8）uboot的booti bootm bootfit命令启动的区别 内核的Image.itb zImage Image Image.gz镜像区别？？？？

## （9）initramfs的作用 如何生成 不用initramfs可不可以启动系统？？？？
