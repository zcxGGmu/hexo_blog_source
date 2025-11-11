---
title: cpu虚拟化
date: 2025-11-11 10:28:38
tags: 虚拟化
description: Intel VT-x虚拟化分析
---



[TOC]

# 1 概述

本章将深入介绍CPU虚拟化部分。早期由于缺乏相应硬件的支持，只能采用陷入-模拟、扫描与修补、二进制翻译等软件模拟的方式解决“虚拟化漏洞”问题，效率较低。而随着Intel VT-x等虚拟化硬件技术的出现，硬件辅助CPU虚拟化技术逐渐成为主流。本文将按照如下顺序进行介绍：

> 1. 简要概述CPU虚拟化
> 2. 介绍Intel VT-x提供的CPU虚拟化硬件支持
> 3. QEMU/KVM CPU虚拟化实现
> 4. QEMU/KVM 中断虚拟化实现
> 5. 介绍新型“多虚一”开源项目GiantVM中CPU虚拟化和中断虚拟化的实现。

首先回顾物理环境中CPU的主要功能。作为运算单元，CPU从主存中取出指令并执行，在此过程中CPU需要从寄存器或主存中获取操作数，并将结果写回寄存器或主存。此外，CPU还需要响应一些发生的系统事件，这些系统事件可能是由指令执行触发，如除零错误、段错误等；也有可能是由外部事件触发，如网卡收到了一个网络包、磁盘数据传输完成等。在这些事件中，最受关注的就是中断和异常事件。简而言之，**CPU应当能高效正确地执行所有指令并响应一些发生的系统事件，这也是虚拟环境中vCPU应当完成的工作。**但在虚拟环境中，要实现上述功能却面临一些挑战。

## 1.1 敏感非特权指令的处理

### 敏感指令、特权指令与敏感非特权指令

在现代计算机架构中，CPU通常拥有两个或两个以上的特权级，其中操作系统运行在最高特权级，其余程序则运行在较低的特权级。而一些指令必须运行在最高特权级中，若在非最高特权级中执行这些指令将会触发特权级切换，陷入最高特权级中，这类指令称为**特权指令。**在虚拟化环境中，还有另一类指令称为**敏感指令，**即操作敏感物理资源的指令，如I/O指令、页表基地址切换指令等。

虚拟化系统的三个基本要求：资源控制、等价与高效。资源控制要求Hypervisor能够控制所有的物理资源，虚拟机对敏感物理资源（部分寄存器、I/O设备等）的访问都应在Hypervisor的监控下进行。这意味着在虚拟化环境中，Hypervisor应当替代操作系统运行在最高特权级，管理物理资源并向上提供服务，当虚拟机执行敏感指令时必须陷入Hypervisor（通常称为虚拟机下陷）中进行模拟，这种敏感指令的处理方式称为“陷入-模拟”方式。

“陷入-模拟” 方式要求所有的敏感指令都能触发特权级切换，从而能够陷入Hypervisor中处理，通常将所有敏感指令都是特权指令的架构称为可虚拟化架构，反之存在敏感非特权指令的架构称为不可虚拟化架构。遗憾的是，大多数计算机架构在设计之初并未将虚拟化技术考虑在内。

> 以早期的x86架构为例，其SGDT（Store Global Descriptor Table，存储全局描述符表）指令将GDTR（Global Descriptor Table Register，全局描述符表寄存器）的值存储到某个内存区域中，其中全局描述符表用于寻址，属于敏感物理资源，但是在x86架构中，SGDT指令并非特权指令，无法触发特权级切换。

在x86架构中类似SGDT的敏感非特权指令多达17条，Intel将这些指令称为“虚拟化漏洞”。**在不可虚拟化架构下，为了使Hypervisor截获并模拟上述敏感非特权指令，一系列软件方案应运而生，**下面介绍这些软件解决方案。

### 敏感非特权指令的软件解决方案

敏感非特权指令的软件解决方案主要包括**解释执行、二进制翻译、扫描与修补以及半虚拟化技术。**

* **解释执行技术。**解释执行技术采用软件模拟的方式逐条模拟虚拟机指令的执行。解释器将程序二进制解码后调用指令相应的模拟函数，对寄存器的更改则变为修改保存在内存中的虚拟寄存器的值。
* **二进制翻译技术。**区别于解释执行技术不加区分地翻译所有指令，二进制翻译技术则以基本块为单位，将虚拟机指令批量翻译后保存在代码缓存中，基本块中的敏感指令会被替换为一系列其他指令。
* **扫描与修补技术。**扫描与修补技术是在执行每段代码前对其进行扫描，找到其中的敏感指令，将其替换为特权指令，当CPU执行翻译后的代码时，遇到替换后的特权指令便会陷入Hypervisor中进行模拟，执行对应的补丁代码。
* **半虚拟化技术。**上述三种方式都是通过扫描二进制代码找到其中敏感指令，半虚拟化则允许虚拟机在执行敏感指令时通过超调用主动陷入Hypervisor中，避免了扫描程序二进制代码引入的开销。

上述解决方案的优缺点如下表所示：

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311221415290.png" alt="image-20231122141503318" style="zoom:33%;" />

这几种方案通过软件模拟解决了敏感非特权指令问题，但却产生了巨大的软件开销。**敏感非特权指令究其本质是硬件架构缺乏对于敏感指令下陷的支持，**近年来各主流架构都从架构层面弥补了“虚拟化漏洞”，解决了敏感非特权指令的陷入-模拟问题，下面简要介绍这些硬件解决方案。

### 敏感非特权指令的硬件解决方案

前面提到，敏感非特权指令存在的根本原因是硬件架构缺乏对敏感指令下陷的支持。因此最简单的一种办法是更改现有的硬件架构，将所有的敏感指令都变为特权指令，使之能触发特权级切换，但是这将改变现有指令的语义，现有系统也必须更改来适配上述改动。

另一种办法是**引入虚拟化模式。**未开启虚拟化模式时，操作系统与应用程序运行在原有的特权级，一切行为如常，兼容原有系统；开启虚拟化模式后，Hypervisor运行在最高特权级，虚拟机操作系统与应用程序运行在较低特权级，虚拟机执行敏感指令将会触发特权级切换陷入Hypervisor中进行模拟。虚拟化模式与非虚拟化模式架构如图2-1所示。

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311221417410.png" alt="image-20231122141655482" style="zoom: 50%;" />

> 注：①陷入；②恢复；③开启虚拟化模式；④关闭虚拟化模式。

非虚拟化模式通常只需要两个特权级，而虚拟化模式需要至少三个特权级用于区分虚拟机应用程序、虚拟机操作系统与Hypervisor的控制权限，此外还需要引入相应的指令开启和关闭虚拟化模式。虚拟化模式对现有软件影响较小，Hypervisor能够作为独立的抽象层运行于系统中，因此当下大多数虚拟化硬件都采用该方式：

* Intel VT-x为CPU引入了根模式与非根模式，分别供Hypervisor和虚拟机运行；
* ARM v8在原有EL0与EL1的基础上引入了新的异常级EL2供Hypervisor运行；
* RISC-V Hypervisor Extension则添加了两个额外的特权级，即VS/VU供虚拟机操作系统和虚拟机应用程序运行，原本的S特权级变为HS，Hypervisor运行在该特权级下。

虚拟化模式的引入解决了敏感非特权指令的陷入以及系统兼容性问题，但是**特权级的增加也带来了上下文切换问题。**下面介绍虚拟化环境中的上下文切换。

## 1.2 虚拟机上下文切换

在操作系统的进程上下文切换中，操作系统与用户态程序运行在不同的特权级中，当用户态程序发起系统调用时，需要将部分程序状态保存在内存中，待系统调用完成后再从内存中恢复程序状态。

而在虚拟化环境下，当虚拟机执行敏感指令时，需要陷入Hypervisor进行处理，Hypervisor与虚拟机同样运行在不同的特权级中，因此硬件应当提供一种机制在发生虚拟机下陷时保存虚拟机的上下文。等到敏感指令模拟完成后，当虚拟机恢复运行时重新加载虚拟机上下文。

> 此处，“虚拟机上下文”表述可能有些不准确，更准确的说法应当是“vCPU上下文”。

一个虚拟机中可能包含多个vCPU，虚拟机中指令执行单元是vCPU，Hypervisor调度虚拟机运行的基本单位也是vCPU。当vCPU A执行敏感指令陷入Hypervisor时，vCPU B将会继续运行。在大部分Hypervisor中，vCPU对应一个线程，通过分时复用的方式共享物理CPU。

以vCPU切换为例说明上下文切换的流程，vCPU切换流程如下图所示，其中**实线表示控制流，虚线表示数据流。**

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311221419129.png" alt="image-20231122141910313" style="zoom:50%;" />

> 注：①保存vCPU寄存器；②加载Hypervisor寄存器；③保存Hypervisor寄存器；④加载vCPU寄存器；⑤指令执行顺序。

可以看到：当vCPU 1时间片用尽时，Hypervisor将会中断vCPU 1执行，vCPU 1陷入Hypervisor中（见图中标号I）。在此过程中，硬件将vCPU 1的寄存器状态保存至固定区域（见图中标号①），并从中加载Hypervisor的寄存器状态（见图中标号②）。Hypervisor进行vCPU调度，选择下一个运行的vCPU，保存Hypervisor的寄存器状态（见图中标号③），并加载选定的vCPU 2的寄存器状态（见图中标号④），而后恢复vCPU 2运行（见图中标号II）。

上述固定区域与系统架构实现密切相关，以Intel VT-x/ARMv8为例：

* 在Intel VT-x中，虚拟机与Hypervisor寄存器状态保存在VMCS（Virtual Machine Control Structure，虚拟机控制结构）中，VMCS是内存中的一块固定区域，通过 `VMREAD/VMWRITE` 指令进行读写；
* 而ARM v8则为EL1和EL2提供了两套系统寄存器，因此单纯发生虚拟机下陷时，无须保存寄存器状态；但是虚拟机下陷后，若要运行其他vCPU，则需要将上一个vCPU状态保存至内存中。

后续章节将以Intel VT-x为例介绍上下文切换过程中VMCS的具体用法。

## 1.3 中断虚拟化

前面提到，vCPU不仅要能高效地执行所有指令，还要能正确处理系统中出现的中断和异常事件。大部分异常（如除零错误、非法指令等）无须虚拟化，直接交给虚拟机操作系统处理即可。而对于部分需要虚拟化的异常，则需要陷入Hypervisor中进行相应的处理，没有固定的解决方案。

> 内存虚拟化文档中将会介绍Hypervisor如何处理虚拟机缺页异常。本节主要关注中断虚拟化的相关内容。

中断是外部设备请求操作系统服务的一种方式，通常由外部设备发起，经由中断控制器发送给CPU。以磁盘为例，在物理环境下，操作系统发起一个读磁盘请求，磁盘将操作系统请求的数据放置在指定位置后给中断控制器发送一个中断请求，中断控制器接收后设置好内部相应寄存器，CPU每次执行指令前都会检查中断控制器中是否存在未处理的中断，若有则调用相应的ISR（Interrupt Service Routine，中断服务例程）进行处理。

**而在虚拟环境下，可能存在多个虚拟机同时运行的情况，它们通过虚拟设备共用物理磁盘，此时若磁盘产生一个物理中断，该中断应该交给哪一个虚拟机的操作系统处理呢？**即如何将该中断注入发起读磁盘操作的虚拟机中，使该虚拟机操作系统执行相应的ISR。

在虚拟化环境下，设备与中断控制器均由Hypervisor模拟，相应寄存器的状态对应于内存中某些数据结构的值。当虚拟机执行I/O指令时，会陷入Hypervisor中进行处理，Hypervisor调用相应的设备驱动完成I/O操作。在上述过程中，Hypervisor不仅知道发起I/O操作的虚拟机的详细信息，还能通过设置虚拟设备和虚拟中断控制器的寄存器状态从而完成中断注入。因此一个理想的方案是将该物理中断交给Hypervisor处理，再由Hypervisor设置虚拟中断控制器，注入一个虚拟中断到虚拟机中。

仍以读磁盘为例，中断虚拟化流程如下图所示：

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311221422001.png" alt="image-20231122142204573" style="zoom:50%;" />

> 注：①向虚拟磁盘发起读请求；②虚拟机下陷；③Hypervisor向物理磁盘发起读请求；④向中断控制器提交中断；⑤执行中断服务例程；⑥Hypervisor设置虚拟磁盘与虚拟中断控制器；⑦恢复虚拟机执行；⑧虚拟机执行中断服务例程。

虚拟机发起一个读磁盘请求（见图中标号①）触发虚拟机下陷进入Hypervisor进行处理（见图中标号②），Hypervisor向物理磁盘发起读磁盘请求（见图中标号③），物理磁盘完成数据读请求后产生一个中断并递交给物理中断控制器（见图中标号④），Hypervisor执行相应的中断服务例程完成数据读取（见图中标号⑤），并设置虚拟磁盘和虚拟中断控制器相应寄存器的状态（见图中标号⑥），而后Hypervisor恢复虚拟机运行（见图中标号⑦），虚拟机发现有待处理的中断，调用相应的中断服务例程（见图中标号⑧）。

上述流程对硬件有如下要求：

* **物理中断将会触发虚拟机下陷进入Hypervisor中处理；**
* **虚拟机恢复运行时要先检查虚拟中断控制器是否有待处理中断。**

后续章节将以Intel VT-x为例说明虚拟化硬件如何完成这两点要求。

---

**通过虚拟机下陷将物理中断转换为虚拟中断解决了多虚拟机系统中物理中断的路由问题，**但是对于直通设备却很不友好。直通设备是指通过硬件支持将一个物理设备直通给某个虚拟机使用，该设备由这个虚拟机独占，故该设备产生的中断也应当由这个虚拟机处理，无须经过Hypervisor路由。

* 为了解决这个问题，ELI（Exitless Interrupt，不退出中断）引入了Shadow IDT（Shadow Interrupt Descriptor Table，影子中断描述符表），区分直通设备产生的中断与其他物理中断，直通设备产生的中断直接递交给虚拟机处理，无须Hypervisor介入。
* DID（Direct Interrupt Delivery，直接中断交付）则进一步通过将虚拟设备产生的中断转换为物理IPI（Inter-Processor Interrupt，处理器间中断）直接递交给虚拟机进行处理。

