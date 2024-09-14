# 						Uboot启动流程

### 1.什么是uboot

##### 当我们厌倦了裸机程序，而想要采用操作系统的时候，uboot就是不得不引入的一段程序。所以，uboot就是一段引导程序，在加载系统内核之前，完成硬件初始化，内存映射，为后续内核的引导提供一个良好的环境。uboot是bootloader的一种，全称为universal boot loader

***



### （需要搞清楚的问题）：

### （1）知道uboot也有设备树吗？？？？

#### 			uboot，全名Universal Boot Loader，顾名思义 “通用的”Boot Loader，既然为通用的，那么就是说明它一定可以支持很多的芯片，很多的架构，比如 ARM,RISC-V,X86....，这些架构又可以衍生出很多的芯片厂商，ARM为例，支持三星的芯片，NXP,ST......，而这些芯片又可以衍生出很多的板卡制作厂商，然后就到了我们用户，因为uboot启动内核之前肯定是先要去初始化硬件，就用我现在使用的imx6ull mini为例，假如它的网卡驱动可能接到了pin1引脚，而imx6ull 阿尔法接到了pin2引脚，因为他们的硬件资源不同，所以就要使用不同的网卡配置文件，而uboot源码里就含有了这一系列的配置文件，使得代码及其庞大臃肿，故uboot引出了设备树的概念，对这些.dtb文件进行管理。

#### ubbot其实就是uboot.bin文件加上某个dtb文件。

***



- #### XIP（executed in place）本地执行。操作系统采用这种系统，可以不用将内核或执行代码拷贝到内存，而直接在代码的存储空间直接运行。XIP是一种能够直接在闪速存储器中执行代码而无须装载到RAM中执行的机制；

  - #### 非XIP设备就是如SD卡，USB，UART，网卡等不同的启动方式，这些启动程序一般都是储存在ROM中的，CPU会与ROM直接通信，ROM由于是只读设备，其中的程序需要拷贝到内存中去执行，这个过程称为并非XIP机制。

#### 	XIP机制作用：减少了内核从Flash拷贝到RAM的时间。但是XIP还是需要硬件的支持，只有NOR型的Flash才可以随机存取，就是因为它符合CPU去指令译码执行的要求，CPU给出一个地址，NorFlash就能给出一个数据让CPU去执行，中间不需要额外的处理操作。

- #### Uboot它支持SD卡启动，但CPU又不能直接去访问到SD卡，那么中间必然就有BRoom（bootrom），而CPU可以直接访问到bootrom，上电后执行的是bootrom里固化的程序，它会帮你把SD卡的uboot复制到内存进行执行。所以说上电后首先执行的不一定是uboot。

- #### uboot大致的启动流程：无疑分为两种，XIP设备启动，非XIP设备启动。

  - #### XIP：a.执行硬件初始化	b.从flash中将内核复制到ram	c.启动内核

  - #### 非XIP：a.bootrom将内核拷贝到ram中      b.执行uboot    c.硬件初始化（不再初始化ram）    d.    .从flash中将内核复制到ram    e.启动内核



## （2）知道uboot的设备树可以合并到kernel吗？？？？



## （3）知道uboot bootargs如何传递给内核吗？？？？

## （4）知道kernel的dts里面的chosen有效还是uboot的bootargs有效吗？？？？

## （5）如果均有效如何合并的？？？？

## （6）如果定义两个重复的键 不同的值 谁的优先级高 谁覆盖谁？？？？

## （顺便也把存储设备分区（基本都是gpt分区）搞清楚）

## （7）fat分区 ext分区和gpt分区关系 谁是大哥？？？？

## （8）uboot的booti bootm bootfit命令启动的区别 内核的Image.itb zImage Image Image.gz镜像区别？？？？

## （9）initramfs的作用 如何生成 不用initramfs可不可以启动系统？？？？
