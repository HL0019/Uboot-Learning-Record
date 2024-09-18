# 一、什么是bootcmd、bootargs，有什么作用？如何传递给kernel？

#### uboot中有很多环境变量，下面介绍其中两个bootcmd和bootargs，这两个环境变量都是NXP自己定义的。

* #### 首先看bootcmd，bootcmd保存着uboot默认命令，uboot倒计时结束以后就会执行bootcmd中的命令，这些命令一般用来启动linu内核。可以在 uboot 启动以后进入命令行设置 bootcmd 环境变量的值。如果 EMMC 或者 NAND 中没有保存 bootcmd 的值，那么 uboot 就会使用默认的值，板子第一次运行 uboot 的时候都会使用默认值来设置 bootcmd 环境变量。

  * 这里就和之前分析的 autoboot_command 、abortboot_single_key等函数呼应上了。















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

