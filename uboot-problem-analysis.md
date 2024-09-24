# 一、什么是bootcmd、bootargs，有什么作用？如何传递给kernel？

#### （1）uboot中有很多环境变量，下面介绍其中两个bootcmd和bootargs。

* #### 首先看bootcmd，bootcmd保存着uboot默认命令，uboot倒计时结束以后就会执行bootcmd中的命令，这些命令一般用来启动linu内核。可以在 uboot 启动以后进入命令行设置 bootcmd 环境变量的值。如果 EMMC 或者 NAND 中没有保存 bootcmd 的值，那么 uboot 就会使用默认的值，板子第一次运行 uboot 的时候都会使用默认值来设置 bootcmd 环境变量。

  * 这里就和之前分析的 autoboot_command 、abortboot_single_key等函数呼应上了。

* #### uboot的所有默认环境变量在 include/env_default.h文件中

~~~
#ifdef  CONFIG_BOOTCOMMAND
          "bootcmd="      CONFIG_BOOTCOMMAND              "\0"
~~~

#### 而 CONFIG_BOOTCOMMAND 这个宏又是在在各个开发板的配置头文件中

##### ./configs/mx6ul_14x14_evk.h。

~~~
#define CONFIG_BOOTCOMMAND \
           "run findfdt;" \
           "mmc dev ${mmcdev};" \
           "mmc dev ${mmcdev}; if mmc rescan; then " \
                   "if run loadbootscript; then " \
                           "run bootscript; " \
                   "else " \
                           "if run loadimage; then " \
                                   "run mmcboot; " \
                           "else run netboot; " \
                           "fi; " \
                   "fi; " \
           "else run netboot; fi"
#endif
~~~

#### **首先运行findfdt命令确定设备树文件的名字，然后切换到emmc，接下来执行loadimage加载内核镜像，最后执行mmcboot，也就是加载设备树文件，并执行bootz命令启动内核**。

假如现在要从EMMC启动，那么宏 CONFIG_BOOTCOMMAND 可以简化为：

~~~
#define CONFIG_BOOTCOMMAND \
"mmc dev 1;" \
"fatload mmc 1:1 0x80800000 zImage;" \
"fatload mmc 1:1 0x83000000 imx6ull-hyq-emmc.dtb;" \
"bootz 0x80800000 - 0x83000000;"
~~~

或者可以直接通过在uboot中设置bootcmd的值：

~~~
setenv bootcmd 'mmc dev 1; fatload mmc 1:1 80800000 zImage; fatload mmc 1:1 83000000 imx6ull-hyq-emmc.dtb; bootz 80800000 - 83000000;'
saveenv
~~~



* #### 然后是bootargs

~~~
ifdef  CONFIG_BOOTARGS
        "bootargs="     CONFIG_BOOTARGS                 "\0"
~~~



~~~
	"mmcboot=echo Booting from mmc ...; " \
		"run mmcargs; " \
		"if test ${boot_fdt} = yes || test ${boot_fdt} = try; then " \
			"if run loadfdt; then " \
				"bootz ${loadaddr} - ${fdt_addr}; " \
			"else " \
				"if test ${boot_fdt} = try; then " \
					"bootz; " \
				"else " \
					"echo WARN: Cannot load the DT; " \
				"fi; " \
			"fi; " \
		"else " \
			"bootz; " \
		"fi;\0" \
~~~

#### 首先执行的就是 mmcargs 环境变量：

~~~
"mmcargs=setenv bootargs console=${console},${baudrate} " \
    CONFIG_BOOTARGS_CMA_SIZE \
    "root=${mmcroot}\0" \
    
    展开后为：
    mmcargs=setenv bootargs console= ttymxc0, 115200 root= /dev/mmcblk1p2 rootwait rw
~~~



#### 从上面我们不难看出，环境变量 mmcargs 就是设置 bootargs 的值为“console= ttymxc0, 115200 root= /dev/mmcblk1p2 rootwait rw”，bootargs 就是设置了很多的参数的值，这些参数 Linux 内核会使用到。



***



#### （2）U-Boot向kernel传递的命令行参数属于环境变量的一种，可以说，整个U-Boot的环境变量都是围绕着命令行参数展开的。u-boot在启动内核时，会向内核传递一些参数，可以通过两种方式传递参数给内核，一种是通过参数链表(ttagged list)方式，一种是通过设备树方式。这些参数主要包括 系统的根设备标志，页面大小，内存的起始地址和大小等参数。 

* 在kernel启动后，我们可以通过  cat /proc/cmdline 来查看bootargs的值：

  ~~~
  console=ttymxc0,115200 root=/dev/mmcblk1p2 rootwait rw
  ~~~

* 当然在打印信息也会看到 Kernel command line 这一行。

.............剩下的还没研究懂，懵逼



#### （3）bootargs、bootcmd参数的含义

