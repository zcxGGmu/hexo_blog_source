---
title: Linux进程调度机制
date: 2025-11-11 10:10:24
tags: 进程调度
description: 内核抢占在一定程度上减少了对某种事件的响应延迟，这也是内核抢占被引入的目的
---



# 0 资料

[【原创】（一）Linux进程调度器-基础 - LoyenWang - 博客园 (cnblogs.com)](https://www.cnblogs.com/LoyenWang/p/12249106.html)

[【原创】（三）Linux进程调度器-进程切换 - LoyenWang - 博客园 (cnblogs.com)](https://www.cnblogs.com/LoyenWang/p/12386281.html)

[进程调度 - 知乎 (zhihu.com)](https://www.zhihu.com/column/c_1523702194322370560)

[内核抢占机制(preempt)-CSDN博客](https://blog.csdn.net/u014426028/article/details/106927601#:~:text=内核抢占指用户程序在执行系统调用期间可以被抢占，该进程暂时挂起，使新唤醒的高优先级进程能够运行。,这种抢占并非可以在内核中任意位置都能安全进行，比如在临界区中的代码就不能发生抢占。 临界区是指同一时间内不可以有超过一个进程在其中执行的指令序列。)

[sched.h - include/linux/sched.h - Linux source code (v6.7-rc7) - Bootlin](https://elixir.bootlin.com/linux/v6.7-rc7/source/include/linux/sched.h#L746)



# 1 Linux进程/线程的抽象模型

## 1.1 进程/线程的统一抽象: task_struct

### 关于task_struct

任何教科书上都会这样写：进程是资源分配的最小单位，而线程是CPU调度的的最小单位。

* **对于进程而言：**进程不仅包括可执行程序的代码段，还包括一系列的资源，比如：打开的文件、内存、CPU时间、信号量、多个执行线程流等等。进程不仅提供了一个线程资源容器，还负责线程调度工作；
* **对于线程而言：**线程是程序的执行实体，相比于进程，线程更真实地呈现在CPU上，CPU上各寄存器保存的值支撑了程序控制流的执行。其中 `thread_struct` 保存的是最核心的CPU寄存器内容，其和特定架构相关；

但在Linux内核中，两者均使用 `task_struct` 进行抽象，给人一种进程和线程相同的错觉。在之前学习的 `rCore` 实现中，进程/线程模型有着比较清晰的数据结构上的层次感：

```rust
pub struct ProcessControlBlock {
    // immutable
    pub pid: PidHandle,
    // mutable
    inner: UPSafeCell<ProcessControlBlockInner>,
}

pub struct ProcessControlBlockInner {
    pub is_zombie: bool,
    pub memory_set: MemorySet,
    pub parent: Option<Weak<ProcessControlBlock>>,
    pub children: Vec<Arc<ProcessControlBlock>>,
    pub exit_code: i32,
    pub fd_table: Vec<Option<Arc<dyn File + Send + Sync>>>,
    pub signals: SignalFlags,
    pub tasks: Vec<Option<Arc<TaskControlBlock>>>,
    pub task_res_allocator: RecycleAllocator,
    ... // 其他同步互斥相关资源
}
```

可以看到，进程控制块中通过 `tasks` 保存着全部线程资源。

但 Linux 进程/线程模型与之有很大的区别。那么，进程和线程在 `task_struct` 结构体中是否有标识上的不同？实际上，在struct task_struct中并没有明确的标识（枚举类型），区分该task是线程还是进程，不过**可以通过pid和tgid简单判断当前task是哪种类型。**

```c
struct task_struct {
//...
    pid_t pid;
    pid_t tgid;
//...
    struct *group_leader;
}
```

* **pid**用于标识不同进程和线程；
* **tgid**用于标识线程组id，在同一进程中的所有线程具有同一tgid。tgid值等于进程第一个线程（主线程）的pid值。接着以CLONE_THREAD来调用clone建立的线程，都具有同样的tgid。也就是说，对于没有创建线程的进程(只包含一个主线程)来说，这个 pid 就是进程的 PID，tgid 和 pid 是相同的。
* **group_leader** 线程组中的主线程的task_struct指针。

>那么除了tgid和group_leadr是进程/线程的区别外，还有什么其他的区别么？
>
>进程还是线程的创建都是由父进程/父线程调用系统调用接口实现的。创建的主要工作实际上就是创建task_strcut结构体，并将该对象添加至工作队列中去。而线程和进程在创建时，通过CLONE_THREAD flag的不同，而选择不同的方式共享父进程/父线程的资源，从而造成在用户空间所看到的进程和线程的不同。

### 从进程/线程创建角度来看task_struct

无论以何种方式创建线程/进程在Linux kernel最终都是调用do_fork接口（定义在kernel/fork.c）

其函数原型为：

```cpp
long do_fork(unsigned long clone_flags,
    unsigned long stack_start,
    unsigned long stack_size,
    int __user *parent_tidptr,
    int __user *child_tidptr)
```

- clone_flags是一个标志集合，用来指定控制复制过程的一些属性。最低字节指定了在子进程终止时被发给父进程的信号号码。其余的高位字节保存了各种常数。
- stack_start是用户状态下栈的起始地址。
- stack_size是用户状态下栈的大小。
- arent_tidptr和child_tidptr是指向用户空间中地址的两个指针，分别指向父子进程的PID。

do_fork的代码流程图如下所示：

![img](https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202312261605388.png)

上面流程特别判断了是否是vfork，该接口是vfork接口call下来，在子进程没有执行完前，父进程处于阻塞态。一般用于子进程直接调用execv时使用。因为子进程不需要copy父进程的资源从而减少do_fork时的消耗，不过由于fork增加了写时复制机制，vfork也很少使用。这些不是这篇介绍的重点。

那么拿掉vfork的过程，do_fork主要做了三件事：

1. **copy_process**
2. **确定PID**
3. **wake_up_new_task**

wake_up_new_task即是将新创建的线程/进程添加至调度程序的队列中。do_fork主要的一部分工作集中在copy_process中，线程与进程之间的区别也是在该接口中体现，接口的代码流程图如下所示：

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202312261605186.png" alt="img" style="zoom:67%;" />

当上层以pthread_create接口call到kernel时，clone_flag是有CLONE_PTHREAD标识。但CLONE_PTHREAD标识只在最后一个步骤（设置各个ID、进程关系）时体现：（current为当前进程/线程的task_struct结构体 ，p为新创建的结构体对象）：

```c
if (clone_flags & CLONE_THREAD) {
    p->group_leader = current->group_leader;
    p->tgid = current->tgid;
} else {
    p->group_leader = p;
    p->tgid = p->pid;
}
```

那么，以我们的理解来看，线程会共享信号、共享虚拟地址空间...又以什么体现呢？去glibc查询了pthread_create的实现，当call到kernel时的clone_flag如下：

```c
const int clone_flags = (CLONE_VM | CLONE_FS | CLONE_FILES | CLONE_SYSVSEM
			   | CLONE_SIGHAND | CLONE_THREAD
			   | CLONE_SETTLS | CLONE_PARENT_SETTID
			   | CLONE_CHILD_CLEARTID
			   | 0);
```

所以在创建线程时，clone_flags有其他许多项共同构成，才让我们看出来最终线程与进程间的不同。这些flag主要体现在【分享/复制进程各个部分中】步骤。

这些CLONE_abc的使用方法相似。在这些形如copy_abc的接口中，通过判断该flag标识，决定对内核子系统资源是与父进程/线程公用还是新创建出来。可参考下图。

![img](https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202312261601074.png)

一开始父进程和子进程对于res_abc指向同一个内容（通过dup_task_struct接口实现，子进程完全copy父进程），然后经过copy_abc程序，当有CLONE_abc标识时，父进程会共享资源，同时res_abc的引用计数+1，当!CLONE_abc时，会创建一个res_abc的副本。

又去glibc查询了fork的clone_flags：

```c
CLONE_CHILD_SETTID | CLONE_CHILD_CLEARTID | SIGCHLD;
```

那么线程创建就比进程多CLONE_VM | CLONE_FS | CLONE_FILES | CLONE_SYSVSEM | CLONE_SIGHAND | CLONE_THREAD，从这些宏的字面意义上看，子线程会与父线程共享虚拟地址空间、文件、信号等。

每个flag的实现原理基本一致，上文已讨论过，这里仅对具体的哪些资源造成了影响进行分析。

* CLONE_VM 为虚拟地址空间：所以子线程会共享父线程的虚拟地址空间（通过struct mm_struct *mm指向实例描述）,active_mm当用户线程切换至内核线程时使用，这里不详述。
* CLONE_FS struct fs_struct *fs 进程当前目录及工作目录；
* CLONE_SIGHAND struct sighand_struct *sighand 信号及信号处理函数

### 进程/线程状态

`task_struct`结构很复杂，下边只针对与调度相关的某些字段进行介绍：

```c
struct task_struct {
    /* ... */
    
    /* 进程状态 */
    volatile long			state;

    /* 调度优先级相关，策略相关 */
	int				prio;
	int				static_prio;
	int				normal_prio;
	unsigned int			rt_priority;
    unsigned int			policy;
    
    /* 调度类，调度实体相关，任务组相关等 */
    const struct sched_class	*sched_class;
	struct sched_entity		se;
	struct sched_rt_entity		rt;
#ifdef CONFIG_CGROUP_SCHED
	struct task_group		*sched_task_group;
#endif
	struct sched_dl_entity		dl;
    
    /* 进程之间的关系相关 */
    	/* Real parent process: */
	struct task_struct __rcu	*real_parent;

	/* Recipient of SIGCHLD, wait4() reports: */
	struct task_struct __rcu	*parent;

	/*
	 * Children/sibling form the list of natural children:
	 */
	struct list_head		children;
	struct list_head		sibling;
	struct task_struct		*group_leader;
    
    /* CPU-specific state of this task: */
	struct thread_struct		thread;
}
```

---

下图中左侧为操作系统中通俗的进程三状态模型，右侧为Linux对应的进程状态切换。每一个标志描述了进程的当前状态，这些状态都是互斥的：

![img](https://img2018.cnblogs.com/blog/1771657/202002/1771657-20200201170358218-1930669459.png)

> 注意：Linux中的`就绪态`和`运行态`对应的都是`TASK_RUNNING`标志位，`就绪态`表示进程正处在队列中，尚未被调度；`运行态`则表示进程正在CPU上运行；

内核中主要的状态字段定义如下：

```c
/* Used in tsk->state: */
#define TASK_RUNNING			0x0000
#define TASK_INTERRUPTIBLE		0x0001
#define TASK_UNINTERRUPTIBLE		0x0002

//...
```

## 1.2 调度器: scheduler

> 后文中将不再区分进程/线程，而采用统一的 “进程” 来描述。因为Linux内核使用 `task_struct` 对两者在数据结构上进行了统一，但两者在引用资源的方式上有所不同。

所谓调度，就是按照某种调度的算法，从进程的就绪队列中选取进程分配CPU，主要是协调对CPU等的资源使用。进程调度的目标是最大限度利用CPU时间。内核默认提供了5个调度器，Linux内核使用 `struct sched_class` 来对调度器进行抽象：

![img](https://img2018.cnblogs.com/blog/1771657/202002/1771657-20200201170552968-1185293225.png)

1. `Stop调度器， stop_sched_class`：优先级最高的调度类，可以抢占其他所有进程，不能被其他进程抢占；
2. `Deadline调度器， dl_sched_class`：使用红黑树，把进程按照绝对截止期限进行排序，选择最小进程进行调度运行；
3. `RT调度器， rt_sched_class`：实时调度器，为每个优先级维护一个队列；
4. `CFS调度器， cfs_sched_class`：完全公平调度器，采用完全公平调度算法，引入虚拟运行时间概念；
5. `IDLE-Task调度器， idle_sched_class`：空闲调度器，每个CPU都会有一个idle线程，当没有其他进程可以调度时，调度运行idle线程；

Linux内核提供了一些调度策略供用户程序来选择调度器，其中 `Stop调度器` 和 `IDLE-Task调度器` ，仅由内核使用，用户无法进行选择：

- `SCHED_DEADLINE`：限期进程调度策略，使task选择 `Deadline调度器` 来调度运行；
- `SCHED_RR`：实时进程调度策略，时间片轮转，进程用完时间片后加入优先级对应运行队列的尾部，把CPU让给同优先级的其他进程；
- `SCHED_FIFO`：实时进程调度策略，先进先出调度没有时间片，没有更高优先级的情况下，只能等待主动让出CPU；
- `SCHED_NORMAL`：普通进程调度策略，使task选择 `CFS调度器` 来调度运行；
- `SCHED_BATCH`：普通进程调度策略，批量处理，使task选择 `CFS调度器 `来调度运行；
- `SCHED_IDLE`：普通进程调度策略，使task以最低优先级选择 `CFS调度器` 来调度运行；

## 1.3 运行队列: runqueue

每个CPU都有一个运行队列，每个调度器都作用于运行队列。分配给CPU的task，作为调度实体加入到运行队列中。task首次运行时，如果可能，尽量将它加入到父task所在的运行队列中（分配给相同的CPU，缓存affinity会更高，性能会有改善）；

![img](https://img2018.cnblogs.com/blog/1771657/202002/1771657-20200201170629353-2136884130.png)



Linux内核使用 `struct rq` 结构来描述运行队列，关键字段如下：

```c
/*
 * This is the main, per-CPU runqueue data structure.
 *
 * Locking rule: those places that want to lock multiple runqueues
 * (such as the load balancing or the thread migration code), lock
 * acquire operations must be ordered by ascending &runqueue.
 */
struct rq {
	/* runqueue lock: */
	raw_spinlock_t lock;

	/*
	 * nr_running and cpu_load should be in the same cacheline because
	 * remote CPUs use both these fields when doing load calculation.
	 */
	unsigned int nr_running;
    
    /* 三个调度队列：CFS调度，RT调度，DL调度 */
	struct cfs_rq cfs;
	struct rt_rq rt;
	struct dl_rq dl;

    /* stop指向迁移内核线程， idle指向空闲内核线程 */
    struct task_struct *curr, *idle, *stop;
    
    /* ... */
}    
```

## 1.4 任务分组: task_group

利用任务分组的机制，可以设置或限制任务组对CPU的利用率，比如将某些任务限制在某个区间内，从而不去影响其他任务的执行效率。引入`task_group` 后，调度器的调度对象不仅仅是进程了，Linux内核抽象出了 `sched_entity/sched_rt_entity/sched_dl_entity` 描述调度实体，调度实体可以是进程或 `task_group`。

![img](https://img2018.cnblogs.com/blog/1771657/202002/1771657-20200201170652689-605326625.png)

Linux内核使用数据结构 `struct task_group ` 来描述任务组，任务组在每个CPU上都会维护一个 `CFS调度实体、CFS运行队列，RT调度实体，RT运行队列` 。

```c
/* task group related information */
struct task_group {
    /* ... */

    /* 为每个CPU都分配一个CFS调度实体和CFS运行队列 */
#ifdef CONFIG_FAIR_GROUP_SCHED
	/* schedulable entities of this group on each cpu */
	struct sched_entity **se;
	/* runqueue "owned" by this group on each cpu */
	struct cfs_rq **cfs_rq;
	unsigned long shares;
#endif

    /* 为每个CPU都分配一个RT调度实体和RT运行队列 */
#ifdef CONFIG_RT_GROUP_SCHED
	struct sched_rt_entity **rt_se;
	struct rt_rq **rt_rq;

	struct rt_bandwidth rt_bandwidth;
#endif

    /* task_group之间的组织关系 */
	struct rcu_head rcu;
	struct list_head list;

	struct task_group *parent;
	struct list_head siblings;
	struct list_head children;

    /* ... */
};
```

## 1.5 调度程序: schedule/...



# 2 进程/线程调度

根据进程是否自愿放弃cpu，调度方式可分为**主动调度和抢占调度**两类，它们的区别如下：

* **主动调度：**进程需要等待IO、锁等资源，而主动放弃cpu；

* **抢占调度：**进程由于时间片用完，或被优先级更高的进程抢占，而被强制剥夺cpu；

内核中那些由于等待资源而需要阻塞的场景，会直接调用 `schedule()` 执行实际的调度流程。而其它需要调度的场景一般都只是设置一个TIF_NEED_RESCHED标志，并在**下一个抢占点到来时**才执行实际的抢占操作。

在支持内核抢占之前，只有在系统调用返回用户态之前，或者中断返回用户态之前才能执行抢占操作。而在支持内核抢占以后，即使在内核执行路径中也允许抢占，因此内核支持了更多的抢占点。

`task_struct` 结构体的首个字段放置的正是 `struct thread_info`，`struct thread_info`结构体中 `flag` 字段就可用于设置`TIF_NEED_RESCHED`，此外该结构体中的 `preempt_count` 也与抢占相关。

```c
struct task_struct {
#ifdef CONFIG_THREAD_INFO_IN_TASK
	/*
	 * For reasons of header soup (see current_thread_info()), this
	 * must be the first element of task_struct.
	 */
	struct thread_info		thread_info;
#endif
        ...
}

/*
 * low level task data that entry.S needs immediate access to.
 */
struct thread_info {
	unsigned long		flags;		/* low level flags */
	mm_segment_t		addr_limit;	/* address limit */
#ifdef CONFIG_ARM64_SW_TTBR0_PAN
	u64			ttbr0;		/* saved TTBR0_EL1 */
#endif
	int			preempt_count;	/* 0 => preemptable, <0 => bug */
};

#include <asm/current.h>
#define current_thread_info() ((struct thread_info *)current)   //通过该宏可以直接获取thread_info的信息
#endif
```

## 2.1 用户抢占

> 在抢占触发前，进程处于用户态，触发条件通常为：执行系统调用或时钟中断。

内核即将返回用户空间的时候，如果need resched标志被设置，会导致schedule()被调用，此时就会发生用户抢占。在内核返回用户空间的时候，它知道自己是安全的。所以，内核无论是在从中断处理程序还是在系统调用后返回，都会检查need resched标志。如果它被设置了，那么，内核会选择一个其他 (更合适的) 进程投入运行。

### 抢占触发点/条件设置

看看具体哪些函数过程中，设置了 `TIF_NEED_RESCHED` 标志吧：

![img](https://img2018.cnblogs.com/blog/1771657/202002/1771657-20200229212204811-1734127314.png)

### 抢占执行点

具体来说，在用户态执行抢占在以下几种情况：

- 异常处理 (系统调用) 后返回到用户态；
- 中断处理后返回到用户态；

![img](https://img2018.cnblogs.com/blog/1771657/202002/1771657-20200229212235516-1038164597.png)

- 用户程序在执行过程中，遇到异常或中断后，将会跳到 `ENTRY(vectors)` 向量表处开始执行；
- 返回用户空间时进行标志位判断，设置了 `TIF_NEED_RESCHED` 则需要进行调度切换；

## 2.2 内核抢占

> 当进程位于内核空间时，有一个更高优先级的任务出现时，如果当前内核允许抢占，则可以将当前任务挂起，执行优先级更高的进程。内核抢占对实时性有帮助。

***为什么Linux kernel需要内核抢占？***

内核抢占在一定程度上减少了对某种事件的响应延迟，这也是内核抢占被引入的目的。之前的内核中，除了显示调用系统调度器的某些点，内核其他地方是不允许中重新调度的，如果内核在做一些比较复杂的工作，就会造成某些急于处理的事得不到及时响应。针对内核抢占其实本质上也是对当前进程而言（不知道这么描述是否合适），因为内核是为用户程序提供服务，换言之，其本身不会主动的去执行某个动作。这里内核抢占，重点在于用户程序请求内核服务时，CPU切换到内核态在执行某个系统服务期间，被抢占。

当然，即使支持内核抢占，也不是什么时候都可以的，还是要考虑对临界区的保护。类似于多处理器架构，如果进程A陷入到内核模式访问某个临界资源，而在访问期间，进程B也要访问临界区，如果这种抢占被允许，那么就发生了临界区被重入。所以，在访问临界资源时需要禁止内核抢占，在出临界区则要开启内核抢占。

### preempt_count

禁止内核抢占的情况列出如下：

（1）内核执行***\*中断处理例程\****时不允许内核抢占，中断返回时再执行内核抢占。
（2）当内核执行***\*软中断或tasklet\****时，禁止内核抢占，软中断返回时再执行内核抢占。 
（3）在**临界区**禁止内核抢占，临界区保护函数通过抢占计数宏控制抢占，计数大于0，表示禁止内核抢占。

为保证Linux内核在以上情况下不会被抢占，抢占式内核使用了一个变量preempt_ count，称为内核抢占锁。这一变量被设置在进程的PCB结构task_struct中。每当内核要进入以上几种状态时，变量preempt_count就加1，指示内核不允许抢占。每当内核从以上几种状态退出时，变量preempt_ count就减1，同时进行可抢占的判断与调度。抢占式Linux内核的修改主要有两点：

* **对中断的入口代码和返回代码进行修改。**在中断的入口内核抢占锁preempt_count加1，以禁止内核抢占；在中断的返回处，内核抢占锁preempt_count减1，使内核有可能被抢占。
* **重新定义了自旋锁、读、写锁，**在锁操作时增加了对preempt count变量的操作。在对这些锁进行加锁操作时preemptcount变量加1，以禁止内核抢占；在释放锁时preemptcount变量减1，并在内核的抢占条件满足且需要重新调度时进行抢占调度。

---

[【进程】preempt_count解析 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/673428998)

Linux内核中使用 `struct thread_info` 中的 `preempt_count` 字段来控制内核抢占：

- `preempt_count` 的低8位用于控制抢占，当大于0时表示不可抢占，等于0表示可抢占；
  - `preempt_enable()` 会将 `preempt_count` 值减1，并判断是否需要进行调度，在条件满足时进行切换；
  - `preempt_disable()` 会将 `preempt_count` 值加1；

此外，`preemt_count` 字段还用于判断进程处于各类上下文以及开关控制等，如图：

![img](https://img2018.cnblogs.com/blog/1771657/202002/1771657-20200229212328079-449160131.png)

### 抢占触发点/条件设置

- 在内核中抢占触发点，也是设置 `struct thread_info` 的 `flag` 字段，设置 `TIF_NEED_RESCHED` 表明需要请求重新调度；
- 抢占触发点的几种情况，在用户抢占中已经分析过，不管是用户抢占还是内核抢占，触发点都是一致的；

### 抢占执行点

![img](https://img2018.cnblogs.com/blog/1771657/202002/1771657-20200229212309761-334760914.png)

总体而言，内核抢占执行点可以归属于两大类：

- 中断执行完毕后进行抢占调度；
- 主动调用`preemp_enable`或`schedule`等接口的地方进行抢占调度；

## 2.3 进程上下文切换

进程上下文切换的入口就是 `__schedule()`，分析也围绕这函数展开。

![img](https://img2018.cnblogs.com/blog/1771657/202002/1771657-20200229212346352-1357028476.png)

主要的逻辑：

- 根据CPU获取运行队列，进而得到运行队列当前的`task`，也就是切换前的`prev`;
- 根据`prev`的状态进行处理，比如`pending`信号的处理等，如果该任务是一个`worker线程`还需要将其睡眠，并唤醒同CPU上的另一个`worker线程`;
- 根据调度类来选择需要切换过去的下一个`task`，也就是`next`；
- `context_switch`完成进程的切换；

---

`context_switch()` 的调用分析如下：

![img](https://img2018.cnblogs.com/blog/1771657/202002/1771657-20200229212409378-167088356.png)

核心的逻辑有两部分：

- `进程的地址空间切换`：切换的时候要判断切入的进程是否为内核线程
  - 所有的用户进程都共用一个内核地址空间，而拥有不同的用户地址空间；
  - 内核线程本身没有用户地址空间。在进程在切换的过程中就需要对这些因素来考虑，涉及到页表的切换，以及`cache/tlb`的刷新等操作。
- `寄存器的切换`：包括CPU的通用寄存器切换、浮点寄存器切换，以及ARM处理器相关的其他一些寄存器的切换；

实际上进程的切换，带来的开销不仅是页表切换和硬件上下文的切换，还包含了`Cache/TLB`刷新后带来的`miss`的开销。