相较于上述软件方案，硬件厂商提供了新的硬件机制，**直接将中断注入正在运行的虚拟机且不会引发虚拟机下陷，**如Intel公司的发布-中断(Posted-Interrupt)机制和ARM公司的ITS（Interrupt Translation Service，中断翻译服务）机制。Directvisor便使用Posted-Interrupt机制将直通设备产生的中断注入虚拟机中。

“多虚一”环境下的中断虚拟化则更为复杂，节点A上I/O设备产生的中断可能需要注入节点B上运行的vCPU中。为了解决上述问题，GiantVM在每一个物理节点上都创建一个虚拟中断控制器，并选定一个节点作为主节点，其他节点上的虚拟中断控制器接收到中断信号时，会将该信号转发给主节点上的虚拟中断控制器，设置相应寄存器的值，而后由主节点虚拟中断控制器决定将该中断注入哪个vCPU中。

# 2 Intel VT-x硬件辅助虚拟化

前文介绍了CPU虚拟化面临的一些挑战和可能的软硬件解决方案。本节将以Intel VT-x为例介绍上述问题在x86架构下是如何解决的。自2005年首次公布硬件辅助虚拟化技术以来，Intel公司陆续推出了针对处理器虚拟化的Intel VT-x技术、针对I/O虚拟化的Intel VT-d技术和针对网络虚拟化的Intel VT-c技术，它们统称为Intel VT（Intel Virtualization Technology，英特尔虚拟化技术）。

本节将着眼于Intel VT-x中与CPU虚拟化相关的部分，Intel EPT（Extended Page Table，扩展页表）内存虚拟化技术和Intel VT-d I/O虚拟化技术将在其他文档中介绍。

## 2.1 VMX操作模式

为了弥补 “虚拟化漏洞”，Intel VT-x引入了VMX（Virtual Machine eXtension，虚拟机扩展）操作模式，CPU可以通过 `VMXON/VMXOFF` 指令打开或关闭VMX操作模式。VMX操作模式类似于前述虚拟化模式，包含根模式与非根模式，**其中Hypervisor运行在根模式，虚拟机则运行在非根模式，**VMX模式下敏感指令处理示意图如下图所示：

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311221539035.png" alt="image-20231122153856154" style="zoom: 50%;" />

CPU从根模式切换为非根模式称为VM-Entry，从非根模式切换为根模式则称为VM-Exit。值得注意的是，根模式与非根模式都有各自的特权级(Ring0~Ring3)，虚拟机操作系统和应用程序分别运行在非根模式的Ring0和Ring3特权级中，而Hypervisor通常运行在根模式的Ring0特权级，解决了三者的特权级的划分问题。

Intel VT-x改变了非根模式下敏感非特权指令的语义，使它们能够触发VM-Exit，而根模式指令的语义保持不变。处于非根模式的虚拟机执行敏感指令将会触发VM-Exit，陷入Hypervisor进行处理。Hypervisor读取VM-Exit相关信息，造成VM-Exit的原因（如I/O指令触发、外部中断触发等）并进行相应处理。处理完成后，Hypervisor调用 `VMLAUNCH/VMRESUME` 指令从根模式切换到非根模式，恢复虚拟机运行。

## 2.2 VMCS

为了解决1.2节提到的虚拟机上下文切换问题，Intel VT-x引入了VMCS。

* VMCS是内存中的一块区域，用于在 `VM-Entry` 和 `VM-Exit` 过程中保存和加载Hypervisor和虚拟机的寄存器状态；
* 此外，VMCS还包含一些控制域，用于控制CPU的行为。

本节将简要介绍VMCS的组成与使用方式。

### VMCS操作指令

在多处理器虚拟机中，VMCS与vCPU一一对应，每个vCPU都拥有一个VMCS。**当vCPU被调度到物理CPU上运行时，首先要将其VMCS与物理CPU绑定，物理CPU才能在 `VM-Entry/VM-Exit` 过程中将vCPU的寄存器状态保存到VMCS中。**Intel VT-x提供了两条指令，分别用于**绑定VMCS和解除VMCS绑定：**

* `VMPTRLD <VMCS地址>`：将指定VMCS与当前CPU绑定。
* `VMCLEAR <VMCS地址>`：同样以VMCS地址为操作数，将VMCS与当前CPU解除绑定，该指令确保CPU缓存中的VMCS数据被写入内存中。

在发生vCPU迁移时，需要先在原物理CPU上执行 `VMCLEAR` 指令，而后在目的物理CPU上执行 `VMPTRLD` 指令。

---

此外，VMCS虽然是内存区域，但是**Intel软件开发手册指出通过读写内存的方式读写VMCS数据是不可靠的，**因为VMCS数据域格式与架构实现是相关的，而且部分VMCS数据可能位于CPU缓存中，尚未同步到内存中。Intel VT-x提供 `VMREAD` 与 `VMWRITE` 指令用于**读写VMCS数据域，**指令格式如下：

* `VMREAD <索引>` ：读取索引指定的VMCS数据域。
* `VMWRITE <索引> <数据>`：将数据写入索引指定的VMCS数据域。

### VMCS结构

VMCS区域大小不固定，最多占用4KB，VMCS的具体大小可以通过查询MSR（Model Specific Register，特殊模块寄存器）`IA32_VMX_BASIC[32∶44]` 得知。VMCS结构如下图所示：

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311221542780.png" alt="image-20231122154206874" style="zoom:50%;" />

**VMCS版本标识符** (VMCS revision identifier) 指明了VMCS数据域 (VMCS data) 的格式，**VMX中止指示符** (VMX-abort indicator) 则记录了VMX的中止原因。**VMCS数据域**则包括以下六部分：

1. **客户机状态域(Guest-state area)。**保存虚拟机寄存器状态的区域，主要包括CR0和CR3等控制寄存器、栈指针寄存器RSP、PC寄存器RIP等重要寄存器的状态。

2. **宿主机状态域(Host-state area)。**与客户机状态域类似，是保存Hypervisor寄存器状态的区域，它包含的寄存器与客户机状态域大致相同。

3. **VM-Execution控制域(VM-Execution control fields)。**控制客户机在非根模式下运行时的行为，如哪些指令会触发VM-Exit、外部中断是否引发VM-Exit等。

   > 在1.3节提到，将物理中断转化为虚拟中断注入虚拟机要求物理中断能够触发虚拟机下陷，这一特性由VM-Execution控制域中的外部中断退出 (External-Interrupt Exiting) 字段控制，当该位置为1时，外部中断将会触发VM-Exit，若为0则不会触发VM-Exit，直接由Guest处理。`ELI` 和 `DID` 都利用了该特性直接递交中断。

4. **VM-Exit控制域(VM-Exit control fields)。**控制VM-Exit过程中的某些行为，如VM-Exit过程中需要加载哪些MSR。

5. **VM-Entry控制域(VM-Entry control fields)。**控制VM-Entry过程中的某些行为，如VM-Entry后CPU的运行模式。

6. **VM-Exit信息域(VM-Exit information fields)。**用以保存VM-Exit的基本原因及其他详细信息。

从以上描述不难发现，VMCS对硬件辅助虚拟化具有极其重要的意义，几乎影响了虚拟化的方方面面。下图主要展示了VMCS在VM-Entry和VM-Exit过程中的作用，后续章节还会涉及部分VMCS域的具体功能。

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311221544962.png" alt="image-20231122154353415" style="zoom:33%;" />

> 注：①保存客户机状态；②加载宿主机状态；③Hypervisor获取退出原因；④保存宿主机状态；⑤加载客户机状态；⑥指令执行顺序。

## 2.3 PIC/APIC

在多虚拟机环境下，可以通过虚拟机下陷将物理中断转换为虚拟中断，解决物理中断的路由问题。在此过程中，Hypervisor通过虚拟中断控制器模拟真实环境下中断的分发和传送，完成虚拟中断的注入，因此在介绍中断虚拟化之前有必要先了解物理中断控制器及其工作方式。

本节将简要介绍Intel x86架构下的两种中断控制器——PIC（Programmable Interrupt Controller，可编程中断控制器）和APIC（Advanced Programmable Interrupt Controller，高级可编程中断控制器）及它们各自的中断处理流程。

### PIC

PIC，即Intel 8259A芯片，是单处理器 (Uni-processor) 时代广泛使用的中断控制器。它主要包含8个中断引脚 `IR0~IR7`，用于连接外部设备以及三个内部寄存器：`IMR`（Interrupt Mask Register，中断屏蔽寄存器）、`IRR`（Interrupt Request Register，中断请求寄存器）和 `ISR`（Interrupt Service Register，中断服务寄存器）。

* `IMR`：共8位，对应于IR0~IR7，置1表示相应引脚中断被屏蔽；
* `IRR`：共8位，对应于IR0~IR7，置1表示收到相应引脚的中断信号；
* `ISR`：共8位，对应于IR0~IR7，置1表示收到相应引脚的中断信号正在被CPU处理。

其中 `IR0~IR7` 用于连接外部设备，连接到不同的中断引脚对应的中断优先级不同，其中 `IR0` 优先级最高，`IR7` 优先级最低。PIC逻辑如下：

1. 每当外设需要发送中断时，便会拉高相连中断引脚的电平；
2. 若相关中断未被屏蔽，就设置 `IRR` 中相应位并拉高 `INT` 引脚电平通知CPU有中断到达；
3. CPU给 `INTA` 引脚发送一个脉冲确认收到中断，8259A芯片收到上述 `INTA` 脉冲信号后，将 `IRR` 最高优先级位清零并将 `ISR` 对应位置1；
4. 而后CPU发送第二次脉冲，8259A芯片收到后将最高优先级的中断向量号发送给到数据线上，CPU接收中断向量号并执行相应的中断服务例程；
5. 中断处理完成后，CPU向8259A的命令寄存器写入指定值并发送一个 `EOI`（End of Interrupt，中断结束）信号；
6. 8259A芯片收到 `EOI` 后，将 `ISR` 中的最高优先级位清零。

根据上述描述，8259A芯片最多支持8个中断源，而为了支持更多的外设，通常将若干个8259A芯片级联，最多可以支持64个中断。

### APIC

APIC是20世纪90年代Intel公司为了应对多处理器 (Multi-Processor) 架构提出的一整套中断处理方案，用于取代老旧的8259A PIC。APIC适用于多处理器机器，每个CPU拥有一个LAPIC（Local APIC，本地APIC），整个机器拥有一个或多个IOAPIC（I/O APIC，输入/输出APIC），**设备的中断信号先经由IOAPIC汇总，再分发给一个或多个CPU的LAPIC。**

LAPIC中也存在 `IRR` 和 `ISR` 以及 `EOI` 寄存器，`IRR` 和 `ISR` 的功能与PIC中相应寄存器类似，大小为256位，对应x86平台下的256个中断向量号。在APIC中断架构下，LAPIC的中断来源可分为以下3类：

* **本地中断。**本地中断包括 `LINT0` 和 `LINT1` 引脚接收的中断、APIC Timer产生的中断、性能计数器产生的中断、温度传感器产生的中断以及APIC内部错误引发的中断。

* **通过IOAPIC接收的外部中断。**当I/O设备通过IOAPIC中断引脚产生中断信号时，IOAPIC从内部的PRT（Programmable Redirection Table，可编程重定向表）中找到相应的RTE（Redirection Table Entry，重定向表项）。PRT共有24条PTE，与IOAPIC的24个中断引脚对应，每条PTE长度为64位。IOAPIC根据PTE中存储的信息（如触发方式、中断向量号、目的处理器等）格式化出一条中断消息发送到系统总线上。

  在PIC中，中断优先级由所连接的8259A芯片引脚决定，而在APIC中，中断优先级由中断向量号（也称为vector）决定，范围为0~255，`ISR` 和 `IRR` 每位对应一个中断向量号，置1表示收到该中断向量号的中断。中断向量号越大，中断优先级越高。`IRR` 和 `ISR` 的最大中断向量号记为 `IRRV` 和 `ISRV`。

* **处理器间中断(IPI)。**CPU可以通过写入APIC的 `ICR`（Interrupt Control Register，中断控制寄存器）发送一条IPI消息到系统总线上，从而发送给CPU。

---

LAPIC收到中断消息后，确认自己是否是中断消息的目标，如果是，则对该中断消息进一步处理，这一步称为中断路由(Interrupt Routing)。**LAPIC接收中断消息后，处理过程如下：**

1. 如果收到的中断是NMI、SMI、INIT、ExtINT或SIPI，则直接交给CPU处理，否则设置 `IRR` 寄存器中适当的位，将中断加入等待队列，这一步称为**中断接受(Interrupt Acceptance)。**
2. 对于在 `IRR` 中阻塞的中断，APIC取出其中优先级最高的中断（要求IRRV[7∶4]>PPR[7∶4]）交给CPU处理，并根据中断向量号设置 `ISR` 寄存器中相应的位。这一步称为**中断确认(Interrupt Acknowledgement)。**
3. CPU通过中断向量号索引 `IDT`（Interrupt Descriptor Table，中断描述符表）执行相应的中断处理例程，这一步称为**中断交付(Interrupt Delivery)。**
4. 中断处理例程执行完毕时，应写入 `EOI` 寄存器，使得APIC从 `ISR` 队列中删除中断对应的项，结束该中断的处理（NMI、SMI、INIT、ExtINT及SIPI不需要写入EOI）。

### MSI

MSI（Message Signaled Interrupt，消息告知中断）是PCI总线发展出的新型中断传送方式，它允许设备直接发送中断到LAPIC而无须经过IOAPIC。**MSI本质上就是在中断发生时，不通过带外(out-band)的中断信号线，而是通过带内(in-band)的PCI写入事务来通知LAPIC中断的发生。**

从原理上来说，MSI产生的事务与一般的DMA事务并无本质区别，需要依赖特定平台的特殊机制从总线事务中区分出MSI并赋予其中断的语义。

