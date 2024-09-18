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
  
  - ### 以上只是对uboot启动流程的大致分析，接下来通过uboot源码来进行详细的分析。
  
    
  
  ## （一）链接脚本 u-boot.lds 详解
  
  #### U-Boot 源码文件众多，我们如何知道最开始的启动文件（程序入口）是哪个呢？程序的链接是由链接脚本来决定的，所以通过链接脚本可以找到程序的入口，链接脚本为arch/arm/cpu/u-boot.lds，它描述了如何生成最终的二进制文件，其中就包含程序入口。
  
  注：**只有编译 u-boot 以后才会在根目录下出现 u-boot.lds 文件！****打开 u-boot.lds，内容如下：**
  
  
  
  ~~~
  /* 指定输出可执行文件: "elf 32位 小端格式 arm指令" */
  1 OUTPUT_FORMAT( ("elf32-littlearm", , "elf32-littlearm", , "elf32-littlearm") )
  /* 指定输出可执行文件的目标架构:"arm" */
  2 OUTPUT_ARCH(arm)
  /* 指定输出可执行文件的起始地址为:"_start" */
  3 ENTRY(_start)
  .........
  ~~~
  
  ####  由于在链接脚本中规定了文件start.o(对应于start.S)作为整个uboot的起始点，因此启动uboot时会执行首先执行start.S。 _start 在文件 arch/arm/lib/vectors.S 中有定义:
  
  ~~~
  .globl _start
      .section ".vectors", "ax"
  
  #if defined(CONFIG_ENABLE_ARM_SOC_BOOT0_HOOK)
  #include <asm/arch/boot0.h>
  #else
  
  _start:
  #ifdef CONFIG_SYS_DV_NOR_BOOT_CFG
      .word   CONFIG_SYS_DV_NOR_BOOT_CFG
  #endif
      ARM_VECTORS
  #endif /* !defined(CONFIG_ENABLE_ARM_SOC_BOOT0_HOOK) */
  
  #if !CONFIG_IS_ENABLED(SYS_NO_VECTOR_TABLE)
  
      .globl  _reset
      .globl  _undefined_instruction     /* 未定义指令异常 */
      .globl  _software_interrupt        /* 软中断异常 */
      .globl  _prefetch_abort            /* 预取异常 */
      .globl  _data_abort                /* 数据异常 */
      .globl  _not_used                  /* 未使用 */
      .globl  _irq                       /* 外部中断请求IRQ */
      .globl  _fiq                       /* 快束中断请求FIQ */
      ...
  ~~~
  
  #### 可以看出，_start 后面就是中断向量表，从图中的“.section “.vectors”, "ax”可以得到，此代码存放在.vectors 段里面。
  
  ***
  
  
  
  
  
  ## （二）u-boot.map文件分析
  
  ~~~
  内存配置
  
  名称           来源             长度             属性
  *default*        0x00000000         0xffffffff
  
  链结器命令稿和内存映射
  
  段 .text 的地址设置为 0x87800000
                  0x00000000                . = 0x0
                  0x00000000                . = ALIGN (0x4)
  
  .text           0x87800000      0x3a8
   *(.__image_copy_start)
   .__image_copy_start
                  0x87800000        0x0 arch/arm/lib/sections.o
                  0x87800000                __image_copy_start
   *(.vectors)
   .vectors       0x87800000      0x2e8 arch/arm/lib/vectors.o
                  0x87800000                _start
                  0x87800020                _undefined_instruction
                  0x87800024                _software_interrupt
                  0x87800028                _prefetch_abort
                  0x8780002c                _data_abort
                  0x87800030                _not_used
                  0x87800034                _irq
                  0x87800038                _fiq
                  0x87800040                IRQ_STACK_START_IN
   arch/arm/cpu/armv7/start.o(.text*)
   .text          0x878002e8       0xc0 arch/arm/cpu/armv7/start.o
                  0x878002e8                reset
                  0x878002ec                save_boot_params_ret
                  0x87800328                c_runtime_cpu_setup
                  0x87800338                save_boot_params
                  0x8780033c                cpu_init_cp15
                  0x8780039c                cpu_init_crit
  ...
  ~~~
  
  - #### u-boot.map 是 uboot 的映射文件，可以从此文件看到某个文件或者函数链接到了哪个地址，从图中可以看到__image_copy_start 为 0X87800000，而.text 的起始地址也是0X87800000。
  
  - #### 继续回到u-boot.lds中,vectors 段保存中断向量表，从vector.S中我们知道了 vectors.S 的代码是存在 vectors 段中的。从u-boot.map可以看出，vectors 段的起始地址也是0X87800000，说明整个 uboot 的起始地址就是 0X87800000，这也是为什么我们裸机例程的链接起始地址选择 0X87800000 了，目的就是为了和 uboot一致。
  
  - #### 在 u-boot.lds 中有一些跟地址有关的“变量”需要我们注意一下
  
  - #### 在 u-boot.map 文件中查找，除了__image_copy_start以外，其他的变量值每次编译的时候可能会变化，如果修改了 uboot 代码、修改了 uboot 配置、选用不同的优化等级等等都会影响到这些值。
  
    ***
  
    
  
  ## **（三）**reset**函数详解**
  
  #### 从程序入口_start定义中得出，_start中首先是跳转到reset函数，具体代码如下：
  
  ~~~
      .globl  reset
      .globl  save_boot_params_ret
      .type   save_boot_params_ret,%function
  #ifdef CONFIG_ARMV7_LPAE
      .global switch_to_hypervisor_ret
  #endif
  
  reset:
      /* Allow the board to save important registers */
      b   save_boot_params
  save_boot_params_ret:
      ...
  ~~~
  
  
  
  代码首先就是跳转到了 save_boot_params 函数。
  
  ~~~
  ENTRY(save_boot_params)
           b       save_boot_params_ret            @ back to my caller
  ENDPROC(save_boot_params)
  ~~~
  
  
  
  save_boot_params 函数也只有跳转到 save_boot_params_ret函数这一句。
  
  ~~~
  save_boot_params_ret:
  mrs     r0, cpsr
  and     r1, r0, #0x1f           @ mask mode bits
  teq     r1, #0x1a               @ test for HYP mode
  bicne   r0, r0, #0x1f           @ clear all mode bits
  orrne   r0, r0, #0x13           @ set SVC mode
  orr     r0, r0, #0xc0           @ disable FIQ and IRQ
  msr     cpsr,r0					@//将r0的内容写入cpsr寄存器
  
  ~~~
  
  
  
  **真正的初始化从这里开始了。其实在CPU一上电以后就是跳到这里执行的，更改处理器模式为SVC模式。****这就需要查看ARMv7的芯片资料了：**
  
  ![image-20240918101019520](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\image-20240918101019520.png)
  
  其中，M[4:0]是对处理器模式进行配置的位：
  
  ![image-20240918101048361](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\image-20240918101048361.png)
  
  **也就是配置成10011。同理需要配置禁止`IRQ`、`FIQ`中断，即I、F位为1。这样算来，需要给的值为1101 0011，即0xD3.**
  
  
  
  **紧接着：**
  
  ~~~
  #if !CONFIG_IS_ENABLED(SYS_NO_VECTOR_TABLE)
  /*
   * Setup vector:
   */
      /* Set V=0 in CP15 SCTLR register - for VBAR to point to vector */
      mrc p15, 0, r0, c1, c0, 0   @ Read CP15 SCTLR Register
      bic r0, #CR_V       @ V = 0
      mcr p15, 0, r0, c1, c0, 0   @ Write CP15 SCTLR Register
  
  #ifdef CONFIG_HAS_VBAR
      /* Set vector address in CP15 VBAR register */
      ldr r0, =_start
      mcr p15, 0, r0, c12, c0, 0  @Set VBAR
  #endif
  #endif
  ~~~
  
  **这部分代码通过对协处理器`CP15`进行操作，设置了处理器的异常向量入口地址为`_start`。这是因为ARM默认的异常向量表入口在0x0地址，然而 IMX6ULL 系统上电或硬件复位后，CPU从0x0000 0000 地址开始 执行Boot ROM代码（以下简称为“Boot ROM代码”）。Boot ROM代码首先会检查CPU的ID。不可修改，自然不可能存放异常向量表，所以需要修改异常向量表入口，将它们映射到其他位置上去。**
  
  **save_boot_params_ret函数主要的操作如下：**
  
  * **1.如果定义宏CONFIG_POSITION_INDEPENDENT，则进行修正重定位的问题（pie_fixup、pie_fix_loop、pie_skip_reloc）；**
  * **2.如果定义宏CONFIG_ARMV7_LPAE，LPAE（Large Physical Address Extensions）是ARMv7系列的一种地址扩展技术，可以让32位的ARM最大能支持到1TB的内存空间，由于嵌入式ARM需求的内存空间一般不大，所以一般不使用LPAE技术；**
  * **3.设置CPU为SVC32模式，除非已经处于HYP模式，同时禁止中断（FIQ和IRQ）；**
  * **4.设置中断向量表地址为_start函数的地址，在map文件中可以看到，为0x87800000；**
  * **5.进行CPU初始化，调用函数cpu_init_cp15和cpu_init_crit分别初始化CP15和CRIT；**
  * **6.最后跳转到_main函数。**
  
   
  
  - #### 那么什么是SVC模式呢？？？
  
    首先要知道ARM中CPU的几种模式是什么。
  
    （1）用户模式（usr）：在此模式下程序不能够访问一些受操作系统保护的系统资源，应用程序也不能直接进行处理器模式的切换。
  
    （2）系统模式（sys）：与用户模式类似，但是具有可以直接切换到其他模式的特权。
  
    （3）快中断（fiq）：FIQ异常响应时进入此模式。
  
    （4）中断（irq）：IRQ异常响应时进入此模式。
  
    （5）管理（svc）：用于操作系统保护代码，当系统复位和软件中断响应时进入此模式。
  
    （6）中止（abt）：这个模式在ARM7TDMI中没有大用处。
  
    （7）未定义（und）：支持硬件协处理器的软件仿真，未定义异常响应时会进入此模式。
  
  - **首先，sys模式和usr模式相比，所用的寄存器组，都是一样的，但是增加了一些访问一些在usr模式下不能访问的资源。**
  
    **而svc模式本身就属于特权模式，本身就可以访问那些受控资源，而且，比sys模式还多了些自己模式下的影子寄存器，所以，相对sys模式来说，可以访问资源的能力相同，但是拥有更多的硬件资源。**
  
    **所以，从理论上来说，虽然可以设置为sys和svc模式的任一种，但是从uboot方面考虑，其要做的事情是初始化系统相关硬件资源，需要获取尽量多的权限，以方便操作硬件，初始化硬件。**
  
    **从uboot的目的是初始化硬件的角度来说，设置为svc模式，更有利于其工作。因此，此处将CPU设置为SVC模式。**
  
  * **其次，uboot作为一个bootloader来说，最终目的是为了启动Linux的kernel，在做好准备工作（即初始化硬件，准备好kernel和rootfs等）跳转到kernel之前，本身就要满足一些条件，其中一个条件，就是要求CPU处于SVC模式的。**