* **bootargs各个参数的含义：bootargs 保存着 uboot 传递给 Linux 内核的参数**
* **从EMMC启动根文件系统：**
  * setenv bootargs 'console=ttymxc0,115200 root=/dev/mmcblk1p2 rootwait rw'
  * console=ttymxc0 ：设置 linux 终端，串口 1 的设备文件是/dev/ttymxc0
  * 115200 串口波特率
  * root=/dev/mmcblk1p2：根文件系统存放在 mmcblk1 设备的分区 2 ，即EMMC 的分区 2 。`/dev/mmcblkx(x=0~n)`表示 mmc 设备，而`/dev/mmcblkxpy(x=0~n,y=1~n)`表示 mmc 设备 x 的分区 y
  * rootwait：表示等待 mmc 设备初始化完成以后再挂载，否则的话 mmc 设备还没初始化完成就挂载根文件系统会出错的
  * rw：表示根文件系统是可以读写的，不加 rw 的话可能无法在根文件系统中进行写操作，只能进行读操作
* **网络启动根文件系统：**（重复的不在叙述）
  * **setenv bootargs 'console=ttymxc0,115200 root=/dev/nfs /**
  * **nfsroot=192.168.137.18:/home/pjw/linux/nfs/rootfs,proto=tcp rw ip=192.168.137.20:192.168.137.18:192.168.1.1:255.255.255.0::eth0:off'**
  * root=/dev/nfs：根文件系统存放在 nfs 挂载目录
  * nfsroot=192.168.137.18:/home/xxx/linux/nfs/rootfs：服务器 IP 地址：即根文件系统存放路径
  * proto=tcp rw：使用 TCP 协议
  * ip=192.168.137.20：客户端 IP 地址（也就是开发板的IP地址）
  * 192.168.137.18:192.168.1.1:255.255.255.0：服务器 IP 地址，网关地址，子网掩码
  * eth0：设备名
  * off：自动配置，一般不使用，所以设置为 off



* **bootcmd各个参数含义，bootcmd 保存着 uboot 默认命令。**

* **从EMMC启动linux系统：**

  * 首先先查看一下EMMC 分区1

    ~~~
    => ls mmc 1:1
        39459   imx6ull-14x14-emmc-10.1-1280x800-c.dtb
        39527   imx6ull-14x14-emmc-4.3-480x272-c.dtb
        39459   imx6ull-14x14-emmc-4.3-800x480-c.dtb
        39459   imx6ull-14x14-emmc-7-1024x600-c.dtb
        39459   imx6ull-14x14-emmc-7-800x480-c.dtb
        40295   imx6ull-14x14-emmc-hdmi.dtb
        40203   imx6ull-14x14-emmc-vga.dtb
      6785480   zimage
        39459   imx6ull-14x14-emmc-4.3-480x272-c.dt.1
        
        
    => ls mmc 1:2
    <DIR>       4096 .
    <DIR>       4096 ..
    <DIR>      16384 lost+found
    <DIR>       4096 sbin
    <DIR>       4096 dev
    <DIR>       4096 boot
    <DIR>       4096 etc
    <DIR>       4096 lib
    <DIR>       4096 usr
    <DIR>       4096 proc
    <SYM>          8 tmp
    <DIR>       4096 var
    <DIR>       4096 opt
    <DIR>       4096 home
    <DIR>       4096 run
    <DIR>       4096 bin
    <DIR>       4096 media
    <DIR>       4096 mnt
    <DIR>       4096 .cache
    <DIR>       4096 sys
    <DIR>       4096 .config
    <DIR>       4096 .local
    < ? >          0 psplash_fifo
                   0 orcexec.3f18xI
    
    ~~~

    可以看到现在EMMC的分区1和2中是存在设备树文件、zimage和根文件系统的

  * **setenv bootcmd 'mmc dev 1; fatload mmc 1:1 80800000 zImage; fatload mmc 1:1 83000000 imx6ull-alientek-emmc.dtb; bootz 80800000 - 83000000;'**

  * mmc dev 1：切换设备到EMMC

  * fatload mmc 1:1 0x80800000 zImage：读取 zImage 到 0x80800000 处

  * fatload mmc 1:1 0x83000000 imx6ull-14x14-evk.dtb：读取设备树到 0x83000000 处

  * bootz 0x80800000 - 0x83000000：启动 Linux（这里用到的启动方式是bootz启动，具体区别下面再说）

* 从网络启动 Linux 系统（使用TFTP文件夹中的镜像文件 zImage 和设备树）

  * setenv bootcmd 'tftp 80800000 zImage; tftp 83000000 imx6ull-alientek-emmc.dtb; bootz 80800000 - 83000000'

以上就是当下阶段对于bootcmd和bootargs的理解，下面将说一下uboot的几种枪方式的区别~~~~

***







***

## #   uboot的设备树可以合并到kernel？？？？

## #   uboot bootargs如何传递给内核？？？？

## #   kernel的dts里面的chosen有效还是uboot的bootargs有效？？？？

## #   如果均有效如何合并的？？？？

## #   如果定义两个重复的键 不同的值 谁的优先级高 谁覆盖谁？？？？

##   （顺便也把存储设备分区（基本都是gpt分区）搞清楚）

## #   fat分区 ext分区和gpt分区关系 谁是大哥？？？？

## #   uboot的booti bootm bootfit命令启动的区别 内核的Image.itb zImage          	Image Image.gz镜像区别？？？？

## #   initramfs的作用 如何生成 不用initramfs可不可以启动系统？？？？

 