> 在x86平台上，是由Host Bridge/Root Complex负责这一职责，将MSI事务翻译成系统上的中断消息，但凡目标地址落在 `[0xfee00000，0xfeefffff]` 的写入事务都会被视为MSI中断请求并翻译成中断消息。

一个MSI事务由地址和数据构成，每个设备可以配置其发生中断时产生MSI事务的地址和数据，并且可以在不同事件发生时产生不同的MSI事务（即不同的地址-数据对）。MSI中断消息到达LAPIC后的处理流程与上述过程一致。

### IRQ & GSI & Vector & Pin

在正式介绍中断虚拟化之前，还需要介绍几个重要的概念：`IRQ`（Interrupt Request，中断请求）、`GSI`（Global System Interrupt，全局系统中断号）、`Vector`（中断向量号）和 `Pin`（中断引脚号）。这几个概念在后续章节将会反复提及，故在此进行区分。

* **IRQ：**IRQ是PIC时代的产物，但是如今可能还会见到IRQ线或者IRQ号等各种说法。在PIC中断架构下，通常将两块8259A芯片级联，支持16个中断，由于ISA设备通常连接到固定的8259A中断引脚，因此设备的IRQ号通常是指它所连接的8259A引脚号。

  如前所述，8259A有8个中断引脚( `IR0~IR7` )，那么连接到这些**主8259A**芯片 `IR0~IR7` 引脚的设备中断号分别对应 `IRQ0~IRQ7`，连接到**从8259A**芯片 `IR0~IR7` 引脚的设备对应的中断号为 `IRQ8~IRQ15`。而IRQ线可以理解为设备与这些引脚的连线，命名方式通常也为 `IRQx`，它们通常与中断控制器引脚一一对应，故使用IRQ时应当注意相应的情境。

* **GSI：**GSI是ACPI（Advanced Configuration and Power Interface，高级配置和电源管理接口）引入的概念，它**为系统中每个中断控制器的输入引脚指定了一个全局统一的编号。**例如系统中有多个IOAPIC，每个IOAPIC都会被BIOS分配一个基础GSI (GSI Base)，每个IOAPIC中断引脚对应的GSI为基础GSI+Pin。

  比如IOAPIC 0的基础GSI为0，有24个引脚，则它们分别对应 `GSI0~GSI23`。在APIC系统中，IRQ和GSI通常会被混用，15号以上的IRQ号与GSI相等；而15号以下的IRQ号与ISA设备高度耦合，只有当相应的ISA设备按照对应的IRQ号连接到IOAPIC 0的1~15引脚时，IRQ才和GSI相等，这种情况称为一致性映射。而若IRQ与GSI引脚不一一对应，ACPI将会维护一个ISO（Interrupt Source Override，中断源覆盖）结构描述IRQ与GSI的映射。如PIT（Programmable Interrupt Timer，可编程中断时钟）接PIC的IR0引脚，因此其IRQ为0；但当接IOAPIC时，它通常接在2号中断引脚，所以其GSI为2。而在QEMU/KVM中，GSI和IRQ完全等价，但是不符合前述基础GSI+Pin的映射关系。

* **Vector：**中断向量号是操作系统中的概念，是中断在 `IDT` 中的索引。每个GSI都对应一个Vector，它们的映射关系由操作系统决定。x86中通常包含256个中断向量号，0~31号中断向量号是x86预定义的，32~255号则由软件定义。

* **Pin：**中断控制器的中断引脚号，对于8259A而言，其Pin取值为 `0~7`；对于IOAPIC，其Pin取值为 `0~23`。

根据以上描述，IRQ、GSI和Vector都可以唯一标识系统中的中断来源，**IRQ和GSI的映射关系以及GSI和Pin的映射关系由ACPI设置，IRQ和Vector的映射关系由操作系统设置。**

## 2.4 Intel VT-x中断虚拟化

中断虚拟化的一般思路中，最为关键的两点便是：

* **中断设备的模拟（外部设备和中断控制器的模拟）**
* **中断处理流程的模拟**

这也是早期Intel VT-x中断虚拟化采用的方法，后来Intel公司推出了 `APICv`（APIC Virtualization，APIC虚拟化）技术对上述两个功能进行了优化。本节将分别介绍**传统中断虚拟化和APICv支持的中断虚拟化。**

### 传统中断虚拟化

在传统中断虚拟化中，Hypervisor会创建虚拟 `IOAPIC` 和虚拟 `LAPIC`，以下统称为虚拟APIC。它们通常表现为存储在内存中的数据结构，而其中的寄存器则作为结构体中的若干域 `fields`。

默认情况下，虚拟机通过MMIO（Memory-Mapped I/O，内存映射I/O）的方式访问虚拟APIC，但是由于底层并没有相应的物理硬件供其访问，故Hypervisor需要截获这些MMIO请求进而访问存储在内存中的虚拟寄存器的值。

> Hypervisor通常会将虚拟机中APIC内存映射区域相应的EPT项状态设为不存在，这样虚拟机访问这一段地址空间时，便会触发缺页错误从而触发VM-Exit陷入Hypervisor中进行模拟。

此外，**虚拟APIC仍需要接收和传递中断。**在物理环境下，外部设备产生的中断会通过连接线传输至APIC进而发送到CPU，在此过程中，硬件会设置相应寄存器的值。而在虚拟环境中，这些连接线则会被替换为一系列的函数调用，并在调用过程中设置内存中相应虚拟寄存器的值。

APIC中断处理流程包括**中断产生、中断路由、中断接受、中断确认和中断交付。**其中，中断产生到中断确认部分都是通过设置特定APIC寄存器完成，对于虚拟APIC而言，可以通过设置内存中相应的虚拟寄存器实现同样的功能。而对于中断交付而言，**需要硬件提供某种机制，使得虚拟机恢复运行时发现待处理的虚拟中断从而执行相应的ISR。**在Intel VT-x中，这是通过VMCS VM-Entry控制域中的VM-Entry中断信息字段（32位）实现的，该字段保存了待注入事件的类型（中断或异常等）和向量号等信息，其格式如下图所示：

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311221551030.png" alt="image-20231122155113238" style="zoom:50%;" />

每次触发VM-Entry时，CPU会检查该域，发现是否有待处理的事件并用向量号索引执行相应的处理函数。

当Hypervisor需要向正在运行的vCPU注入中断时，需要给vCPU发送一个信号，使其触发VM-Exit，从而在VM-Entry时注入中断。如果vCPU正处于屏蔽外部中断的状态，如vCPU的 `RFLAGS.IF=0`，将不允许在VM-Entry时进行中断注入。此时可以将VM-Execution控制域中的中断窗口退出(Interrupt-Window Exiting)字段置为1，这样一旦vCPU进入能够接收中断的状态，便会产生一个VM-Exit，Hypervisor就可以注入刚才无法注入的中断，并将中断窗口退出字段置为0。

### APICv支持的中断虚拟化

APICv是Intel公司针对APIC虚拟化提出的优化技术，它主要包括两方面内容：**虚拟APIC访问优化和虚拟中断递交。**

#### 虚拟APIC访问

前面提到，当虚拟机通过MMIO的方式访问APIC寄存器时，需要VM-Exit陷入Hypervisor中设置内存中相应虚拟寄存器的值，这将会触发大量的VM-Exit，严重影响虚拟机的性能。于是APICv引入虚拟APIC页 (Virtual APIC Page) 的概念，它相当于一个影子APIC，虚拟机对APIC的部分甚至全部访问都可以被硬件重定向为对虚拟APIC页的访问，这样就不必频繁触发VM-Exit了。要启用虚拟APIC页，需要使能如下**三个VM-Execution控制域字段：**

* **影子TPR(Use TPR Shadow)字段。**该字段置1后，虚拟机访问CR8时将会自动访问虚拟APIC页中的TPR寄存器。否则，虚拟机访问CR8时将会触发VM-Exit。
* **虚拟APIC访问(Virtualize APIC Accesses)字段。**该字段置1后，虚拟机对于APIC访问页(APIC Access Page)的访问将会触发APIC访问(APIC Access)异常类型的VM-Exit。单独设置该域时，与以前通过设置EPT页表项触发VM-Exit相比，仅仅只是将VM-Exit类型从EPT违例(EPT Violation)变为了APIC访问。需要进一步设置APIC寄存器虚拟化(APIC-Register Virtualization)字段消除VM-Exit。
* **APIC寄存器虚拟化字段。**该字段置1后，通常虚拟机对于APIC访问页的访问将被重定向到虚拟APIC页而不会触发VM-Exit，但部分情况下仍需要VM-Exit到Hypervisor中处理。

使用APICv前后，虚拟APIC访问方式如下图所示：

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311221554783.png" alt="image-20231122155250442" style="zoom:50%;" />

---

#### 虚拟中断递交

而虚拟中断递交 (Virtual-Interrupt Delivery) 通过VM-Execution控制域中的虚拟中断递交字段开启。在引入该机制前，Hypervisor需要设置 `VIRR`和 `VISR` 相应位（虚拟APIC中的 `IRR` 和 `ISR`），然后通过上文提到的事件注入机制在下一次VM-Entry时注入一个虚拟中断，调用客户机中相应的中断处理例程。

开启虚拟中断递交后，虚拟中断注入通过VMCS客户机状态域中的客户机中断状态 (Guest Interrupt Status) 字段完成：

* 其低8位为 `RVI`（Requesting Virtual Interrupt，待处理虚拟中断），表示虚拟机待处理中断中优先级最高的中断向量号，相当于 `IRRV` ；
* 而高8位为 `SVI`（Servicing Virtual Interrupt，处理中虚拟中断），表示虚拟机正在处理中断中优先级最高的中断向量号，相当于 `ISRV`。

开启虚拟中断传送后，Hypervisor只需要设置 `RVI` 的值，在VM-Entry时，CPU将会根据RVI的值进行虚拟中断提交，过程如下：

1. 若VM-Execution控制域中断窗口退出 (Interrupt-Window Exiting) 字段为0且 `RVI[7∶4] > VPPR[7∶4]`，则确认存在待处理的虚拟中断，其中VPPR指的是vCPU的PPR寄存器；
2. 根据 `RVI`，清除 `VIRR` 中对应位，设置 `VISR` 中对应位，并设置 `SVI=RVI`；
3. 设置 `VPPR = RVI & 0xf0`；
4. 若 `VIRR` 中还有非零位，则设置 `RVI=VIRRV`，即 `VIRR` 中优先级最高的中断向量号，否则设置 `RVI=0`；
5. 根据 `RVI` 提供的中断向量号，调用虚拟机中注册的中断处理例程。

在上述流程中，中断确认和中断交付工作将由硬件自动完成，Hypervisor无须手动设置虚拟APIC中 `VIRR` 和 `VISR` 寄存器的值。此外，设置 `RVI`后，即使当前vCPU处于屏蔽中断的状态也无妨，硬件会持续检查vCPU是否能够接收中断。一旦vCPU能接收中断，则立即进行虚拟中断交付，无须再通过前述中断窗口产生VM-Exit注入中断。

---

虚拟中断虽然省略了中断确认和中断递交的过程，但是中断接受仍需要Hypervisor完成。当Hypervisor需要将虚拟中断注入vCPU时，必须使其发生VM-Exit并设置好 `RVI` 的值，才能顺利进行后续操作。而**发布-中断 (Posted-Interrupt) 机制**可以省略中断接受的过程，直接让正在运行的vCPU收到一个虚拟中断，而不产生VM-Exit。

> 发布-中断机制还可以配合VT-d的发布-中断功能使用，实现直通设备的中断直接发给vCPU而不引起VM-Exit。

发布-中断机制通过VM-Execution控制域中的发布-中断处理 (Process Posted-Interrupt) 字段开启，它引入了发布-中断通知向量(Posted-Interrupt Notification Vector)和发布-中断描述符(Posted-Interrupt Descriptor)。其中发布-中断描述符保存在内存中，而发布-中断描述符的地址保存在VMCS的发布-中断描述符地址(Posted-Interrupt Descriptor Address)字段中。发布-中断描述符的格式如下图所示：

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311221554449.png" alt="image-20231122155421654" style="zoom:50%;" />

其中，各字段作用：

* `PIR`（Posted-Interrupt Requests，发布-中断请求）是一个256位的位图，与 `IRR` 类似，相应位置1表示有待处理中断；
* `ON`（Outstanding Notification，通知已完成）表示是否已经向CPU发送通知事件（ON本质上是一个物理中断），向CPU通知 `PIR` 中有待处理中断。

发布-中断处理字段置1后，当处于非根模式的CPU收到一个外部中断时，它首先完成中断接受和中断确认，并取得中断向量号。然后，若中断向量号与发布-中断通知向量相等，则进入发布-中断处理流程，否则照常产生VM-Exit。发布-中断处理流程如下：

1. 清除发布-中断描述符的 `ON` 位；
2. 向CPU的 `EOI` 寄存器写入0并执行，至此在硬件APIC上该中断已经处理完毕；
3. 令 `VIRR |= PIR` ，并清空 `PIR`；
4. 设置 `RVI = max(RVI,PIRV)`，其中 `PIRV` 为 `PIR` 中优先级最高的中断向量号；
5. CPU根据 `RVI` 按照前述流程递交虚拟中断。

从上述流程不难发现，**<font color='red'>只需要将待注入的中断放置在发布-中断描述符的 `PIR` 中，并向CPU发送一个发布-中断通知，CPU就会自动将 `PIR` 中存储的虚拟中断同步到 `RVI` 中，无须Hypervisor手动设置RVI的值。</font>**

**通过发布-中断机制便可以在不发生VM-Exit的情况下向vCPU中注入一个或者多个虚拟中断。**

# 3 QEMU/KVM CPU虚拟化实现

> 本节以x86架构下的QEMU/KVM实现为例，介绍前述硬件虚拟化技术如何被应用到实践中。

