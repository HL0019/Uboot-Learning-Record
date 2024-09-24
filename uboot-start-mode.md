# 二、uboot的booti bootm bootefi命令启动的区别 内核的Image.itb zImage、Image Image.gz镜像有什么区别？

## 1.uboot的本质就是去引导kernel，针对不同的内核镜像，uboot提供了不同的启动方式。

要启动 Linux内核，需要先将 Linux 内核镜像文件拷贝到 DRAM 中，如果使用到设备树的话也需要将设备树拷贝到 DRAM 中。可以从 EMMC 、 NAND 、U盘和硬盘等存储设备中将 Linux 镜像和设备树文件拷贝到 DRAM，也可以通过网络 nfs 或者 tftp 将 Linux 镜像文件和设备树文件下载到 DRAM 中。不管用那种方法，只要能将 Linux 内核镜像和设备树文件存到 DRAM 中就行，然后使用 Uboot 命令用于启动镜像文件。
#### （1）bootm

* 引导 Uboot 自定义的内核镜像 uimage; bootm 用于将内核镜像加载到内存的指定地址处，如果有需要还要解压镜像，然后根据操作系统和体系结构的不同给内核传递不同的启动参数，最后启动内核。

* bootm addr [initrd[:size]] [fdt]

  * addr : kernel Image文件所在的memory地址，必选

  * [initrd[:size]] : initrd文件在memory中的地址位置和大小size，可以不指定，使用“-”代替即可

  * fdt : 设备树dtb文件在memory中的地址，在ARM64中，必选

    ~~~
    load mmc 0:1 $kernel_addr $kern_name
    load mmc 0:1 $fdt_addr $fdt_name
    bootm $kernel_addr - $fdt_addr
    或者
    load mmc 0:1 $kernel_addr $kern_name
    load mmc 0:1 $fdt_addr $fdt_name
    load mmc 0:1 $initrd_addr ramdisk.cpio.gz
    bootm $kernel_addr $initrd_addr:$filesize $fdt_addr
    ~~~

#### （2）booti

* booti 是 bootm 命令的一个子集，可用于执行位于内存中的ARM64 kernel Image，其格式：

  * booti 命令类似 bootm 命令 （三个参数依次是kernel地址、initrd地址、dtb地址），举例从`USB`中读取文件如下：

  ~~~
  load usb 0:1 $kernel_addr $kern_name
  load usb 0:1 $fdt_addr $fdt_name
  booti $kernel_addr - $fdt_addr
  或者
  load usb 0:1 $kernel_addr $kern_name
  load usb 0:1 $fdt_addr $fdt_name
  load usb 0:1 $initrd_addr ramdisk.cpio.gz
  booti $kernel_addr $initrd_addr:$filesize $fdt_addr
  ~~~

#### （3）bootz

* bootz 命令用于启动 zImage 镜像文件，也是最常用的。
  * ~~~
    * fatload mmc 1:1 0x80800000 zImage：
    
    * fatload mmc 1:1 0x83000000 imx6ull-14x14-evk.dtb：
    
    * bootz 0x80800000 - 0x83000000：启动 Linux
    ~~~
  
  * 具体作用就不多说了~~



#### （4）boot

* boot 命令也是用来启动 Linux 系统的，只是他会直接读取环境变量bootcmd来启动linux系统。

  * 假如现在使用tftp来启动，那么输入以下内容则可以直接启动

  * ~~~
    setenv bootcmd 'load mmc 0:1 $kernel_addr Image; load mmc 0:1 $fdt_addr xxx.dtb; booti $kernel_addr - $fdt_addr'
    saveenv
    boot
    ~~~

**前面说过，uboot 倒计时结束以后就会启动 Linux 系统，其实就是执行的 bootcmd 中的启动命令。只要不修改 bootcmd 中的内容，以后每次开机 uboot 倒计时结束以后都会存储设备拷贝 内核 和 dtb文件，然后启动 Linux。**



#### (5)uboot如何运行Image.gz

* 在 UBoot 中，有两种方式会用到 Image.gz 文件，一是通过 bootefi 命令引导系统时，使用的内核镜像格式；二是通过 booti/bootm 命令引导系统时，使用 gzip 压缩的内核镜像，则需要使用 gunzip 命令进行解压缩。
  以下是在 UBoot 中启动 gzip 压缩的内核镜像的示例命令：

  ~~~
  tftp $kernel_gz_addr Image.gz
  gunzip $kernel_gz_addr
  bootm $kernel_addr
  ~~~

* tftp:从 TFTP 服务器下载内核镜像到内存地址 0x91000000

* gunzip : 将内存地址 0x91000000 处的压缩文件解压缩到内存中

* bootm : 启动解压缩后的内核镜像，内核的起始地址为 0x90008000

**需要注意的是：**
**在使用 gunzip 命令解压缩内核镜像时，需要确保解压后的内存地址不会与其他代码或数据冲突。否则可能会导致系统崩溃或无法正常启动。**

#### (6)bootefi 命令

* 使用 bootefi 命令来启动， bootefi 命令用于启动 Image.gz 镜像文件， bootefi 命令格式如下：

* booti addr [initrd[:size]] [fdt]

  * 执行 bootm 命令（三个参数依次是kernel地址、initrd地址、dtb地址），举例通过网络工具`tftp`传输文件

  * ~~~
    load mmc 0:1 $kernel_addr EFI/BOOT/bootaa64.efi
    load mmc 0:1 $fdt_addr xxx.dtb
    bootefi $kernel_addr $fdt_addr
    ~~~

**上面多次提到了initrd，那么什么是initrd呢？**

initrd是 **initial RAM DISK的简写**，在系统引导过程中挂载的一个临时根文件系统，用来支持两阶段的引导过程。initrd文件中包含了各种可执行程序和驱动程序，它们可以用来挂载实际的根文件系统，然后再将这个initrd RAM DISK卸载，并释放内存。

以上就是uboot几种命令的区别（大致了解）下面所涉及到的问题，在后续学习中会一一解决！

## 2.然后再来看一下几种不同的image镜像文件格式有什么不同

* 内核编译之后会生成两个文件，一个Image，一个zImage，其中Image为内核映像文件，而zImage为内核的一种映像压缩文件，Image大约为4M，而zImage不到2M。

* 而image.gz则是一个经过gzip压缩过的内核镜像，如何去加载他在上述的booteif，bootm\booti中已写出

* Image.itb对我来说是陌生的，device tree在ARM架构中普及之后，u-boot也马上跟进、大力支持，毕竟，美好的Unify kernel的理想，需要bootloader的成全。为了支持基于device tree的unify kernel，u-boot需要一种新的Image格式，这种格式需要具备如下能力：
  * 1）Image中需要包含多个dtb文件。
    2）可以方便的选择使用哪个dtb文件boot kernel。
  * 综合上面的需求，u-boot推出了全新的image格式----FIT uImage，其中FIT是flattened image tree的简称。是不是觉得FIT和FDT（flattened device tree）有点像？没错，它利用了Device Tree Source files（DTS）的语法，生成的image文件也和dtb文件类似（称作itb）。参考文档：<https://zhuanlan.zhihu.com/p/615517515>

## #   kernel的dts里面的chosen有效还是uboot的bootargs有效？？？？

## #   如果均有效如何合并的？？？？

## #   如果定义两个重复的键 不同的值 谁的优先级高 谁覆盖谁？？？？

##   （顺便也把存储设备分区（基本都是gpt分区）搞清楚）

## #   fat分区 ext分区和gpt分区关系 谁是大哥？？？？

## #   initramfs的作用 如何生成 不用initramfs可不可以启动系统？？？？