**到这里，上述代码所完成的任务为：**

* **初始化异常向量表**
* **设置CPU SVC模式，关中断**
* **配置CP15，设置异常向量入口**



紧接着下面两句跳转语句：

~~~
	/* the mask ROM code should have PLL and others stable */
#ifndef CONFIG_SKIP_LOWLEVEL_INIT
	bl	cpu_init_cp15
	bl	cpu_init_crit
#endif
	bl  _main

~~~

***



## （四）**cpu_init_cp15函数讲解**

**cpu_init_cp15函数，在文件arch/arm/cpu/armv7/start.S中定义，具体代码如下：**

~~~
/*************************************************************************
 *
 * cpu_init_cp15
 *				//配置CP15，关闭的cache、MMU、TLBs。icache视情况
 * Setup CP15 registers (cache, MMU, TLBs). The I-cache is turned on unless
 * CONFIG_SYS_ICACHE_OFF is defined.
 *
 *************************************************************************/
ENTRY(cpu_init_cp15)
	/*
	 * Invalidate L1 I/D
	 */
	mov	r0, #0			@ set up for MCR
	mcr	p15, 0, r0, c8, c7, 0	@ invalidate TLBs			//失效TLBs
	mcr	p15, 0, r0, c7, c5, 0	@ invalidate icache			//失效icache
	mcr	p15, 0, r0, c7, c5, 6	@ invalidate BP array		//失效分支预测
	mcr     p15, 0, r0, c7, c10, 4	@ DSB				//数据同步屏障
	mcr     p15, 0, r0, c7, c5, 4	@ ISB				//指令同步屏障

	/*
	 * disable MMU stuff and caches
	 */
	mrc	p15, 0, r0, c1, c0, 0			//将SCTLR的值赋给r0
	bic	r0, r0, #0x00002000	@ clear bits 13 (--V-)		//清零第13位，异常向量映射到0x00000000
	bic	r0, r0, #0x00000007	@ clear bits 2:0 (-CAM)		//失效dcache，失效对齐检查，禁用MMU
	orr	r0, r0, #0x00000002	@ set bit 1 (--A-) Align			//使能对齐检查模式
	orr	r0, r0, #0x00000800	@ set bit 11 (Z---) BTB				//使能分支预测