QEMU原本是纯软件实现的一套完整的虚拟化方案，支持CPU虚拟化、内存虚拟化以及设备模拟等，但是性能不太理想。随着硬件辅助虚拟化技术逐渐兴起，Qumranet公司基于新兴的虚拟化硬件实现了KVM。KVM遵循Linux的设计原则，以内核模块的形式动态加载到Linux内核中，利用虚拟化硬件加速CPU虚拟化和内存虚拟化流程，I/O虚拟化则交给用户态的QEMU完成，QEMU/KVM架构如下图所示：

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311221902898.png" alt="image-20231122164858226" style="zoom: 33%;" />

CPU虚拟化主要关心上图的左侧部分，即vCPU是如何创建并运行的，以及当vCPU执行敏感指令触发VM-Exit时，QEMU/KVM又是如何处理这些VM-Exit的。整体流程如下：

1. 当QEMU启动时，首先会解析用户传入的命令行参数，确定创建的虚拟机类型（通过QEMU-machine参数指定）与CPU类型（通过QEMU-cpu参数指定），并**创建相应的机器模型和CPU模型；**
2. 而后QEMU打开KVM模块设备文件并发起 `ioctl(KVM_CREATE_VM)`，**请求KVM创建一个虚拟机。**KVM创建虚拟机相应的结构体并为QEMU返回一个虚拟机文件描述符；
3. QEMU通过虚拟机文件描述符发起 `ioctl(KVM_CREATE_VCPU)` ，**请求KVM创建vCPU。**与创建虚拟机流程类似，KVM创建vCPU相应的结构体并初始化，返回一个vCPU文件描述符；
4. QEMU通过vCPU文件描述符发起 `ioctl(KVM_RUN)`，vCPU线程执行 `VMLAUNCH` 指令**进入非根模式，**执行虚拟机代码直至发生VM-Exit；
5. KVM根据VM-Exit的原因进行相应处理，如果与I/O有关，则需要进一步**返回到QEMU中**进行处理。

以上就是QEMU/KVM CPU虚拟化的主要流程。

---

本节将从**KVM模块初始化、虚拟机创建、vCPU创建和vCPU运行**四个方面进行介绍，最后给出CPU虚拟化实例。

> 在没有特殊说明的情况下，后续章节所列出的示例代码对应的**Linux内核版本为4.19.0，QEMU版本为4.1.1，**由于篇幅有限，示例代码只摘取了源码中的一部分。

## 3.1 KVM模块初始化

前面提到QEMU通过设备文件 `/dev/kvm` 发起ioctl系统调用请求KVM创建虚拟机，该设备文件便是在KVM模块初始化时创建的。

`kvm-intel.ko` 模块初始化函数为 `vmx_init`，该函数将会调用 `kvm_init` 函数进而调用 `misc_register` 函数**注册kvm_dev这一misc设备，**代码如下：

`linux-4.19.0/arch/x86/kvm/vmx.c`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311221650472.png" alt="image-20231122165049068" style="zoom:33%;" />

`linux-4.19.0/virt/kvm/kvm_main.c`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311221651404.png" alt="image-20231122165114833" style="zoom:33%;" />

注册成功后会在 `/dev/` 目录下产生名为kvm的设备节点，即 `/dev/kvm`。该设备文件对应的fd（文件描述符）的file_operations为`kvm_chardev_ops`。该设备仅支持ioctl系统调用，其中最重要的便是 `ioctl(KVM_CREATE_VM)` 。KVM模块接收该系统调用时将会创建虚拟机。`kvm_chardev_ops` 定义及具体代码如下：

`linux-4.19.0/virt/kvm/kvm_main.c`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311221652808.png" alt="image-20231122165153904" style="zoom:33%;" />

实际上，KVM模块初始化时，除了创建设备文件，还做了许多与架构相关的初始化工作，例如：

* `kvm_init` 接收的第一个参数vmx_x86_ops是一个函数指针集合，封装了Intel VT-x相关虚拟化操作。对于AMD-V而言，KVM则提供了另一个函数指针集合svm_x86_ops。KVM会根据当前所处平台选择将vmx_x86_ops或svm_x86_ops赋给 `arch/x86/kvm/x86.c` 中的全局变量kvm_x86_ops，这样后续通用x86虚拟化操作可以通过这一变量的相应成员调用 `arch/x86/kvm/vmx.c` 中的相应函数。kvm_x86_ops的设置工作由 `kvm_arch_init` 函数完成。
* 此外，`kvm_init` 函数还调用了 `kvm_hardware_setup` 函数，该函数最终会调用 `arch/86/kvm/vmx.c` 中的 `hardware_setup` 函数完成虚拟化硬件的初始化工作，这与Intel VT-x虚拟化硬件支持密切相关。部分 `hardware_setup` 代码如下：

`linux-4.19.0/arch/x86/kvm/vmx.c`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311221652601.png" alt="image-20231122165229572" style="zoom:33%;" />

1. `setup_vmcs_config` 函数读取物理CPU中与VMX相关的MSR（特殊模块寄存器），检测物理CPU对VMX的支持能力，生成一个 `vmcs_config` 结构，后续将根据 `vmcs_config` 设置VMCS控制域；
2. 而后 `hardware_setup` 函数调用 `cpu_has_vmx_ept` 等函数判断CPU是否支持EPT，若不支持，则将全局变量 `enable_ept` 置0。
3. `alloc_kvm_area` 函数则会调用 `alloc_vmcs_cpu` 函数**为每一个物理CPU分配一个4KB大小的区域作为VMXON区域。**Intel SDM指出执行VMXON指令需要提供一个4KB对齐的内存区间，即VMXON区域。VMXON区域的物理地址后续将作为VMXON指令的操作数。

## 3.2 虚拟机创建

虚拟机的创建始于 `kvm_init` 函数，该函数由QEMU调用 `TYPE_KVM_ACCEL` 类型QOM（QEMU Object Model，QEMU对象模型）对象的`init_machine` 成员来调用。由于篇幅有限，这里不再详述，仅画出相关函数调用流程，如下图所示。

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311221654322.png" alt="image-20231122165335748" style="zoom: 33%;" />

> 注：①QEMU进行ioctl系统调用。

KVM模块初始化后，`kvm_init` 函数便可以通过前述 `/dev/kvm` 设备文件发起 `ioctl(KVM_CREATE_VM)` ，请求创建虚拟机，相关代码如下：

`qemu-4.1.1/accel/kvm/kvm-all.c`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311221657954.png" alt="image-20231122165649008" style="zoom:33%;" />

`kvm_init` 函数首先打开 `/dev/kvm` 设备文件，然后通过相应的文件描述符发起 `ioctl(KVM_GET_API_VERSION)` 获取KVM模块的接口版本，检验与QEMU支持的KVM版本是否相等。随后 `kvm_init` 函数发起 `ioctl(KVM_CRETAE_VM)` 便会陷入前述KVM模块中的 `kvm_dev_ioctl` 函数进行处理。该函数根据ioctl类型调用 `kvm_dev_ioctl_create_vm` 函数创建虚拟机，相关代码如下：

`linux-4.19.0/virt/kvm/kvm-main.c`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311221657561.png" alt="image-20231122165730289" style="zoom:33%;" />

`kvm_dev_ioctl_create_vm` 首先调用 `kvm_create_vm` 函数创建虚拟机对应的结构体，然后调用 `anon_inode_getfd` 函数为虚拟机创建一个匿名文件，对应的file_operations为 `kvm_vm_fops`，其定义如下：

`linux-4.19.0/virt/kvm/kvm_main.c`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311221658885.png" alt="image-20231122165809892" style="zoom:33%;" />

这个文件为QEMU提供了虚拟机层级的API，如创建vCPU、创建设备等。KVM最终会将该文件的文件句柄作为返回值返回给QEMU供其使用。

---

`kvm_create_vm` 函数除了创建虚拟机对应的结构体以外，还会调用 `hardware_enable_all` 函数，该函数最终会在所有的物理CPU上调用`hardware_enable_nolock` 函数，该函数最终会通过前述kvm_x86_ops的 `hardware_enable` 成员调用vmx.c中的 `hardware_enable` 函数**使能VMX操作模式。**

`hardware_enable` 函数将获取为每个CPU分配的VMXON区域的物理地址作为参数传入 `kvm_vcpu_vmxon` 函数。`kvm_vcpu_vmxon` 函数会**设置CR4寄存器中的 `VMXE(VMX Enabled)` 位使能VMX操作模式，并执行VMXON指令进入VMX操作模式。**代码如下：

`linux-4.19.0/virt/kvm/kvm_main.c`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311221659758.png" alt="image-20231122165859541" style="zoom:33%;" />

`linux-4.19.0/arch/x86/kvm/vmx.c`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311221659507.png" alt="image-20231122165925104" style="zoom:33%;" />

## 3.3 vCPU创建

虚拟机创建完成后，QEMU便可以通过前述虚拟机文件描述符发起系统调用，请求KVM创建vCPU，完整的创建流程如下图所示：

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311221700826.png" alt="image-20231122170030586" style="zoom:33%;" />

> 注：①QEMU进行ioctl系统调用。

在QEMU/KVM中，每个vCPU对应宿主机操作系统中的一个线程，由QEMU创建，其执行函数为 `qemu_kvm_cpu_thread_fn`，该函数将调用`kvm_init_vcpu` 函数进而调用 `kvm_get_vcpu` 函数发起 `ioctl(KVM_CREATE_VCPU)`，其代码如下：

`qemu-4.1.1/accel/kvm/kvm_all.c`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311221701940.png" alt="image-20231122170141404" style="zoom:33%;" />

同虚拟机创建一样，KVM会为每个vCPU创建一个vCPU文件描述符返回给QEMU，然后QEMU发起 `ioctl(KVM_GET_VCPU_MMAP_SIZE)` 查看QEMU与KVM共享内存空间的大小。

其中共享内存空间的第一个页将被映射到KVM vCPU结构体 `struct kvm_vcpu` 的run成员，该成员将保存一些VM-Exit相关信息，如VM-Exit的原因以及部分虚拟机寄存器状态等，便于QEMU处理VM-Exit。`kvm_init_vcpu` 函数然后调用 `mmap` 函数将其映射到QEMU的虚拟地址空间中。

---

接下来，我们重点看KVM中的vCPU创建流程。

QEMU发起 `ioctl(KVM_CREATE_VCPU)` 后将陷入KVM模块进行处理，处理函数为 `kvm_vm_ioctl_create_vcpu`。`kvm_vm_ioctl_create_vcpu`函数首先调用 `kvm_arch_vcpu_create` 函数创建vCPU对应的结构体，然后调用 `create_vcpu_fd` 函数进而调用 `anon_inode_getfd` 为vCPU创建对应的文件描述符，对应的file_operations为 `kvm_vcpu_fops`。

`kvm_vcpu_fops` 定义了vCPU文件描述符对应的ioctl系统调用的处理函数，该函数为 `kvm_vcpu_ioctl`。主要代码如下。

`linux-4.19.0/virt/kvm/kvm_main.c`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311221702131.png" alt="image-20231122170233676" style="zoom:33%;" />

值得注意的是，`kvm_arch_vcpu_create` 函数除了创建vCPU对应的结构体以外，还完成了vCPU相应的VMCS初始化工作。前面提到**每个vCPU都有一个VMCS与其对应，使用时需要执行 `VMPTRLD` 指令将其与物理CPU绑定。**

`kvm_arch_vcpu_create` 函数通过前述kvm_x86_ops的 `vcpu_create` 成员调用vmx.c中的 `vmx_create_vcpu` 函数。`vmx_create_vcpu` 函数调用 `alloc_loaded_vmcs` 函数进而调用 `alloc_vmcs_cpu` 函数为vCPU分配VMCS结构，然后调用 `vmx_vcpu_load` 函数进而调用 `vmcs_load` 函数执行 `VMPTRLD` 指令将VMCS与当前CPU绑定，相关代码如下：

`linux-4.19.0/arch/x86/kvm/vmx.c`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311221703658.png" alt="image-20231122170326180" style="zoom:33%;" />

---

绑定之后，`vmx_create_vcpu` 函数调用 `vmx_vcpu_setup` 函数通过若干 `vmcs_write16/vmcs_write32/vmcs_write64` 函数进而调用`__vmcs_writel` 函数设置VMCS中相关域的值，如前所述，`vmcs_writexx` 函数最终会执行 `VMWRITE` 指令写VMCS，相关代码如下：

`linux-4.19.0/arch/x86/kvm/vmx.c`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311221704961.png" alt="image-20231122170417429" style="zoom:33%;" />

`vmcs_setup_config` 函数主要设置VMCS中控制域的值，客户机状态域和宿主机状态域的初始化工作则由 `kvm_vm_ioctl_create_vcpu` 函数调用 `kvm_arch_vcpu_setup` 函数进而调用 `kvm_vcpu_reset` 函数完成。`kvm_vcpu_reset` 函数通过kvm_x86_ops的 `vcpu_reset` 成员调用vmx.c中的 `vmx_vcpu_reset` 函数完成，部分代码如下：

`linux-4.19.0/arch/x86/kvm/vmx.c`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311221705749.png" alt="image-20231122170513125" style="zoom:33%;" />

根据上述代码，KVM会调用 `vmcs_writel` 函数将VMCS中客户机的代码段寄存器 `GUEST_CS_SELECTOR` 设置为0xf000，将代码段基地址`GUEST_CS_BASE` 设置为0xffff0000。而 `kvm_rip_write` 函数则会将KVM模拟的虚拟 `RIP`（Return Instruction Pointer，返回指令指针）寄存器设为0xfff0，当后续调用 `vmx_vcpu_run` 函数运行vCPU时，该函数会将模拟的 `RIP` 寄存器值写入VMCS中，部分代码如下：

`linux-4.19.0/arch/x86/kvm/vmx.c`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311221706680.png" alt="image-20231122170637463" style="zoom:33%;" />

这与Intel x86架构硬件要求相符。在Intel x86架构下，计算机加电后会将 `CS`（Code Segment，代码段）寄存器设置为0xf000，`RIP` 寄存器设置为0xfff0，而CS寄存器中隐含的代码段基地址则设置为 `0xffff0000`，这样当程序启动时，执行的第一条指令位于0xfffffff0处 `CS_BASE+RIP`。

