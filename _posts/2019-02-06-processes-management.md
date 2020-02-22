---
title: 6. 进程管理
---

操作系统功能包括进程管理、内存管理、文件系统、设备四大部分。这小节我们讲进程管理。

XV6 是分时多任务操作系统，与任何其他分时多任务操作系统一样，进程都是其核心概念。XV6 通过 PCB 进程控制块管理进程，XV6 的进程控制块是一个 [proc](https://github.com/professordeng/xv6-expansion/blob/master/proc.h#L37) 结构体。所有的进程控制块构成一个静态的数组 [ptable](https://github.com/professordeng/xv6-expansion/blob/master/proc.c#L10)（最多可容纳 64 个进程），未使用的进程控制块其状态设置为未使用（`ptable[X].state` = `UNUSED`）。XV6 进程控制块在进程亲缘关系方面只记录父进程，并不像 Linux 那样还记录子进程、兄弟进程的关系。进程控制块 `proc` 中记录的进程资源主要是该进程所打开的文件、页表、内核栈等少量信息。

从这里可以看出 XV6 仅可以作为教学系统，虽然 “麻雀虽小五脏俱全”，但是设计和实现上确实非常简陋。

## 1. 进程调度

由于各个处理器共享一个全局进程表 `ptable[]`，调度算法采用的是简单的时间片轮转算法。各处理器核上的调度器并发地共享一个就绪进程队列，因此需要用自旋锁进行互斥保护。

### 1.1 时间片轮转调度

如果用户程序一直占有 CPU 使得 XV6 操作系统得不到运行，那么操作系统什么事也做不了。因此计算机上通常设置了定时中断，通过中断转去执行中断服务代码（这些代码属于操作系统内核），因此操作系统才有机会接管系统并进行相关的管理。

定时器每经过一个计时周期 `tick` 就会发出时钟中断，每次都会经时钟中断处理函数进入到调度函数，从而引发一次调度。调度器则是简单地循环扫描 `ptable[]`，找到下一个就绪进程，并且切换过去。

也就是说，XV6 的时间片并不是指一个进程每次能执行一个时间片 `tick` 对应的时长，而是不管该进程什么时候开始拥有 `CPU` 的，只要到下一个时钟 `tick` 就必须再次调度。每一遍轮转过程中，各个进程所占有的 CPU 时间是不确定的，小于或等于一个时钟 `tick ` 长度。例如某个进程 A 在执行了半个 `tick` 时间然后阻塞睡眠，此时调度器执行另一个就绪进程 B，那么 B 执行半个 `tick` 后就将面临下一次调度。

### 1.2 调度切换时机

除了被动的定时中断原因外，还可能因为进程通过系统调用进入内核，出于某些原因而需要阻塞，从而主动要求切换到其他进程。例如发出磁盘读写操作、或者直接进行 `sleep()` 系统调用都可能进入阻塞并且换到其他进程。

### 1.3 执行断点

在继续讨论之前，我们先要澄清一下 “断点” 的概念，根据被中断的代码运行级别，它可能是

1. 用户态断点
2. 内核态断点

根据引发代码断点原因，可以分为

1. 系统调用 / 中断
2. 代码切换

这些断点根据上述组合，可以出现 4 大类，如下表所示

|        | 系统调用 / 中断                | 进程切换         |
| ------ | ------------------------------ | ---------------- |
| 用户态 | 进入内核态，断点在 `trapframe` | 无               |
| 内核态 | 保持内核态，断点在 `trapframe` | 断点在 `context` |

**系统调用 / 中断引起的断点**

这类断点都是用 `trapframe` 来保存的，如果发生在用户态则在 `trapframe` 中有 SS 和 ESP，而在内核态发生系统调用 / 中断，则 `trapframe` 中没有保存 SS 和 ESP 的。

- 进程的用户态断点 

  例如 shell 进程发出读按键操作的 `read()` 系统调用从而进入内核态，此时 shell 的用户态代码断点就是发出 `read()` 系统调用的位置，当从系统调用返回时需要从该位置的下一条指令开始运行。又或者，shell 进程在执行某条用户态代码中的指令 `X`，此时发生了时钟中断从而进入内核态执行中断服务程序，那么此时 shell 用户态断点就是 `X` 指令处，下次再被调度执行返回到用户态时，需要从该断点处恢复执行 `X+1` 指令。

- 进程的内核态断点

  例如，当进程在执行 `getpid()` 系统调用的过程中，发生了一次中断，那么断点在 `getpid()` 函数的某处。此时该断点也是通过一个 `trapframe` 保存在本进程的内核栈中。

**切换代码引起的断点**

该断点仅仅是进程切换的辅助代码中的断点，与进程本身没有关系（既不是系统调用应该执行的代码，也不是中断服务所需要执行的代码）。而仅仅是 `swith()` 切换代码的断点。

例如，上述 shell 进程进入到 `read()` 系统调用的内核态代码后，发现没有按键，则通过 `sleep()` 进入阻塞以等待按键发生，`sleep()` 进一步调用 `sched()` 换到其他进程，此时 shell 进程在 `sched()` 中的某处停止运行，该断点称为切换代码的断点。当用户按键之后进程重新成为就绪进程，下次被调度时需要从 `sched()` 处的断点恢复运行，并逐级返回将按键值传给 `shell` 进程。类似地，如果 `shell` 是因为时钟中断而进入内核代码，也存在 `sched()` 某处停止运行的断点，再次被调度时从该断点处继续运行。

- 注意 ：我们后面将系统调用 / 中断引起的断点简称为 “中断断点”，而切换代码断点成为 "切换断点"。

### 1.4 进程切换过程

XV6 通过上面的调度机制，实现多个进程在处理器上断续地、轮流地获得执行。我们称让出 CPU 的进程为 “换出进程”，即将获得 CPU 的进程为 “切入进程”，换出和切入两个操作就称为进程切换过程。在 XV6 代码中，调度代码相对比较简单，但是进程切换过程的代码则比较复杂。

进程切换总是发生在内核态，在底层实现上分成两个步骤：从换出进程的内核执行流切换到调度器 scheduler 的执行流，然后再从调度器 scheduler 的执行流切换到切入进程的执行流。

从换出进程 shell 切换到切入进程 cat 的过程如下：

|         | 流程描述                                        |
| ------- | ----------------------------------------------- |
| 第 1 步 | shell 从用户态进入内核态，保存数据到 `kstack`   |
| 第 2 步 | 执行 `swtch()` 切换到  scheduler，执行调度算法  |
| 第 3 步 | `swtch()` 到 cat 的核心态，从 `kstack` 中取数据 |
| 第 4 步 | cat 从核心态换到用户态，执行 cat                |

选择下一个被执行进程的调度操作是通过 [scheduler()](https://github.com/professordeng/xv6-expansion/blob/master/proc.c#L314) 函数完成，该函数虽然没有独立的进程号，但是具有独立的执行流，拥有独立的用于保存断点现场的堆栈。`scheduler()` 被设计成一个无限循环，其工作就是选取下一个就绪进程，然后切换过去执行。

从上表可以看出，用户进程必须进入到内核态（中断或系统调用）才可能发生切换，也就是说无论是 shell 进程被打断的位置，还是 cat 恢复执行的位置，都是处于某一段内核代码中的。因此进程切换所关心的进程执行 “切换断点” 并不在用户代码处（也就是说此处所谓的执行流的 “断点” 和用户代码没有关系）。当然从切换断点恢复执行后，最终还是要返回到因 “系统调用 / 中断” 引起的用户态断点或内核态断点的。例如我们这里从 shell 进程的 “切换断点” 恢复执行后，最终还将返回到用户态断点 `read()` 的下一条指令继续运行。

由于存在 “换出进程→内核→切入进程” 执行流的转换，需要解决执行流被打断执行后内核断点的现场保存，以及将新进程从上次内核断点处恢复现场并再次运行的问题。XV6 使用堆栈来保存这些现场，这就要求每个进程有自己的独立 “内核栈”，而每个处理器上的 `scheduler()` 内核执行流也要有自己的独立内核栈。同理，用户态断点也需要保存现场，XV6 代码借助 X86 的硬件中断机制，利用堆栈中的陷阱帧 `trapframe` 保存现场的。

## 2. 负载均衡

XV6 进程管理中只涉及进程的简单调度（多个处理器核共用一个就绪任务队列并按照轮询方式进行调度）。也就是说 XV6 并没有显式地进行负载均衡（只是在调度中简单地处理负载均衡问题）。
例如两个处理器，各自的调度器从一个公共的 `ptable[]` 中查找就绪进程，并执行该进程一个时间片，然后进入下一个调度周期。因此 `ptable[]` 的任务并不绑定于某个处理器，从而不会出现一个处理器忙而另一个处理器空闲的情况（这就间接地在一定程度上实现了负载均衡）。