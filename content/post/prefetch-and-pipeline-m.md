---
title: "ARM Cortex-M 流水线与指令预取pipeline & prefetch"
date: 2020-11-12T17:31:33+08:00
draft: false
---

## 		ARM Cortex-M 流水线与指令预取pipeline & prefetch

[TOC]

最近的项目碰到了ECC的问题， 但问题的根源其实跟Arm-v7m架构的指令预取（instruction prefetch）有关。

​		项目调试的NXP i.MX8系列上的Cortex-M4F 核的TCM（Tightly-Coupled Memory， 具有与Core相同频率的RAM）有ECC功能，程序的代码入口放在外部的QSPI flash上, 但是有一部分时间敏感（需要快速执行）的代码被放到了内部TCM上的**QuickAccess**段。我们发现代码执行时偶尔会有”触碰“ QuickAccess段之外的TCM的情况（触发了ECC error, 我们只对用到的TCM进行了ECC clean, CPU读没有用到的TCM时会触发ECC error)。最后发现是因为如果CPU执行到QuickAccess段的末尾时，由于M4有PFU(prefetch unit, 下面会介绍)，在当前指令执行完成前会从内存中获取指令以填充流水线， 而指令的预取基于分支预测。

> **Branch prediction**
>
> Is where a processor chooses a future execution path to prefetch along (see Prefetching). For example, after a branch instruction, the processor can choose to **prefetch either the instruction following the branch or the instruction at the branch target**.
>

​		也就是说处理器可以选择预取当前分支接下来的指令或者跳转后的目标分支的指令， 这也就导致了我们的软件偶发ECC error。 假如预取当前分支接下来的指令，则会访问超出QuickAccess段的TCM，这时触发ECC， 系统宕机。 通常情况下，软件是不需要关心流水线pipeline与指令预取的，他们对于软件执行来说是透明的。

​		下面具体介绍一些相关知识， 毕竟我是做软件的，对CortexM核的硬件细节认知有限，难免有不准确的地方。如有纰漏欢迎指正。

#### 	背景知识：

##### 什么是ECC：

​		在ECC技术出现之前，内存中应用最多的另外一种错误检查技术，是奇偶校验位（Parity）技术。而带有“奇偶校验”的内存在每一字节（8位）外又额外增加了一位用来进行错误检测。

​		但奇偶校验位技术有个缺点，当内存查到某个数据位有错误时，由于不一定能确定错误在哪一个位，也就不一定能修正错误。所以带有奇偶校验的内存的主要功能仅仅是“发现错误”，并能纠正部分简单的错误。此外，奇偶校验技术是通过在原来数据位的基础上增加一个数据位来检查当前8位数据的正确性，但随着数据位的增加，用来检验的数据位也成倍增加，就是说当数据位为16位时它需要增加2位用于检查，当数据位为32位时则需增加4位，依此类推。特别是当数据量非常大时，数据出错的几率也就越大，对于只能纠正简单错误的奇偶检验的方法就显得力不从心了。正是基于这样一种情况，错误检查和纠正（Error Checking and Correcting）应运而生了。

ECC的英文全称是“ Error Checking and Correcting”（错误检查和纠正），从这个名称就可以看出它的主要功能就是“发现并纠正错误”。与奇偶校验技术一样，ECC纠错技术也需要额外的空间来储存校正码，但其占用的位数跟数据的长度并非成线性关系。具体来说，它是以8位数据、5位ECC码为基准，随后每增加一个8位数据只需另增加一位ECC码即可。

##### 指令流水线

​		Cortex-M0是冯诺依曼体系，3级流水线RISC架构，没有指令预取的。Cortex M3, M4, 采用三级流水线支持分支预测，哈佛结构，除了原system总线负责SRAM存取外，还新增两条ICode、DCode总线分别完成Flash上指令和数据存取。M7拥有双发射六级流水线并支持分支预测， 同M3,M4同属 ARM v7m RISC指令架构。

指令流水线是一种被用于处理器设计的技术，提升指令的吞吐量。经典的流水线由三个阶段组成 --- 取指、译码和执行，如图所示

![pipeline](/website/images/pipeline-3.png)