值得注意的是，这里KVM仅仅是对VMCS中的客户机状态域进行初始化，QEMU仍可以通过 `KVM API` 设置VMCS客户机状态域。QEMU在调用`kvm_cpu_exec` 函数运行虚拟机代码前，首先调用 `kvm_arch_put_registers` 函数，该函数将会调用 `kvm_getput_regs` 函数和 `kvm_put_sregs` 函数，通过前述vCPU设备文件描述符发起 `ioctl(KVM_SET_REGS)` 和 `ioctl(KVM_SET_SREGS)` ，分别设置vCPU通用寄存器和段寄存器的值，KVM会进而将QEMU传入的寄存器值写入VMCS中。具体寄存器的值则由 `x86_cpu_reset` 函数指定，相关代码如下：

`qemu-4.1.1/target/i386/cpu.c`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311221707341.png" alt="image-20231122170749577" style="zoom:33%;" />

`qemu-4.1.1/target/i386/kvm.c`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311221710028.png" alt="image-20231122171005785" style="zoom:33%;" />

根据上述代码，QEMU传入的 `CS` 和 `RIP` 寄存器的值与默认VMCS相应域的设置相同：`CS=0xf000`，`RIP=0xfff0`，`CS_BASE=0xffff0000`。

---

这里通过一个小实验验证上述代码的有效性。QEMU提供了- `s` 和 `-S` 选项允许GDB远程连接调试内核，其中 `-s` 选项使得QEMU等待来自1234端口的TCP连接，`-S` 选项则使得QEMU阻塞客户机执行，直到远程连接的GDB（Linux下常用的程序调试器）允许它继续执行，这允许用户方便地在GDB中查看虚拟机运行过程中物理寄存器的状态。

可以打开两个终端，终端1 `Terminal 1` 启动QEMU等待GDB连接，终端2 `Terminal 2` 则运行GDB通过1234端口远程连接至QEMU，命令如下：

`Terminal 1`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311221717185.png" alt="image-20231122171239861" style="zoom:33%;" />

`Terminal 2`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311221829082.png" alt="image-20231122172033861" style="zoom:33%;" />

根据上述实验，vCPU启动后，`CS` 寄存器和 `RIP` 寄存器的值与QEMU设置的一致，说明虚拟机启动时硬件会将VMCS中相应域加载到寄存器中。

而为了兼容早期8086架构，程序启动后，`0xffff0` 和 `0xfffffff0` 内存处指令一致，都是跳转至BIOS进行初始化，加载bootloader。在`0xfffffff0` 处设置断点，运行虚拟机后发现断点被触发；而删去断点后，虚拟机正常启动，说明程序的入口位于 `0xfffffff0` 处。改写后的`x86_cpu_reset` 函数代码如下：

`qemu-4.1.1/target/i386/cpu.c`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311221720693.png" alt="image-20231122172005657" style="zoom:33%;" />

如上所述，将CS寄存器设置为 `0x1234`，将 `RIP` 寄存器设置为 `0x5678`，重新编译QEMU并运行上述代码，实验结果如下。

`Terminal 3`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311221830763.png" alt="image-20231122183007439" style="zoom:33%;" />

从GDB输出可以发现上述寄存器设置生效，感兴趣的读者可以自行修改QEMU `target/i386/cpu.c` 中的 `x86_cpu_reset` 函数，重复上述实验。

---

以上便是QEMU/KVM vCPU创建与初始化流程，与虚拟机创建类似，vCPU创建的起点实际上位于 `TYPE_X86_CPU` 类型的QOM对象的realize_fn成员中。该成员在CPU对应的QOM对象初始化时被设置为 `x86_cpu_realizefn` 函数，该函数调用 `qemu_init_vcpu` 函数创建了vCPU线程，线程执行函数为前述的 `qemu_kvm_cpu_thread_fn`。

## 3.4 vCPU运行

在QEMU中，vCPU线程创建完成后，便会进入循环状态，等到条件满足时开始运行虚拟机代码。虚拟机执行敏感指令或收到外部中断时，便会触发VM-Exit，进入KVM模块进行处理；部分虚拟机退出时，则需要进一步退出到QEMU中进行处理，如I/O模拟。处理完成后，恢复虚拟机运行。vCPU整体运行流程如下图所示：

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311221826877.png" alt="image-20231122182612556" style="zoom:50%;" />

**QEMU/KVM vCPU运行函数调用流程**如下图所示：

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311221828294.png" alt="image-20231122182801479" style="zoom: 50%;" />

> 注：①QEMU进行ioctl系统调用，进入KVM；②从ioctl系统调用返回。

接下来遵循上述流程查看QEMU/KVM中vCPU运行是如何实现的，如下：

1. 首先 `qemu_kvm_cpu_thread_fn` 函数调用 `kvm_init_vcpu` 函数在KVM中创建vCPU对应的结构体，得到相应的vCPU文件描述符后便会进入循环；
2. 在循环中，先判断CPU能否运行，若不能运行便调用 `qemu_kvm_wait_io_event` 函数进而调用 `qemu_cond_wait` 函数等待 `cpu->halt_cond`；
3. 而QEMU后续在 `main` 函数中将会调用 `vm_start` 函数直至最终调用 `qemu_cond_broadcast` 函数唤醒所有的vCPU线程，相关代码如下：

`qemu-4.1.1/cpus.c`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311221842899.png" alt="image-20231122184211163" style="zoom:33%;" />

vCPU线程被唤醒后将执行 `kvm_cpu_exec` 函数运行vCPU。`kvm_cpu_exec` 函数首先调用前述 `kvm_arch_put_registers` 函数设置vCPU通用寄存器和段寄存器的值，然后调用 `kvm_vcpu_ioctl` 通过vCPU文件描述符发起 `ioctl(KVM_RUN)` 调用陷入KVM中。

KVM中相应的处理函数为前述 `kvm_arch_vcpu_ioctl_run`，该函数首先调用 `vcpu_load` 函数，该函数最终通过kvm_x86_ops的 `vcpu_load` 成员调用前述 `vmx_vcpu_load` 函数执行VMPTRLD指令绑定该vCPU对应的VMCS，然后调用 `vcpu_run` 函数运行vCPU。

`vcpu_run` 函数首先调用 `kvm_vcpu_running`  函数判断该vCPU能否运行，若能运行则调用 `vcpu_enter_guest` 函数准备进入非根模式。具体代码如下：

`linux-4.19.0/arch/x86/kvm/x86.c`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311221841875.png" alt="image-20231122183647622" style="zoom:33%;" />

---

`vcpu_enter_guest` 函数首先判断vCPU是否存在待处理的请求，并调用一系列 `kvm_check_request` 函数对这些请求进行处理。这些请求可能在各个地方发生，以中断事件注入为例，当虚拟IOAPIC需要注入一个虚拟中断时，会调用 `kvm_make_request` 函数发起一个KVM_REQ_EVENT类型的请求，设置 `kvm_vcpu` 结构中的requests成员对应的位，并在 `vcpu_enter_guest` 函数中进行处理。`vcpu_enter_guest` 函数检查requests发现有KVM_REQ_EVENT类型的请求，则调用 `inject_pending_event` 函数注入相应事件，后续中断虚拟化部分将详述该流程。具体代码如下。

`linux-4.19.0/arch/x86/kvm/x86.c`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311221841672.png" alt="image-20231122183735630" style="zoom:33%;" />

阻塞请求处理完成后，`vcpu_enter_guest` 函数通过kvm_x86_ops的 `prepare_guest_switch` 成员调用vmx.c中的`vmx_prepare_switch_to_guest` 函数，该函数将部分宿主机的状态保存到VMCS与 `host_state` 中，如 `FS/GS` 相应的段选择子和段地址等。具体代码如下：

`linux-4.19.0/arch/x86/kvm/vmx.c`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311221841427.png" alt="image-20231122183902417" style="zoom:33%;" />

然后 `vcpu_enter_guest` 函数通过kvm_x86_ops的run成员调用vmx.c中的 `vmx_vcpu_run` 函数，该函数通过一系列内联汇编指令保存宿主机通用寄存器的值，并加载客户机通用寄存器的值，然后**调用 `VMLAUNCH/VMRESUME` 指令进入非根模式执行虚拟机代码。**具体代码如下：

`linux-4.19.0/arch/x86/kvm/vmx.c`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311221844541.png" alt="image-20231122184318049" style="zoom:33%;" />

---

当vCPU发生VM-Exit时，`vmx_vcpu_run` 函数保存虚拟机通用寄存器的值并从VMCS中读取虚拟机退出原因保存至 `vmx->exit_reason` 中，然后返回至 `vcpu_enter_guest` 函数。

`vcpu_enter_guest` 函数通过kvm_x86_ops的 `handle_exit` 成员调用vmx.c中的 `vmx_handle_exit` 函数，该函数根据前面的 `vmx->exit_reason` 调用全局数组 `kvm_vmx_exit_handlers` 中对应的虚拟机退出处理函数。

> 对于由I/O指令触发的EXIT_REASON_IO_INSTRUCTION类型的VM-Exit，它对应的处理函数 `handle_io` 最终调用 `emulator_pio_in_out` 函数进行处理。

`emulator_pio_in_out` 函数首先调用 `kernel_pio` 函数尝试在KVM中处理该PIO请求，若KVM无法处理，它将 `vcpu->run->exit_reason` 设置为KVM_EXIT_IO并最终返回0，这导致 `vcpu_run` 退出循环并返回至QEMU中进行处理。前面提到vCPU中的run成员主要保存VM-Exit相关信息并通过 `mmap` 与QEMU共享，故退出至QEMU后，QEMU将读取 `kvm_run` 结构中的exit_reason成员，根据其退出原因进行进一步处理。具体代码如下：

`linux-4.19.0/arch/x86/kvm/vmx.c`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311221844097.png" alt="image-20231122184400824" style="zoom:33%;" />

`linux-4.19.0/arch/x86/kvm/x86.c`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311221845970.png" alt="image-20231122184457895" style="zoom:33%;" />

## 3.5 实验: CPU虚拟化实例

前几节从源码层级分析了QEMU/KVM CPU虚拟化的完整流程，不难发现大部分工作都由KVM完成，QEMU主要负责维护虚拟机和vCPU模型，并通过KVM模块文件描述符、虚拟机文件描述符和vCPU文件描述符调用KVM API接口，本节实现了一个类似于QEMU的小程序，执行指定二进制代码输出 `Hello，World!`。主要代码如下：

`sample-qemu.c`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311221901266.png" alt="image-20231122190102707" style="zoom:33%;" />

代码流程如下：

1. 通过前面所述的若干ioctl调用创建并运行vCPU，将vCPU起始指令地址设置为 `0x1000(GPA)`，并将指定的二进制代码复制到相应的内存位置(HVA)；
2. 二进制代码将依次调用OUT指令向 `0x3f8` 端口写入 `Hello，World!` 包含的各个字符，触发 `EXIT_REASON_IO_INSTRUCTION` 类型的VM-Exit，这使得程序退回到用户态进行处理，用户态程序调用 `putchar` 函数输出对应字符；
3. 二进制代码最终执行 `hlt` 指令触发VM-Exit，系统回到用户态并退出应用程序。

# 4 QEMU/KVM 中断虚拟化实现

本节将结合QEMU/KVM源码介绍中断虚拟化的具体实现，包括：

* **PIC与APIC等中断控制器的模拟**
* **虚拟中断递交流程的模拟**

本节在最后将通过GDB查看中断从产生到注入的完整流程。

## 4.1 PIC/IOAPIC模拟

QEMU和KVM都实现了对中断芯片的模拟，这是由于历史原因造成的。早在KVM诞生之前，QEMU就提供对一整套设备的模拟，包括中断芯片。而KVM诞生之后，为了进一步提高中断性能，又在KVM中模拟了一套中断芯片。

QEMU提供 `kernel-irqchip` 参数决定中断芯片由谁模拟，`kernel-irqchip` 参数可取值为：

* `on`：PIC和APIC中断芯片均由KVM模拟；
* `off`：PIC和APIC中断芯片均由QEMU模拟；
* `split`：PIC和IOAPIC由QEMU模拟，LAPIC由KVM模拟。

由于KVM模拟中断芯片性能更高，故本节**以KVM模拟中断芯片为例。**

### PIC模拟

当QEMU设置 `kernel-irqchip` 参数为on时，会在前述 `kvm_init` 函数中调用 `kvm_irqchip_create` 函数，通过虚拟机文件描述符发起`ioctl(KVM_CREATE_IRQCHIP)` 陷入KVM中。KVM中相应处理函数 `kvm_arch_vm_ioctl` 调用 `kvm_pic_init` 函数创建并初始化PIC相应的结构体 `struct kvm_pic`，并将指针保存在 `kvm->arch.vpic` 中，相关代码如下：

`linux-4.19.0/arch/x86/kvm/irq.h`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311221906319.png" alt="image-20231122190606179" style="zoom:33%;" />

`linux-4.19.0/arch/x86/kvm/x86.c`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311221907374.png" alt="image-20231122190711985" style="zoom:33%;" />

`linux-4.19.0/arch/x86/kvm/i8259.c`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311221907617.png" alt="image-20231122190739192" style="zoom:33%;" />

根据上述代码，PIC寄存器状态保存在结构体 `struct kvm_kpic_state` 中，该结构体包含了前述 `IRR`、`ISR` 以及 `IMR` 等的状态。

此外，如前所述，通常将两块8259A芯片级联以支持更多的中断来源，`kvm_pic` 将主PIC和从PIC的状态保存在 `pics` 成员中，`kvm_pic` 对应的内存空间由 `kvm_pic_init` 函数分配。

除了模拟PIC寄存器状态，`kvm_pic_init` 函数还提供CPU对PIC访问的支持。CPU通常通过PIO的方式访问PIC，于是 `kvm_pic_init` 函数首先调用 `kvm_io_device_init` 函数初始化主PIC、从PIC和eclr三个设备对应的读写操作函数，并调用 `kvm_io_bus_register_dev` 函数在KVM_IO_BUS上注册三个设备相应的I/O端口。

---

