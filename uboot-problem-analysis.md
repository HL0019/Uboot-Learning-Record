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



#### 







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

 