#ifdef CONFIG_SYS_ICACHE_OFF						//该宏被定义
	bic	r0, r0, #0x00001000	@ clear bit 12 (I) I-cache	
#else
	orr	r0, r0, #0x00001000	@ set bit 12 (I) I-cache		//使能icache
#endif
	mcr	p15, 0, r0, c1, c0, 0			//将r0的值赋给SCTLR
	mov	pc, lr			@ back to my caller					//子过程运行结束，跳转回去
ENDPROC(cpu_init_cp15)
~~~

**cpu_init_cp15函数是通过配置`CP15`协处理器相关寄存器来进行一些设置，主要设置就是失效 L1 I/D Cache，禁用MMU和缓存，失效icache、失效BP array。**

* **什么是icache**
  * 缓存（cache）是主存（DARAM）和CPU之间设置的一个高速的、容量相对较小的存储器，把正在执行的指令地址附近的一部分指令或数据从主存调入这个存储器，供CPU在一段时间内使用，以提高程序的运行速度。
  * 出于对简化设计的考虑，也为了提高系统的性能，设计采用了指令Cache（icache）、数据Cache（dcache）分开的方式。在icache中存储有微处理器需要的指令，在微处理器的取指阶段，通过程序计数器PC提供给icache的地址，微处理器可以获取需要的指令；而dcache则是作为一个数据的存储，并提供对于Load/Store指令所要操作地址的数据，它地址则来自于ALU运算的结果。参考文档：https://blog.csdn.net/bytxl/article/details/50275377