当CPU通过PIO相关指令（IN、OUT等）通过注册端口对设备进行读写时，将会触发 `EXIT_REASON_IO_INSTRUCTION` 类型的VM-Exit，其对应的处理函数为 `handle_io`。`handle_io` 函数首先会判断I/O指令类型是否为string I/O指令，若是则调用 `kvm_emulate_instruction` 函数模拟I/O指令，否则调用 `kvm_fast_pio` 函数处理，二者最后都会调用 `kernel_pio` 函数。

对于读/写端口指令，`kernel_pio` 函数分别调用 `kvm_io_bus_read/kvm_io_bus_write` 函数。以写端口为例，`kvm_io_bus_write` 函数将会调用 `_kvm_io_bus_write` 函数，该函数根据I/O端口从KVM_IO_BUS总线上找到相应的设备并调用 `kvm_iodevice_write` 函数写入设备端口，这会触发前述设备对应的读写操作函数。以主PIC为例，其对应的读/写操作函数代码如下：

`linux-4.19.0/arch/x86/kvm/i8259.c`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311221910505.png" alt="image-20231122191029212" style="zoom:33%;" />

`linux-4.19.0/include/kvm/iodev.h`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311221910260.png" alt="image-20231122190904382" style="zoom:33%;" />

`linux-4.19.0/virt/kvm/kvm_main.c`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311221911015.png" alt="image-20231122191049578" style="zoom:33%;" />

### IOAPIC模拟

同PIC创建类似，IOAPIC相应的结构体 `struct kvm_ioapic` 由 `kvm_arch_vm_ioctl` 函数调用 `kvm_ioapic_init` 函数创建并初始化，同时将`kvm_ioapic` 指针赋给 `kvm->arch.vioapic` 。相关代码如下：

`linux-4.19.0/arch/x86/kvm/ioapic.c`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311221912890.png" alt="image-20231122191153907" style="zoom:33%;" />

> 对于 `kvm_ioapic` 结构体而言，其中最重要的成员为 `redirtbl`，该成员对应于IOAPIC中的 `PRT`（Programmable Redirection Table，可编程重定向表）。如前所述，当IOAPIC某个引脚收到中断时，将根据该引脚对应的RTE格式化出一条中断消息发送给指定LAPIC，后续的中断路由章节将详述这一成员的作用。

`kvm_ioapic` 结构体定义如下。

`linux-4.19.0/arch/x86/kvm/ioapic.h`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311221912527.png" alt="image-20231122191234503" style="zoom:33%;" />

与PIC不同的是，CPU通常通过MMIO的方式访问IOAPIC，故 `kvm_ioapic_init` 函数先调用 `kvm_iodevice_init` 函数，将IOAPIC MMIO区域访问回调函数设置为 `ioapic_mmio_ops`，然后调用 `kvm_io_bus_register_dev` 函数在KVM_MMIO_BUS上注册IOAPIC MMIO区域。IOAPIC MMIO区域的起始地址为 `0xfec00000`（由 `kvm_ioapic_reset` 函数指定），长度为 `0x100`。相关代码如下：

`linux-4.19.0/arch/x86/kvm/ioapic.c`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311221913537.png" alt="image-20231122191314308" style="zoom:33%;" />

当KVM首次访问MMIO区域时，由于EPT中缺乏相应的映射，会触发EPT Violation类型的VM-Exit，KVM发现该区域为MMIO区域，在为其建立相应的页表项时会将该页表项设置为可写可执行但不可读，这样在后续访问该MMIO区域时会触发EPT配置错误 (EPT Misconfig) 类型的VM-Exit，在`kvm_vmx_exit_handlers` 函数中相应的处理函数为 `handle_ept_misconfig`。与PIC访问类似，`handle_ept_misconfig` 函数最终会调用`x86_emulate_instruction` 函数对该指令进行模拟。

在大部分情况下，由于外部设备由QEMU模拟，故 `x86_emulate_instruction` 函数将会返回EMULATE_USER_EXIT，导致 `vcpu_enter_guest` 函数返回至QEMU中处理。但因为IOAPIC由KVM模拟，故 `x86_emulate_instruction` 函数最终会调用 `vcpu_mmio_read/vcpu_mmio_write` 函数处理对于IOAPIC MMIO区域的读写。以 `vcpu_mmio_write` 函数为例，它同样调用前述 `kvm_io_bus_write` 函数，根据访问的内存地址在KVM_MMIO_BUS找到IOAPIC对应的设备，并调用前面注册的写回调函数 `ioapic_mmio_write`。`vcpu_mmio_write` 函数代码如下。

`linux-4.19.0/arch/x86/kvm/x86.c`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311221915231.png" alt="image-20231122191435082" style="zoom:33%;" />

### LAPIC模拟

LAPIC创建路径与PIC和IOAPIC不同，由于LAPIC与vCPU一一对应，故LAPIC将在创建vCPU时构造，这一工作由前述 `vmx_create_vcpu` 函数最终调用 `kvm_create_lapic` 函数完成。`kvm_create_lapic` 函数首先分配LAPIC在KVM中对应的结构体 `struct kvm_lapic`，并将其保存在 `vcpu->arch.apic` 中，然后分配LAPIC寄存器页保存LAPIC寄存器状态，寄存器页的起始地址保存在 `kvm_lapic` 的regs成员中。`kvm_lapic` 定义如下：

`linux-4.19.0/arch/x86/kvm/lapic.h`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311221915632.png" alt="image-20231122191530103" style="zoom:33%;" />

`linux-4.19.0/arch/x86/kvm/lapic.c`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311221923286.png" alt="image-20231122191925850" style="zoom:33%;" />

与IOAPIC类似，CPU也通过MMIO的方式访问LAPIC。但是**根据不同的VM-Execution控制域设置，KVM对于LAPIC MMIO有三种处理方式，**这里仍以写MMIO区域为例：

> **虚拟APIC访问字段为0**

此时LAPIC MMIO处理流程与IOAPIC相同，但是LAPIC MMIO区域并未在KVM_MMIO_BUS上注册。`vcpu_mmio_write` 函数调用 `lapic_in_kernel` 函数判断LAPIC是否由KVM模拟：

* 若是，则直接调用 `kvm_io_device_write` 函数尝试写入LAPIC，这将会触发 `apic_mmio_ops` 函数中的 `apic_mmio_write` 回调函数，`apic_mmio_write` 函数将会调用 `apic_mmio_in_range` 函数判断访问的内存地址是否落在LAPIC MMIO区域中；
* 若不是，将访问地址减去 `base_address` 得到 `offset`（偏移地址），然后通过 `kvm_lapic_reg_write` 函数写入LAPIC相应的寄存器中。

`apic_mmio_write` 函数代码如下：

`linux-4.19.0/arch/x86/kvm/lapic.c`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311221923374.png" alt="image-20231122191737714" style="zoom:33%;" />

---

> **虚拟APIC访问字段为1且虚拟APIC寄存器虚拟化字段为0**

此时通过MMIO方式访问LAPIC时，会触发APIC访问类型的VM-Exit。LAPIC MMIO区域的地址通过APIC访问地址(APIC Access Address)字段指定。

在前述vCPU创建流程中，将调用 `vcpu_vmx_reset` 函数，该函数会调用 `kvm_make_request` 函数发起一个 `KVM_REQ_APIC_PAGE_RELOAD` 类型的请求。后续vCPU调用 `vcpu_enter_guest` 函数运行前，会检查是否有待处理的请求。

若有未处理的 `KVM_REQ_APIC_PAGE_RELOAD` 请求，`vcpu_enter_guest` 函数会调用 `kvm_vcpu_reload_apic_access_page` 函数处理，该函数通过 `kvm_x86_ops` 函数的 `set_apic_access_page_addr` 成员调用vmx.c中的 `vmx_set_apic_access_page_addr` 函数，将VMCS APIC访问地址字段设置为APIC MMIO区域起始地址对应的HPA。

而APIC Access类型VM-Exit的处理函数为 `handle_apic_access`，该函数最终会调用 `kvm_emulate_instruction` 函数进而调用`x86_emulate_instruction` 函数，实际处理函数与上一种方式相同，只是VM-Exit类型变为APIC访问。相关代码如下。

`linux-4.19.0/arch/x86/kvm/x86.c`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311221922637.png" alt="image-20231122192044197" style="zoom:33%;" />

`linux-4.19.0/arch/x86/kvm/vmx.c`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311221922157.png" alt="image-20231122192123884" style="zoom:33%;" />

> **虚拟APIC访问字段为1且APIC寄存器虚拟化字段为1**

此时CPU通过MMIO访问LAPIC时会被重定向到虚拟APIC页，而不会触发VM-Exit。在这种情况下，只有特定寄存器访问可能会触发APIC访问或APIC写(APIC Write)异常类型的VM-Exit，此处不再详述，读者可以参考Intel SDM相关资料了解哪些APIC寄存器访问会触发VM-Exit。

而虚拟APIC页地址由VMCS中的虚拟APIC页地址 (Virtual-APIC Address) 字段指定，这一工作由前述vCPU创建流程中的 `vmx_vcpu_reset` 函数完成，虚拟APIC地址字段被设置为 `kvm_lapic` 中的regs成员，即寄存器页的物理地址。`vmx_vcpu_reset` 函数代码如下：

`linux-4.19.0/arch/x86/kvm/vmx.c`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311221922800.png" alt="image-20231122192218756" style="zoom:33%;" />

## 4.2 PCI设备中断

在APIC中断架构下，CPU中断来源主要有三种：**本地中断、通过IOAPIC接收的外部中断以及处理器间中断。**

本节侧重于介绍通过IOAPIC接收的外部中断。但是由于芯片组的差异以及外部设备的差异，设备产生和传递中断的流程也不尽相同。为了与后续的实验部分相照应，本节主要介绍在 `i440FX + PIIX` 芯片组中PCI设备中断模拟。

> 4.3节将会介绍QEMU模拟设备产生中断是如何传递给KVM模拟的APIC的。

### PCI设备中断原理

计算机主板上通常有多个PCI插槽供PCI设备使用，插入PCI槽的PCI卡通常称为物理PCI设备，每个**物理PCI设备**上可能存在**多个独立的功能单元，**这些功能单元也称为**逻辑PCI设备。**因此**逻辑PCI设备的标识符**通常包括三部分：**设备所在的总线(Bus)号、设备所在的物理PCI设备(Device)号、设备的功能(Function)号，简称为BDF。**

每个物理PCI设备有4个中断引脚：`INTA#`、`INTB#`、`INTC#` 和 `INTD#`，单功能物理PCI设备只会使用 `INTA#` 引脚，多功能物理PCI设备可能会用到其余中断引脚。由于一个物理PCI设备最多包含8个逻辑PCI设备，故存在**多个逻辑PCI设备共享同一个中断引脚**的情况。

逻辑PCI设备配置空间中的中断引脚 (Interrupt `Pin:0x3D`) 寄存器记录了该设备使用哪个中断引脚，`1~4` 分别对应 `INTA#~INTD#` 引脚。PCI总线通常提供4条或8条中断请求线供PCI设备使用，物理PCI设备中断引脚将连接到这些中断请求线上。假设PCI总线提供了4条中断请求线：`LNKA`、`LNKB`、`LNKC` 和 `LNKD` ，并非所有的PCI设备 `INTA#` 引脚都连接至固定 `LNKA` 中断请求线，因为通常大部分PCI设备都使用 `INTA#` 引脚，这样会造成各中断请求线负载不均衡的情况。PCI设备中断路由连接如下图所示：

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311221929327.png" alt="image-20231122192505047" style="zoom:50%;" />

> 此时PCI总线中断请求线与PCI设备引脚映射关系为 `LNK = (D+I) mod 4`，其中 `LNK` 表示PCI总线中的中断请求线号，`D` 表示物理PCI设备号，`I` 表示物理PCI设备的中断引脚号。

上述连接方式使得各物理PCI设备中断引脚交错连接至PCI总线中断请求线。而PCI总线中断请求线还需要连接至中断控制器，它们与中断控制器引脚之间的映射关系保存在BIOS中的中断路由表中。这样每个PCI逻辑设备都连接至中断控制器的一个中断引脚，中断控制器引脚号记录在PCI配置空间的中断线 (Interrupt Line, 0x3C) 寄存器中。**当PCI设备发起中断请求时，便会通过映射的中断控制器引脚将中断递交给CPU。**

### **PCI设备中断模拟**

在QEMU中，虚拟PCI设备调用 `pci_set_irq` 函数来**触发中断，**具体流程如下：

1. `pci_set_irq` 函数首先调用 `pci_intx` 函数获取该PCI设备使用的中断引脚，`pci_intx` 函数将会从该设备的PCI配置空间中读取中断引脚寄存器的值并减1；
2. 然后 `pci_set_irq` 函数将调用 `pci_irq_handler` 处理PCI设备中断。`pci_irq_handler` 函数首先判断当前中断引脚的状态是否发生改变，若改变，则调用 `pci_set_irq_state` 函数设置设备中断引脚的状态，并调用 `pci_update_irq_status` 函数更新PCI设备的状态。若当前PCI设备中断未被禁用，则 `pci_irq_handler` 函数最终会调用 `pci_change_irq_level` 函数触发中断。相关代码如下：

`qemu-4.1.1/hw/pci/pci.c`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311221929219.png" alt="image-20231122192558988" style="zoom:33%;" />

---

`pci_change_irq_level` 函数首先获取设备所在的PCI总线，然后调用其 `map_irq` 回调函数获取该设备中断引脚所连接的中断请求线，最后调用所在PCI总线的 `set_irq` 回调函数触发中断。

以QEMU `i440FX+PIIX` 架构为例，其PCI总线 `map_irq` 相应的回调函数为 `pci_slot_get_pirq`，`set_irq` 相应的回调函数为 `piix3_set_irq`，这两个回调函数在i440FX芯片组初始化函数 `i440fx_init` 中注册。`i440fx_init` 函数首先调用 `pci_root_bus_new` 函数创建PCI根总线，然后调用 `pci_bus_irqs` 函数设置PCI根总线对应QOM对象的 `map_irq` 成员和 `set_irq` 成员。相关代码如下：

`qemu-4.1.1/hw/pci-host/piix.c`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311221929125.png" alt="image-20231122192703238" style="zoom:33%;" />