​		一般来说，ARM架构试图对编程者隐藏流水线影响。这意味着，编程者只能通过阅读处理器手册来决定流水线结构。然而，一些流水线工件仍然存在。例如，程序计数器寄存器(R15)在ARM状态时，指向超前于当前正执行指令两个指令的指令，这是最初ARM1处理器三阶段流水线的遗留物。

​		长流水线一个更近一步的缺点是有时来自内存的指令的串行执行会被打断。这可能会发生在一个分支指令执行的结果，或被一个异常事件(例如一个中断)。当此发生时，处理器无法决定下一条应该被获取指令的正确位置，直到分支被解决。在典型的代码中，很多分支指令是有条件的，作为循环或者if声明的结果。因此，分支是否会被执行不能再指令获取时决定。如果我们获取的指令紧跟着一个分支，并且分支被执行，流水线必须清空，来自分支目的地的指令新指令集必须从内存获取。ARM v7m 有分支预测逻辑，目的是减少分支惩罚的影响。**如果预测正确，分支不会清空流水线**。**如果预测错误，流水线必须清空，并且获取来自正确位置的指令来充填**。

##### 分支预测（Branch prediction）

​		一个条件跳转指令第一次被获取时，没有关于下一条指令地址的可依赖信息。旧的ARM处理器使用静态分支预测。这是最简单的分支预测方法，因为它不需要关于分支的先前信息。我们推测向后的分支会被采用，向前的分支不会。向后的分支具有一个目标地址，低于它自己的地址。我们因此可以看一个单一的操作位来决定分支方向。这个技术可以给与合理的预测精度，归因于代码中循环的普遍，循环代码总是包含后向分支，并且后向总比前向多很多。由于Cortex-M7系列处理器的流水线长度，我们通过使用更为复杂的给予更好预测精度的分支预测方法，可以获得更好的性能。

​		动态分支预测可以进一步减少平均分支惩罚，通过使用在前面执行时条件分支是否被采用的历史信息。Cortex-M7处理器中的分支目标地址缓存(Branch Target Address Cache ,BTAC)

ARM®v7-M Architecture Reference Manual里写道：

> An ARMv7-M implementation must choose how far ahead of the current point of execution it prefetches instructions. This can be either a fixed or a dynamically varying number of instructions. As well as choosing how many instructions to prefetch, an implementation can choose which possible future execution path to prefetch along.
> For example, after a branch instruction, it can prefetch either the instruction appearing in program order after the branch or the instruction at the branch target. This is known as branch prediction.

具体预取多少在Cortex-M4 Technical Reference Manual和Cortex-M7 Technical Reference Manual里分别有介绍。

- 对于Cortex-M4

**Prefetch Unit (PFU)** The PFU fetches instructions from the memory system that can supply one word each cycle. The PFU buffers up to three word fetches in its FIFO, which means that it can buffer up to three
32-bit Thumb instructions or six 16-bit Thumb instructions.

- 对于Cortex-M7

  The Prefetch Unit (PFU) provides:
  • 64-bit instruction fetch bandwidth.
  • 4x64-bit pre-fetch queue to decouple instruction pre-fetch from DPU pipeline operation.
  • A Branch Target Address Cache (BTAC) for single-cycle turn-around of branch predictor
  state and target address.
  • A static branch predictor when no BTAC is specified.
  • Forwarding of flags for early resolution of direct branches in the decoder and first
  execution stages of the processor pipeline.

  可见对于M4来说预取长度是6个16-Bit Thumb 指令，或3个32 Bit指令。对于M7是8个Byte。



#### 总结与提醒

​			根据上述分析可知，TCM的越界访问来自于指令预取。那么当程序text段全部放在TCM里时，我们也没有把TCM全部占满，但并没有碰到CPU越界访问的情况（没有预取到text段以外的内存）， 这是为什么呢？其实具体的应用中很少会有code将TCM占满的情况， 一般会有Read only data或者一些空的（reserved）中断服函数之类的。 实际测试发现， armgcc 会在binary结尾加0， KEIL会将readonly data放在结尾， 当然最终的binary也会跟linker有关。总之在实际的应用中要考虑到prefetch的情况。

*原创文章：转载请注明出处https://github.com/willisyi*