* **什么是 BP array**
  * BP，Branch Predictor，即分支预测。条件分支指令通常具有两路后续执行分支。即不采取跳转，顺序执行后面紧挨JMP的指令；以及采取跳转，到另一块程序内存去执行那里的指令。是否条件跳转，只有在该分支指令在指令流水线中通过了执行阶段才能确定。
  * 如果没有分支预测器，处理器将会等待分支指令通过了指令流水线的执行阶段，才把下一条指令送入流水线的第一个阶段——取指令阶段或者将后续流水线全部清空。这种技术叫做流水线停顿、流水线冒泡。

#### 需要注意到，为什么U-Boot需要失效 dcache、禁止MMU，却不失效icache？

* Cache是CPU内部的缓存，它的作用是将常用的数据和指令放在CPU内部。Cache是通过CP15管理的，刚上电的时候，CPU还不能管理Cache。上电的时候icache可关闭，也可不关闭，但dcache一定要关闭，否则可能导致刚开始的代码里面，去取数据的时候，从dcache里面取，而这时候DARAM中数据还没有过来，导致数据预取异常 。而在使用MMU之前要进行一系列的初始化，并且过程比较复杂，并且U-Boot暂时用不到所以要关闭它。



  

## （2）uboot的设备树可以合并到kernel？？？？

## （3）uboot bootargs如何传递给内核？？？？

## （4）kernel的dts里面的chosen有效还是uboot的bootargs有效？？？？

## （5）如果均有效如何合并的？？？？

## （6）如果定义两个重复的键 不同的值 谁的优先级高 谁覆盖谁？？？？

## （顺便也把存储设备分区（基本都是gpt分区）搞清楚）

## （7）fat分区 ext分区和gpt分区关系 谁是大哥？？？？

## （8）uboot的booti bootm bootfit命令启动的区别 内核的Image.itb zImage Image Image.gz镜像区别？？？？

## （9）initramfs的作用 如何生成 不用initramfs可不可以启动系统？？？？