`qemu-4.1.1/hw/pci/pci.c`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311221930622.png" alt="image-20231122193011131" style="zoom:33%;" />

`pci_slot_get_pirq` 函数首先获取PCI设备的设备号，然后通过设备号和它使用的PCI设备中断引脚获取该PCI设备连接至PCI总线中的哪一条中断请求线。QEMU规定i440FX芯片组中PCI设备与PCI总线中断线的映射关系为 `(slot+pin)&3->"LNK[D|A|B|C]"`，而非 `(slot+pin)&3->"LNK[A|B|C|D]"`。代码如下：

`qemu-4.1.1/hw/pci-host/piix.c`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311221928712.png" alt="image-20231122192844118" style="zoom:33%;" />

`piix3_set_irq` 回调函数则主要调用 `piix3_set_irq_level` 函数，该函数首先访问PIIX3设备配置空间，获取PCI总线中断请求线对应的中断控制器引脚号。对于 `PIIX3/PIIX4` 来说，其配置空间中的中断请求线路由控制寄存器 ( `PIRQRC[A:D],0x60~0x63` ) 分别记录了 `LNKA~LNKD` 中断请求线对应的中断控制器引脚号，这几个寄存器的值由BIOS设置。相关代码如下：

`qemu-4.1.1/hw/pci-host/piix.c`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311221931569.png" alt="image-20231122193059886" style="zoom:33%;" />

`piix3_set_irq_level` 函数获得PCI设备对应的中断控制器引脚号后调用 `piix3_set_irq_pic` 函数。`piix3_set_irq_pic` 函数根据传入的中断控制器引脚号索引PIIX3State结构中的 `pic` 成员，该成员在 `i440fx_init` 函数中被设置为 `pcms->gsi`，故 `piix_set_irq_pic` 函数本质上索引的是 `pcms->gsi`。

> `pcms->gsi` 是**QEMU中断路由的起点，**其用法将在4.3节介绍。

`piix3_set_irq_pic` 函数从 `pcms->gsi` 中获得一个 `qemu_irq` 后将其传递给 `qemu_set_irq` 函数。

* `qemu_irq` 是一个指向IRQState结构体的指针，在QEMU中，IRQState通常用于表示设备或中断控制器的中断引脚，`n` 表示中断引脚号，`opaque` 由创建者指定，`handler` 表示该中断引脚收到信号时的处理函数。
* `qemu_set_irq` 函数所做的工作其实就是调用传入 `qemu_irq` 的handler函数，相关代码如下：

`qemu-4.1.1/hw/core/irq.c`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311221931493.png" alt="image-20231122193145582" style="zoom:33%;" />

`qemu-4.1.1/hw/pci-host/piix.c`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311221932450.png" alt="image-20231122193212078" style="zoom:33%;" />

## 4.3  QEMU/KVM中断路由

4.2节介绍了PCI设备中断的模拟，而外部设备产生的中断需要经由中断控制器才能传递给CPU，故本节将介绍**外部设备中断路由至中断控制器的完整流程。**本节仍以KVM模拟中断控制器为例。

### QEMU中断路由

**QEMU中断路由的起点**位于 `pcms->gsi`，它本质上是一个 `qemu_irq` 数组，为所有的虚拟设备定义了统一的中断传递入口。QEMU根据传入的中断控制器引脚号以及系统当前使用的中断控制器类型，决定该中断如何进行传递。完整的QEMU中断路由流程如下图所示。

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311221933165.png" alt="image-20231122193334156" style="zoom: 33%;" />

`pcms` 是PCMachineState类型的结构体，可以理解为整个虚拟机的设备模型，其 `gsi` 成员在虚拟机设备模型初始化函数 `pc_init1` 中创建：

* `pc_init1` 函数首先调用 `kvm_ioapic_in_kernel` 函数判断IOAPIC芯片是否由KVM模拟；
  * 若是，则调用 `qemu_allocate_irqs` 函数；
    * 进而调用 `qemu_extend_irqs` 函数创建若干个IRQState结构体，其handler为`kvm_pc_gsi_handler`，opaque则指向预先分配的GSIState结构体。

相关代码如下：

`qemu-4.1.1/hw/i386/pc_piix.c`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311221935295.png" alt="image-20231122193507216" style="zoom:33%;" />

`qemu-4.1.1/hw/core/irq.c`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311221935137.png" alt="image-20231122193543132" style="zoom:33%;" />

---

`pcms->gsi` 创建完成后，`pc_init1` 函数调用 `i440fx_init` 函数将 `pcms->gsi` 赋给PIIX3State中的 `pic` 成员，相关代码如下：

`qemu-4.1.1/hw/pci-host/piix.c`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311221936244.png" alt="image-20231122193643293" style="zoom:33%;" />

前面提到PCI设备中断模拟最终通过 `qemu_set_irq` 调用 `PIIX3State pic` 成员中的某个 `qemu_irq` 数组的handler，而其 `pic` 成员被设置为 `pcms->gsi`。

实际上对于ISA总线，`pc_init1` 函数也会调用 `isa_bus_irqs` 函数将 `pcms->irq` 赋值给isabus的 `irqs` 成员。因此最终ISA设备和PCI设备中断都会调用 `pcms->gsi` 中某个 `qemu_irq` 数组的handler，即前面注册的`kvm_pc_gsi_handler` 函数。`kvm_pc_gsi_handler` 函数接收三个参数：第一个参数 `opaque` 指向前述分配的GSIState，第二个参数为 `IRQ` 号，第三个参数为电平信号。

GSIState包含两个成员：`i8259_irq` 和 `ioapic_irq`：

* **`i8259_irq` 对应PIC，**包含16个中断引脚（qemu_irq结构体），handler为 `kvm_pic_set_irq`；
* **`ioapic_irq` 成员对应IOAPIC，**包含24个中断引脚，handler为 `kvm_ioapic_set_irq`。

> 二者分别由 `pc_init1` 函数调用 `kvm_i8259_init` 和 `ioapic_init_gsi` 函数进行初始化。当某条IRQ线有中断信号时，QEMU将根据当前系统使用的中断控制器调用不同的handler。

值得思考的是，当相应的IRQ号小于16时，`kvm_pc_gsi_handler` 便会调用 `GSIState i8259_irq` 中 `qemu_irq` 相应的回调函数，这并非意味着当前系统使用的就是PIC中断控制器。实际上当IRQ号小于16时，KVM会将中断信号同时发送给虚拟PIC和虚拟IOAPIC，虚拟机通过当前真正使用的中断控制器接收该中断信号。相关代码如下：

`qemu-4.1.1/include/hw/i386/pc.h`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311221939521.png" alt="image-20231122193745586" style="zoom:33%;" />

`qemu-4.1.1/hw/i386/pc_piix.c`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311221939120.png" alt="image-20231122193835808" style="zoom:33%;" />

`qemu-4.1.1/hw/i386/kvm/ioapic.c`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311221940782.png" alt="image-20231122194011204" style="zoom:33%;" />

---

因此当虚拟设备调用 `qemu_set_irq` 函数触发中断时，最终会调用 `kvm_pic_set_irq` 函数或`kvm_ioapic_set_irq` 函数，而这两个函数最终都会调用 `kvm_set_irq` 函数通过ioctl将中断信号传入KVM进行处理。代码如下：

`qemu-4.1.1/hw/i386/kvm/i8259.c`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311221945488.png" alt="image-20231122194539574" style="zoom:33%;" />

`qemu-4.1.1/hw/i386/kvm/ioapic.c`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311221946513.png" alt="image-20231122194643008" style="zoom:33%;" />

`qemu-4.1.1/accel/kvm/kvm-all.c`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311221948139.png" alt="image-20231122194819460" style="zoom:33%;" />

### KVM中断路由

KVM中断路由由 `struct kvm_irq_routing_table` 完成，这一结构体功能类似于QEMU中的GSIState，根据传入的 `IRQ` 号调用不同的回调函数：

* 当 `IRQ < 16` 时，KVM会同时将中断信号发送给虚拟PIC和虚拟IOAPIC，虚拟机会通过当前使用的中断控制器接收该中断信号，其余中断信号将被忽略；
* 当 `IRQ > 16` 时，中断信号只会发送给虚拟IOAPIC。

完整的KVM中断路由流程如下图所示：

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311221949814.png" alt="image-20231122194917539" style="zoom: 50%;" />

当中断芯片全部由内核模拟时，**KVM对应的中断路由信息由QEMU设置并传入KVM中，**这一工作由 `pc_init1` 函数调用 `kvm_pc_setup_irq_routing` 函数完成：

* 该函数调用 `kvm_irqchip_add_irq_route` 函数为每个中断控制器的每个中断引脚都添加一条 `kvm_irq_routing_entry`，记录所属中断芯片编号、中断引脚号和对应的GSI号；
  * 最后调用 `kvm_irqchip_commit_routes` 函数将中断路由信息提交给KVM。

值得注意的是，这里IOAPIC的2号中断引脚并不满足前述基础 `GSI + Pin` 映射关系，在QEMU/KVM中，GSI等同于IRQ。相应的，前述 `kvm_set_irq` 传入KVM的其实也是IRQ号。相关代码如下：

`linux-4.19.0/include/uapi/linux/kvm.h`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311222012439.png" alt="image-20231122201145743" style="zoom:33%;" />

`qemu-4.1.1/hw/i386/kvm/ioapic.c`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311222012809.png" alt="image-20231122201238762" style="zoom:33%;" />

----

KVM接收到QEMU传入的中断路由信息后，调用 `kvm_set_irq_routing` 函数构建 `kvm_irq_routing_table`。`kvm_irq_routing_table` 的组织方式与QEMU中的GSIState不同，其定义如下：

`linux-4.19.0/include/linux/kvm_host.h`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311222013988.png" alt="image-20231122201340726" style="zoom:33%;" />

* `chip` 成员是一个二维数组，记录了每个中断芯片中断引脚对应的GSI号。`map` 成员是一个链表数组，`nr_rt_entries` 记录了它的长度。
* 对于每个GSI，`map` 成员都会为其保存一个链表，链表的每一项是 `kvm_kernel_irq_routing_entry`，该结构体类似于上述QEMU中的 `kvm_irq_routing_entry`，记录了GSI在某个中断芯片中对应的引脚号和中断回调函数。
* 对于PIC和IOAPIC，它们的某个引脚可能会映射到同一个GSI，`map` 成员为它们各自维护一条`kvm_kernel_irq_routing_entry` 保存在链表中。
* 当收到某个GSI信号后，KVM将遍历GSI相应的链表，调用每条 `kvm_kernel_irq_routing_entry` 中的set回调函数，将中断信号传递给各个中断芯片。

---

对于传入KVM中的每条 `kvm_irq_routing_entry`，`kvm_set_irq_routing` 函数都会调用 `setup_routing_entry` 函数将其转换为 `kvm_kernel_irq_routing_entry`，设置其set回调函数，并填充`kvm_irq_routing_table` 中的map成员。

PIC路由表项对应的set函数为 `kvm_set_pic_irq`，IOAPIC路由表项对应的set函数为 `kvm_ioapic_set_irq`。相关代码如下：

`linux-4.19.0/virt/kvm/irqchip.c`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311222014196.png" alt="image-20231122201427085" style="zoom: 33%;" />

`linux-4.19.0/arch/x86/kvm/irq_comm.c`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311222015494.png" alt="image-20231122201516866" style="zoom:33%;" />

---

当QEMU调用前述 `kvm_set_irq` 函数陷入KVM时，调用流程如下：

* KVM将调用 `kvm_vm_ioctl_irq_line` 函数进而调用`kvm_set_irq` 函数进行处理；
  * `kvm_set_irq` 函数首先调用 `kvm_irq_map_gsi` 函数，根据传入的GSI从 `kvm_irq_routing_table` 的map成员中取出该GSI对应的所有 `kvm_kernel_irq_routing_entry` 并调用其set函数。

至此，中断信号被传递到KVM模拟的中断芯片中。相关代码如下：

`linux-4.19.0/virt/kvm/irqchip.c`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311222017905.png" alt="image-20231122201606127" style="zoom:33%;" />

---

考虑到后续实验部分主要涉及IOAPIC，这里**以IOAPIC为例介绍中断芯片接收到中断信号后的处理流程。**前面提到IOAPIC对应的set函数为 `kvm_set_ioapic_irq`，该函数首先获取前述虚拟IOAPIC对应的 `kvm_ioapic` 结构体，然后调用 `kvm_ioapic_set_irq` 函数，其代码如下：

`linux-4.19.0/arch/x86/kvm/irq_comm.c`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311222016661.png" alt="image-20231122201647137" style="zoom:33%;" />

`kvm_ioapic_set_irq` 函数最终调用 `ioapic_service` 函数处理该中断请求，它首先以传入的IOAPIC中断引脚号为索引，查询 IOAPIC PRT以获得对应的RTE，PRT记录在前述 `kvm_ioapic` 的 `redirtbl` 成员中，它本质上是一个 `kvm_ioapic_redirect_entry` 数组，每一项记录了**相应的中断引脚RTE**，`kvm_ioapic_redirect_entry` 定义如下：

`linux-4.19.0/arch/x86/kvm/ioapic.h`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311222018148.png" alt="image-20231122201740118" style="zoom:33%;" />

获取相应的RTE后，即 `kvm_ioapic_redirect_entry` 结构体，`ioapic_service` 函数根据RTE创建一个中断请求保存在 `kvm_lapic_irq` 结构体中，并调用 `kvm_irq_delivery_to_apic` 函数将其传递给LAPIC。相关代码如下。

`linux-4.19.0/arch/x86/kvm/ioapic.c`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311222018937.png" alt="image-20231122201840731" style="zoom:33%;" />

---

`kvm_irq_delivery_to_apic` 函数最主要的工作是找到中断请求的目标LAPIC集合，可能包含一个或多个LAPIC，这与前述RTE中的 `dest_mode` 和 `dest_id` 成员密切相关，这里不再详述该流程。

对于找到的每一个目标LAPIC，`kvm_irq_delivery_to_apic` 函数都会调用 `kvm_apic_set_irq` 函数将中断请求发送给它：

* `kvm_apic_set_irq` 函数首先从 `vcpu->arch.apic` 中获得虚拟LAPIC对应的 `kvm_lapic` 结构体；
  * 然后调用`__apic_accept_irq` 函数接收该中断请求。

至此，中断信号传递给LAPIC，与此同时 **`GSI/IRQ` 号也被转换为相应的中断向量号。**相关代码如下：

`linux-4.19.0/arch/x86/kvm/lapic.c`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311222019934.png" alt="image-20231122201932698" style="zoom:33%;" />

中断传送方式不同，即RTE中的 `delivery_mode` 不同，`__apic_accept_irq` 的处理也不同。以Fixed模式为例，其对应的宏为 `APIC_DM_FIXED`。代码如下：

`linux-4.19.0/arch/x86/kvm/lapic.c`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311222020085.png" alt="image-20231122202014299" style="zoom:33%;" />

* LAPIC在设置 `IRR` 前，首先要设置 `TMR`（Trigger Mode Register，触发模式寄存器）。`TMR` 与 `IRR`、`ISR` 类似，都为256位，每一位与一个中断向量号相对应。对于边沿触发的中断，`TMR` 中对应位置0，对于水平触发的中断，`TMR` 中对应位置1。
* 然后 `_apic_accept_irq` 函数检查是否通过发布-中断机制递交虚拟中断？
  * 若是，则通过 `kvm_x86_ops` 的`deliver_posted_interrupt` 成员调用vmx.c中的 `vmx_deliver_posted_interrupt` 函数；
  * 否则先调用 `kvm_lapic_set_irr` 函数设置虚拟 `LAPIC IRR` 的值。`kvm_lapic_set_irr` 函数将通过前述 `kvm_lapic` 的regs成员设置虚拟 `IRR` 并将其 `kvm_lapic irr_pending` 成员设为true；
* 接下来，`__apic_set_irq` 函数调用 `kvm_make_request` 函数发起一个KVM_REQ_EVENT类型的请求，并调用 `kvm_vcpu_kick` 函数通知vCPU。
  * 如果当前vCPU线程正处于睡眠状态，`kvm_vcpu_kick` 函数将会调用 `kvm_vcpu_wake_up` 函数唤醒vCPU线程；
  * 如果vCPU线程当前正在运行，`kvm_vcpu_kick` 函数将会调用 `smp_send_reschedule` 函数给其所在物理CPU发送一个IPI使vCPU发生VM-Exit。
  * 无论是哪种情况，vCPU最终都会调用前述 `vcpu_enter_guest` 函数重新进入非根模式，而在此之前，`vcpu_enter_guest` 函数会调用 `kvm_check_request` 函数发现当前vCPU有待注入的中断请求从而调用`inject_pending_event` 函数注入该虚拟中断。

相关代码如下：

`linux-4.19.0/arch/x86/kvm/lapic.h`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311222021111.png" alt="image-20231122202140086" style="zoom:33%;" />

`linux-4.19.0/virt/kvm/kvm_main.c`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311222022068.png" alt="image-20231122202208218" style="zoom:33%;" />

至此，KVM中断路由结束，4.4节将会介绍**虚拟中断的注入流程。**

## 4.4 虚拟中断注入

前文提到虚拟中断注入由 `vcpu_enter_guest` 函数调用 `inject_pending_event` 函数完成。实际上，`inject_pending_event` 函数不仅可以注入中断事件，还能注入异常事件，但本节只关心中断注入的部分。

`inject_pending_event` 函数首先会检查 `vcpu->arch.interrupt.injected` 判断系统中是否已经存在待注入的虚拟中断。如在注入上一个虚拟中断过程中发生了VM-Exit，硬件将会把当前正在注入的中断信息保存在VMCS中断向量号信息 (IDT-Vectoring Information) 字段中，然后KVM将会读取该信息并调用 `kvm_queue_interrupt` 函数将中断信息保存到 `vcpu->arch.interrupt` 成员中，并将 `vcpu->arch.interrupt.injected` 设为true便于在下一次调用 `vcpu_enter_guest` 函数时重新注入该事件，这一工作由前述 `vmx_vcpu_run` 函数调用`vmx_complete_interrupts` 函数完成。相关代码如下：

`linux-4.19.0/arch/x86/kvm/vmx.c`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311222024091.png" alt="image-20231122202404486" style="zoom:33%;" />

若当前系统中不存在待注入中断，`inject_pending_event` 函数将会调用 `kvm_cpu_has_injectable_intr` 函数检查虚拟LAPIC中是否有待处理的中断。若有，则同样调用 `kvm_queue_interrupt` 函数保存该中断信息，并通过`kvm_x86_ops` 的set_irq成员调用vmx.c中 `vmx_inject_irq` 函数注入中断。相关代码如下：

`linux-4.19.0/arch/x86/kvm/x86.h`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311222025985.png" alt="image-20231122202536391" style="zoom:33%;" />

`linux-4.19.0/arch/x86/kvm/x86.c`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311222026642.png" alt="image-20231122202624438" style="zoom:33%;" />

---

`vmx_inject_irq` 函数的主要工作是根据 `vcpu.arch` 中保存的中断信息设置前述VM-Entry控制域中断信息字段。在VM-Entry时，CPU会检查该字段发现有待处理的中断，并用中断向量号索引IDT以执行相应的处理函数。相关代码如下：

`linux-4.19.0/arch/x86/kvm/x86.h`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311222027648.png" alt="image-20231122202726079" style="zoom:33%;" />

`linux-4.19.0/arch/x86/kvm/vmx.c`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311222030535.png" alt="image-20231122203008579" style="zoom:33%;" />

至此，**虚拟中断注入完成。**4.5节将以e1000网卡中断为例，介绍中断虚拟化的完整流程。

## 4.5 实验: e1000网卡中断虚拟化

前几节介绍了当中断芯片全部由KVM模拟时，外部设备中断传送的完整流程。本节将以e1000网卡为例，通过GDB调试工具回顾前述流程。

### 实验概述

为了使用GDB调试虚拟中断在KVM模块中的传送流程，本次实验需要在嵌套虚拟化的环境下进行，物理机和虚拟机使用的内核版本均为4.19.0。本节使用前述QEMU提供的 `-s` 和 `-S` 选项启动一个虚拟机。启动命令如下：

`Physical Machine Terminal 1`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311231719972.png" alt="image-20231123171810813" style="zoom:33%;" />

在终端2启动GDB加载Linux内核调试信息并连接至1234端口，然后开始运行虚拟机。运行命令如下：

`Physical Machine Terminal 2`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311231719309.png" alt="image-20231123171859047" style="zoom:33%;" />

为了加载KVM模块的调试信息，读者需要在虚拟机中通过 `/sys/module/module_name/sections` 查看 `kvm.ko` 和 `kvm-intel.ko` 模块所在的GPA，并在GDB中手动引入KVM模块的调试信息。运行命令如下。

`Virtual Machine Terminal 1`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311231719366.png" alt="image-20231123171941476" style="zoom:33%;" />

`Physical Machine Terminal 2`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311231720645.png" alt="image-20231123172009276" style="zoom:33%;" />

接着**在虚拟机中启动嵌套虚拟机并运行，**`-monitor` 选项指定了QEMU监视器 (QEMU monitor) 的运行端口为 `4444`。读者可以另外启动一个终端连接至QEMU监视器。QEMU监视器提供了各种命令用于查看虚拟机的当前状态。这里可以通过 `info qtree` 查看当前虚拟机的架构。可以发现虚拟机使用的中断控制器为APIC。虚拟IOAPIC直接挂载在 `main-system-bus` 上，而e1000网卡挂载名为 `pic.0` 的PCI总线上。启动命令如下：

`Virtual Machine Terminal 2`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311231721921.png" alt="image-20231123172038266" style="zoom:33%;" />

`Virtual Machine Terminal 3`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311231721711.png" alt="image-20231123172104898" style="zoom:33%;" />

---

在嵌套虚拟机中启动一个终端并执行 `lspci-v` 指令，可以查看当前虚拟机中的PCI设备，e1000网卡具体信息如下。

`Nested Virtual Machine Terminal 1`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311231722716.png" alt="image-20231123172147932" style="zoom:33%;" />

上述信息表明e1000网卡的BDF为 `00:03.0`，即24，而e1000网卡使用的中断IRQ号为11，介绍中断传递时提到在QEMU/KVM中GSI与IRQ等价，除了0号中断引脚外，其余IOAPIC引脚与GSI满足 `GSI = GSI_base + Pin` 映射关系，故e1000网卡对应的中断引脚号为11。然后使用QEMU监视器输入 `info pic` 查看虚拟机IOAPIC信息。

`Virtual Machine Terminal 3`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311231722226.png" alt="image-20231123172231931" style="zoom:33%;" />

以上输出表明虚拟IOAPIC的11号中断引脚对应的中断向量号为38，即e1000网卡使用 `38` 号中断向量。下面将通过GDB查看e1000网卡中断传送过程中的关键函数调用以及中断信息。

### e1000网卡中断传送流程

网卡在收发包时都会产生中断，而对于e1000网卡，无论是收包中断还是发包中断，最后都会调用 `set_interrupt_cause` 函数。读者可以通过前述 `Virtual Machine Terminal 2` 中运行的GDB在 `set_interrupt_cause` 函数处设置断点并继续运行该程序。当触发该断点后，打印出e1000网卡的设备号为24，与 `lspci-v` 指令结果相符。

`Virtual Machine Terminal 2`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311231725989.png" alt="image-20231123172415478" style="zoom:33%;" />

`set_interrupt_cause` 函数最终会调用PCI设备共用的中断接口 `pci_set_irq` 函数触发中断，为了区别e1000网卡与其他PCI设备，我们可以使用GDB条件断点，使得只有设备号为24时才命中 `pci_set_irq` 中的断点。终端输出表明e1000网卡使用的中断引脚号（`intx` 变量）为0，即e1000网卡使用 `INTA#` 中断引脚。

`Virtual Machine Terminal 2`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311231725685.png" alt="image-20231123172450445" style="zoom:33%;" />

如前所述，`pci_irq_handler` 函数最终会调用 `pci_change_irq_level` 函数获得PCI设备中断引脚所连接的PCI总线中断请求线。`pci_change_irq_level` 函数通过调用所属总线的 `map_irq` 回调函数 `pci_slot_get_irq` 完成上述转换。在该行设置断点并打印出对应的PCI链接设备号（对应 `irq_num`）为2，故e1000网卡的 `INTA#` 中断引脚连接至 `LNKC` 中断请求线。

`Virtual Machine Terminal 2`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311231726018.png" alt="image-20231123172606710" style="zoom:33%;" />

而 `pci_change_irq_level` 函数将会调用总线成员的set_irq回调函数 `piix3_set_irq`，进而调用 `piix3_set_irq_level` 函数。该函数通过PIIX3设备PCI配置空间中的 `PIRQRC[A:D]` 寄存器**获取PCI总线某条中断请求线对应的IOAPIC IRQ线。**在该函数中断点打印e1000网卡对应的IRQ线(`pci_irq`)，其值为11。

`Virtual Machine Terminal 2`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311231727713.png" alt="image-20231123172659958" style="zoom:33%;" />

`piix3_set_irq_level` 函数获得PCI设备对应的IRQ线后会调用 `piix3_set_irq_pic` 函数，该函数进而调用 `qemu_set_irq` 函数经由4.4节介绍的QEMU中断路由过程后，最终调用 `kvm_set_irq` 函数将中断信号传递至KVM模拟的中断芯片。调用GDB `backtrace` 命令打印出当前函数调用栈帧，与QEMU中断路由流程相符。如下：

`Virtual Machine Terminal 2`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311231727828.png" alt="image-20231123172742648" style="zoom:33%;" />

---

**中断信号传入KVM后，**经由4.4节介绍的内核中断路由表，将中断信号发送至所有可能的虚拟中断控制器。对于本实验来说，虚拟机使用的中断控制器为IOAPIC，对应的回调函数为 `kvm_set_ioapic_irq`，该函数将调用 `kvm_ioapic_set_irq` 函数处理指定中断引脚传来的中断信号。通过GDB在该函数处设置断点，可以发现传入 `kvm_ioapic_set_irq` 函数的中断引脚号为11，即e1000中断对应的中断引脚号为11。

`Physical Machine Terminal 1`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311231728332.png" alt="image-20231123172820241" style="zoom:33%;" />

如前所述，`kvm_ioapic_set_irq` 函数最终调用 `ioapic_service` 函数处理指定引脚的中断请求。`ioapic_service` 函数根据传入的中断引脚号在 `PRT(ioapic->redirtbl)` 中找到对应的RTE并格式化出一条中断消息 `irqe` 发送给所有的目标LAPIC。

`irqe` 中包含供CPU最终使用的中断向量号。在 `ioapic_service` 函数中设置断点打印中断消息 `irqe`，可以发现e1000网卡对应的中断向量号为38，触发方式为水平触发 (trig_mode=1)，与通过QEMU监视器执行 `info pic` 命令得到的信息完全一致。

`Physical Machine Terminal 1`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311231729013.png" alt="image-20231123172852451" style="zoom:33%;" />

后续流程为：

1. 虚拟LAPIC收到中断消息后，将设置 `IRR` 并调用 `vcpu_kick` 函数通知vCPU；
2. 当vCPU再次调用 `vcpu_enter_guest` 函数准备进入非根模式时，发现当前有待注入的虚拟中断。最终 `vcpu_enter_guest` 函数会调用 `vmx_inject_irq` 函数设置VMCS VM-Entry控制域中的中断信息字段；
3. 当虚拟机恢复运行时，CPU会检查该字段发现有待处理的e1000网卡中断，则CPU将执行相应的中断服务例程。

至此，e1000网卡产生的虚拟中断处理流程完成。

---

本节通过GDB展示了e1000网卡虚拟中断处理流程，着重展示了**PCI设备中断引脚号与IRQ号的映射**以及**IRQ号与中断向量号的映射**关系。

# 5 //TODO: GiantVM CPU虚拟化
