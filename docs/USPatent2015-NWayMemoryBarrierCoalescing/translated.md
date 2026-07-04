(10) Patent No.: US 8,997,103 B2

> 
(10) 专利号：US 8,997,103 B2




(54) N-WAY MEMORY BARRIER OPERATION COALESCING

> 
(54) N 路内存屏障操作合并




(75) Inventors: Shirish Gadre, Fremont, CA (US); Charles McCarver, Madison, AL (US); Anjana Rajendran, San Jose, CA (US); Omkar Paranjape, Austin, TX (US); Steven James Heinrich, Madison, AL (US)

> 
(75) 发明人：Shirish Gadre，加利福尼亚州弗里蒙特（美国）；Charles McCarver，阿拉巴马州麦迪逊（美国）；Anjana Rajendran，加利福尼亚州圣何塞（美国）；Omkar Paranjape，德克萨斯州奥斯汀（美国）；Steven James Heinrich，阿拉巴马州麦迪逊（美国）




(73) Assignee: NVIDIA Corporation, Santa Clara, CA (US)

> 
(73) 专利权人：NVIDIA公司，加利福尼亚州圣克拉拉 (US)




(*) Notice: Subject to any disclaimer, the term of this patent is extended or adjusted under 35 U.S.C. 154(b) by 351 days.

> 
(*) 注：受任何免责声明影响，本专利期限依据《美国法典》第35编第154(b)条延长或调整了351天。




(21) Appl. No.: 13/441,785

> 
(21) 申请号：13/441,785




(22) Filed: Apr. 6, 2012

> 
(22) 申请日：2012年4月6日




Prior Publication Data

> 
在先公开数据




US 2012/0198214 A1 Aug. 2, 2012

> 
US 2012/0198214 A1 Aug. 2, 2012




Related U.S. Application Data

> 
相关美国申请数据




(63) Continuation-in-part of application No. 12/887,081, filed on Sep. 21, 2010.

> 
(63) 申请号12/887,081的部分继续申请，提交于2010年9月21日。




(60) Provisional application No. 61/246,047, filed on Sep. 25, 2009.

> 
(60) 临时申请号 61/246,047，提交于2009年9月25日。




(51) Int. Cl.

> 
(51) 国际专利分类




<table><tr><td>G06F 9/46</td><td>(2006.01)</td></tr><tr><td>G06F 1/04</td><td>(2006.01)</td></tr><tr><td>G06F 7/00</td><td>(2006.01)</td></tr><tr><td>G06F 9/38</td><td>(2006.01)</td></tr><tr><td>G06F 9/52</td><td>(2006.01)</td></tr><tr><td>G06F 9/52</td><td>(2006.01)</td></tr></table>

(52) U.S. Cl.

> 
(52) 美国专利分类




CPC ............ G06F 9/3834 (2013.01); G06F 9/3004 (2013.01); G06F 9/30087 (2013.01); G06F 9/3851 (2013.01); G06F 9/522 (2013.01) USPC ............ 718/103; 718/106; 713/375; 707/610

> 
合作专利分类............ G06F 9/3834 (2013.01); G06F 9/3004 (2013.01); G06F 9/30087 (2013.01); G06F 9/3851 (2013.01); G06F 9/522 (2013.01) 美国专利分类............ 718/103; 718/106; 713/375; 707/610




(58) Field of Classification Search

> 
(58) 分类检索领域




None

> 
本专利提出了一种提高并行处理架构中内存屏障操作效率的技术。主要研究问题是如何降低开销巨大的内存屏障指令对性能的影响，这些指令用于在多个协作的线程组之间排序内存事务。

关键的贡献在于一种N路内存屏障操作合并方法。当为某个线程组接收到第一条内存屏障指令时，该组后续内存操作的执行即被暂停。系统并不立即将此内存屏障发送至内存系统，而是开启一个合并窗口。在此时间窗口内到达的、来自不同线程组的后续内存屏障指令，会与第一条屏障合并（聚合），生成一条单一的合并内存屏障。系统构建一个带标记的内存命令流，其中每笔内存事务和合并屏障都与一个合并索引关联。这允许多组事务及其合并屏障同时处于执行中。当某条合并屏障正在处理时，仅阻塞受影响的线程组，其他独立线程组的内存操作可以继续执行。

主要结论是，该技术显著降低了内存屏障的开销。通过合并来自多个线程组的屏障，并重叠处理不同的合并集合，系统将单条屏障的长延迟分摊到众多线程上，并防止未参与同步的线程组不必要的停滞，从而提高了总体指令处理吞吐量。




See application file for complete search history.

> 
有关完整的检索历史，请参见申请文件。




(56) References Cited

> 
(56) 引用文献




### U.S. PATENT DOCUMENTS

7,725,618 B2* 5/2010 Day et al. ..................... 710/22

> 
7,725,618 B2* 5/2010 Day 等人 ..................... 710/22




2011/0078692 A1* 3/2011 Nickolls et al.

> 
2011/0078692 A1* 3/2011 Nickolls 等人。




* cited by examiner

> 
* 审查员引用




Primary Examiner - Mengyao Zhe

> 
首席审查员 - 孟瑶哲

本专利提出一种在并行处理架构中提高内存屏障操作效率的技术。主要研究问题是如何降低昂贵的内存屏障指令对性能的影响，这类指令用于在多个协作线程组之间排序内存事务。

关键贡献是一种 N 路内存屏障合并方法。当收到针对某个线程组的第一条内存屏障指令时，会暂停该线程组后续内存操作的执行。系统并不立即将该内存屏障发送至内存系统，而是开启一个合并窗口。在此时间窗口内到达的、来自不同线程组的后续内存屏障指令与第一条合并，生成一条单一的合并内存屏障。系统构建一个带标记的内存命令流，其中每个内存事务和合并后的屏障都与一个合并索引关联。这样，多组事务及其合并屏障便可同时处于执行状态。当一条合并屏障正在被处理时，仅阻塞受影响的线程组，其他独立线程组的内存操作可以继续执行。

主要结论是，该技术显著降低了内存屏障的开销。通过合并来自多个线程组的屏障，并重叠处理不同的合并集合，系统将单条屏障的长延迟分摊到众多线程上，同时避免无关线程组不必要的停顿，从而提高了整体指令处理吞吐量。




(74) Attorney, Agent, or Firm - Artegis Law Group, LLP

> 
(74) 律师、代理人或事务所 — Artegis Law Group, LLP




## (57) ABSTRACT

One embodiment sets forth a technique for N-way memory barrier operation coalescing. When a first memory barrier is received for a first thread group execution of subsequent memory operations for the first thread group are suspended until the first memory barrier is executed. Subsequent memory barriers for different thread groups may be coalesced with the first memory barrier to produce a coalesced memory barrier that represents memory barrier operations for multiple thread groups. When the coalesced memory barrier is being processed, execution of subsequent memory operations for the different thread groups is also suspended. However, memory operations for other thread groups that are not affected by the coalesced memory barrier may be executed.

> 
一个实施例阐述了一种用于N路内存屏障操作合并的技术。当接收到针对第一个线程组的第一个内存屏障时，该第一个线程组的后续内存操作的执行将被挂起，直到该第一个内存屏障执行完毕。随后来自不同线程组的内存屏障可与第一个内存屏障合并，产生一个代表多个线程组内存屏障操作的合并内存屏障。当该合并内存屏障正在被处理时，这些不同线程组的后续内存操作的执行也被挂起。然而，不受该合并内存屏障影响的其他线程组的内存操作可以被执行。




![0_667_1510_457_659_0.jpg](images/fig01.jpg)

![1_377_475_1050_1433_0.jpg](images/fig02.jpg)

Figure 1

> 
图1




![2_330_391_1113_1636_0.jpg](images/fig03.jpg)

Figure 2

> 
图2




![3_329_626_1155_1087_0.jpg](images/fig04.jpg)

Figure 3A

> 
图3A




![4_375_480_1044_1228_0.jpg](images/fig05.jpg)

Figure 3B

> 
图3B




![5_366_465_1043_1487_0.jpg](images/fig06.jpg)

Figure 3C

> 
图 3C




![6_270_433_1183_1251_0.jpg](images/fig07.jpg)

![7_325_693_1112_842_0.jpg](images/fig08.jpg)

Figure 5A

> 
图5A




![8_238_355_1233_1702_0.jpg](images/fig09.jpg)

![9_364_442_1076_1576_0.jpg](images/fig10.jpg)

![10_326_358_1109_1611_0.jpg](images/fig11.jpg)

Figure 7A

> 
图 7A




![11_332_414_1131_1618_0.jpg](images/fig12.jpg)

![12_359_391_1193_1673_1.jpg](images/fig13.jpg)

![13_405_462_974_1474_0.jpg](images/fig14.jpg)

Figure 8A

> 
图8A




![14_614_495_667_1209_0.jpg](images/fig15.jpg)

Figure 8B

> 
图 8B




![15_383_456_1061_1553_0.jpg](images/fig16.jpg)

## 1 N-WAY MEMORY BARRIER OPERATION COALESCING

## CROSS-REFERENCE TO RELATED APPLICATIONS

This application is a continuation in part of U.S. patent application titled, "COALESCING MEMORY BARRIER OPERATIONS ACROSS MULTIPLE PARALLEL THREADS," filed on Sep. 21, 2010 and having Ser. No. 12/887,081 and claiming priority benefit to U.S. provisional patent application titled, "PARALLEL, COALESCING MEMORY BARRIER IMPLEMENTATION," filed on Sep. 25, 2009 and having Ser. No. 61/246,047. These related patent applications are hereby incorporated herein by reference in their entirety.

> 
本申请是题为“COALESCING MEMORY BARRIER OPERATIONS ACROSS MULTIPLE PARALLEL THREADS”的美国专利申请（申请日：2010年9月21日，申请号：12/887,081）的部分继续申请，并要求享有题为“PARALLEL, COALESCING MEMORY BARRIER IMPLEMENTATION”的美国临时专利申请（申请日：2009年9月25日，申请号：61/246,047）的优先权。这些相关专利申请通过引用整体并入本文。




## BACKGROUND OF THE INVENTION

1. Field of the Invention

> 
1. 技术领域




Embodiments of the invention relate generally to multithreaded program execution, and more specifically to N-way memory barrier operation coalescing.

> 
本发明的实施例一般涉及多线程程序执行，更具体地涉及N路内存屏障操作合并。




2. Description of the Related Art

> 
2. 相关技术说明




Conventional parallel processing architectures support execution of multiple threads. A memory transaction is considered "performed" when it has been committed to memory order and is visible to any thread, processing unit, or device that may access the memory, e.g. a store or write operation has been "committed" to memory and subsequent load or read operations will see the stored data. Memory barrier instructions (or fence instructions) are used to order the execution of memory transactions. From the standpoint of one thread, processing unit, or device, when it executes a memory barrier instruction, it waits until all its prior memory transactions have committed to memory before executing any subsequent memory transactions. Within that thread, memory transactions that occur after the memory barrier instruction, in program order, are delayed until all of the threads' memory transactions that occur prior to the memory barrier instruction in program order are committed to memory. The results of committed memory transactions may be visible to other threads, and the memory barrier instruction delays the requesting thread until all its prior memory transactions are visible to other threads. After waiting for a memory barrier, the requesting thread may then synchronize or communicate with other threads knowing that they can access the results of its prior memory transactions. Parallel processors that support large numbers of parallel threads that cooperate or communicate, such as multi-threaded processors that execute thousands of parallel threads, need to frequently execute memory barrier instructions to ensure proper ordering and visibility of memory transactions. A conventional memory barrier instruction waits until the request travels to the system memory commit point where results are visible to all threads, processing units, and devices in the system, and then waits until an acknowledgement returns to the requesting thread. Round-trip latency to the system memory commit point can be very long, e.g. hundreds of cycles. Therefore, execution of memory barrier instructions can reduce the instruction processing throughput of a conventional parallel processing architecture since the multiple requesting threads are idle during execution of a long-latency memory barrier (waiting for memory transactions to be committed to memory and for results to become visible to all other threads).

> 
传统的并行处理架构支持多线程的执行。当内存事务已被提交到内存顺序，并且对可能访问该内存的任何线程、处理单元或设备可见时，即认为该内存事务“已执行”。例如，存储或写操作已“提交”到内存，随后的加载或读操作将看到存储的数据。内存屏障指令（或栅栏指令）用于对内存事务的执行进行排序。从某个线程、处理单元或设备的角度来看，当它执行内存屏障指令时，它会等待，直到其所有先前的内存事务都已提交到内存，然后才执行任何后续的内存事务。在该线程中，按程序顺序位于内存屏障指令之后的内存事务会被延迟，直到该线程按程序顺序位于内存屏障指令之前的所有内存事务都已提交到内存。已提交内存事务的结果可能对其他线程可见，而内存屏障指令会延迟请求线程，直到其所有先前的内存事务对其他线程可见。在等待内存屏障之后，请求线程随后可以与其他线程同步或通信，因为知道它们可以访问其先前内存事务的结果。支持大量协作或通信的并行线程的并行处理器，例如执行数千个并行线程的多线程处理器，需要频繁执行内存屏障指令，以确保内存事务的正确排序和可见性。传统的内存屏障指令会等待，直到请求传送到系统内存提交点（在该点结果对系统中的所有线程、处理单元和设备可见），然后等待确认返回到请求线程。到系统内存提交点的往返延迟可能非常长，例如数百个周期。因此，执行内存屏障指令可能会降低传统并行处理架构的指令处理吞吐量，因为多个请求线程在执行长延迟内存屏障期间处于空闲状态（等待内存事务提交到内存，并等待结果对所有其他线程可见）。




More recently, parallel processing architectures allow for sets of parallel threads to execute cooperatively together at

> 
最近，并行处理架构允许成组的并行线程在




2

> 
本专利提出了一种在并行处理架构中提高内存屏障操作效率的技术。主要研究问题是如何降低高开销的内存屏障指令对性能的影响，这些指令用于在多个协作线程组之间对内存事务进行排序。

其关键贡献是一种 N 路内存屏障操作合并的方法。当收到针对某个线程组的第一条内存屏障指令时，该组后续内存操作的执行将被暂停。系统不会立即将此内存屏障发送至内存系统，而是开启一个合并窗口。在此时间窗口内到达的、来自不同线程组的后续内存屏障指令将与第一条合并，产生一个合并后的单一内存屏障。系统构造一个带标记的内存命令流，其中每个内存事务及合并后的屏障均与一个合并索引相关联。这使得多组事务及其合并后的屏障可以同时处于执行状态。在处理一个合并后的屏障时，仅阻塞受影响的线程组，其他独立线程组的内存操作可以继续执行。

主要结论是，这项技术显著降低了内存屏障的开销。通过合并来自多个线程组的屏障，并重叠处理不同的合并集，系统将单个屏障的长延迟分摊到众多线程上，并避免未参与同步的线程组发生不必要的停顿，从而提高了整体指令处理吞吐量。




different thread grouping levels. For example, a set of parallel threads comprising a cooperative thread array (CTA) can execute together within a multi-threaded processor. Multiple CTAs can execute concurrently and cooperate within a pro- 5 cessor or among several processors, and also cooperate with other threads, processors, and devices in large systems. A CTA program may need to order memory transactions among the set of threads comprising the CTA, or among the threads executing in the same processor, or among different CTAs in 10 different processors, or among the threads, processors, and devices of the whole system. Therefore, execution of memory barrier instructions can further reduce the instruction processing throughput of a parallel processing architecture when threads are cooperating and interacting at multiple levels of cooperation across a parallel system having many parallel threads and processors.

> 
不同的线程分组层次。例如，构成协作线程阵列(CTA)的一组并行线程可以在一个多线程处理器内一起执行。多个CTA可以在一个处理器或多个处理器内并发执行并协作，还可以与大型系统中的其他线程、处理器及设备进行协作。CTA程序可能需要在组成该CTA的线程集之间、或在同一个处理器中执行的线程之间、或在不同处理器中的不同CTA之间、或在整个系统的线程、处理器和设备之间，对内存事务进行排序。因此，当线程在一个拥有众多并行线程和处理器的并行系统中以多个协作层次进行协作和交互时，内存屏障指令的执行会进一步降低并行处理架构的指令处理吞吐量。




Accordingly, what is needed in the art is an improved technique for performing a memory barrier operation across multiple parallel threads that are cooperating at multiple lev- 20 els in a parallel system.

> 
因此，本领域所需要的是，一种用于在并行系统中对跨多个并行线程执行内存屏障操作的改进技术，这些线程在多个 lev- 20 els 上进行协作。




## SUMMARY OF THE INVENTION

A system and method performs N-way memory barrier operation coalescing. Given the scalable nature of a multithreaded parallel processing architecture, the total number of threads on a given processing unit, and the total number of memory transactions that may be simultaneously in flight across an entire system, memory barrier operations can be costly to perform. As such, memory barrier requests from multiple threads or processing units are coalesced to reduce the impact to the rest of the system. Multiple memory barrier operations may be in-flight during execution of multiple thread groups, where each thread group includes a set of parallel threads.

> 
一种系统和方法执行N路内存屏障操作合并。鉴于多线程并行处理架构的可扩展特性、特定处理单元上的线程总数，以及整个系统中同时进行的内存事务总数，内存屏障操作的执行代价可能非常高昂。因此，多个线程或处理单元的内存屏障请求会被合并，以降低对系统其余部分的影响。在多个线程组执行期间，可以同时存在多个进行中的内存屏障操作，其中每个线程组包含一组并行线程。




Various embodiments of a method of the invention for processing memory barrier instructions include receiving a first memory barrier instruction for a first thread group that includes multiple parallel execution threads and blocking the execution of memory transactions for the first thread group that are subsequent to the first memory barrier instruction in program order. Subsequent to the first memory barrier instruction, a first set of memory transactions for at least a second thread group is received and a tagged memory command stream that includes the first set of memory transactions followed by the first memory barrier instruction is constructed, where a first coalescing index is associated with the first memory barrier instruction and each one of the memory transactions of the first set of memory transactions. The tagged memory command stream is transmitted to a memory management unit to process the first set of memory transactions and the first memory barrier instruction. After determining that all memory transactions that occur prior to the first memory barrier instruction in the tagged memory command stream are committed to memory the first memory barrier is released to allow the memory transactions for the first thread group that are subsequent to the first memory barrier instruction in program order to be executed.

> 
本发明用于处理内存屏障指令的方法的各种实施例包括：接收针对第一线程组的第一内存屏障指令，所述第一线程组包含多个并行执行线程；并且阻止所述第一线程组在程序顺序上位于所述第一内存屏障指令之后的内存事务的执行。在所述第一内存屏障指令之后，接收针对至少第二线程组的第一组内存事务，并构建带标记的内存命令流，该命令流包含所述第一组内存事务，其后跟随着所述第一内存屏障指令，其中第一合并索引与所述第一内存屏障指令以及所述第一组内存事务中的每个内存事务相关联。所述带标记的内存命令流被传输至内存管理单元，以处理所述第一组内存事务和所述第一内存屏障指令。在确定所述带标记的内存命令流中在所述第一内存屏障指令之前出现的所有内存事务都已提交至内存后，释放所述第一内存屏障，以允许所述第一线程组在程序顺序上位于所述第一内存屏障指令之后的内存事务被执行。




Various embodiments of the invention include a processing subsystem for processing memory barrier instructions. The processing subsystem includes an instruction scheduling unit that is configured to issue for execution a first memory barrier instruction for a first thread group that includes multiple parallel execution threads. Subsequent to the first memory

> 
本发明的各种实施例包括一种用于处理内存屏障指令的处理子系统。该处理子系统包括一个指令调度单元，其被配置为为第一线程组发出用于执行的第一内存屏障指令，该第一线程组包括多个并行执行线程。在第一内存




65 barrier instruction, the instruction scheduling unit issues for execution a first set of memory transactions for at least a second thread group. The instruction scheduling unit blocks

> 
65条屏障指令时，指令调度单元为至少第二个线程组发出第一组内存事务以供执行。指令调度单元阻塞




3

> 
本专利提出了一种提高并行处理架构中内存屏障操作效率的技术。主要研究问题是如何降低内存屏障指令（对多个协同线程组的内存事务进行排序）带来的高昂性能开销。

核心贡献是一种N路内存屏障操作合并方法。当第一个内存屏障指令到达一个线程组时，该组后续内存操作的执行将被暂停。系统不会立即将此内存屏障发送到内存系统，而是会打开一个合并窗口。在此时间窗口内到达的、来自不同线程组的后续内存屏障指令，会与第一个屏障合并，生成一条合并后的内存屏障。系统构建了一个带标记的内存命令流，其中每个内存事务和合并后的屏障都关联一个合并索引。这使得多组事务及其合并屏障可以同时并发执行。当某个合并屏障正在处理时，仅阻止受影响的线程组，其他独立线程组的内存操作可以继续执行。

主要结论是，该技术显著降低了内存屏障的开销。通过合并来自多个线程组的屏障，并重叠处理不同的合并集合，系统将单次屏障的长延迟分摊到众多线程上，并避免了未参与同步的线程组不必要的停顿，从而提高了整体指令处理吞吐量。




the execution of memory transactions for the first thread group that are subsequent to the first memory barrier instruction in program order and releases the first memory barrier to allow execution of the memory transactions for the first thread group that are subsequent to the first memory barrier instruction in program order when an acknowledgement signal is received. The processing subsystem also includes a memory management unit that is configured to process memory transactions and memory barrier instructions and a memory barrier instruction execution unit. The memory barrier instruction execution unit is configured to receive the first memory barrier instruction, receive the first set of memory transactions, and construct a tagged memory command stream that includes the first set of memory transactions followed by the first memory barrier instruction, where a first coalescing index is associated with the first memory barrier instruction and each one of the memory transactions of the first set of memory transactions.

> 
暂停第一个线程组中在程序顺序上位于第一个内存屏障指令之后的内存事务的执行，并在接收到确认信号时释放第一个内存屏障，以允许第一个线程组中在程序顺序上位于第一个内存屏障指令之后的内存事务执行。该处理子系统还包括一个内存管理单元，该内存管理单元被配置为处理内存事务和内存屏障指令，以及一个内存屏障指令执行单元。该内存屏障指令执行单元被配置为接收第一个内存屏障指令，接收第一组内存事务，并构建一个带标签的内存命令流，该命令流包括第一组内存事务，后跟第一个内存屏障指令，其中第一个合并索引与第一个内存屏障指令及第一组内存事务中的每一个内存事务相关联。




One advantage of the disclosed system and method is that when a first memory barrier is received for a first thread group execution of subsequent memory operations for the first thread group are held until the first memory barrier is executed. Subsequent memory barriers for different thread groups may be coalesced with the first memory barrier to produce a coalesced memory barrier that represents memory barriers for multiple thread groups. When the coalesced memory barrier is being processed, execution of subsequent memory operations for the first thread group and the different thread groups is suspended. However, memory operations for other thread groups that are not affected by the coalesced memory barrier may be executed.

> 
所公开的系统和方法的一个优点是，当针对第一个线程组接收到第一个内存屏障时，第一个线程组的后续内存操作将被挂起，直到第一个内存屏障执行完毕。针对不同线程组的后续内存屏障可以与第一个内存屏障合并，生成一个代表多个线程组内存屏障的合并内存屏障。当合并内存屏障被处理时，第一个线程组以及所述不同线程组的后续内存操作的执行会被暂停。然而，不受该合并内存屏障影响的其他线程组的内存操作仍可执行。




## BRIEF DESCRIPTION OF THE DRAWINGS

So that the manner in which the above recited features of the present invention can be understood in detail, a more particular description of the invention, briefly summarized above, may be had by reference to embodiments, some of which are illustrated in the appended drawings. It is to be noted, however, that the appended drawings illustrate only typical embodiments of this invention and are therefore not to be considered limiting of its scope, for the invention may admit to other equally effective embodiments.

> 
为能详细理解本发明的上述特征，可通过引用实施例来获得对上文简要概述之本发明的更具体描述，其中部分实施例示于附图中。然而，应当注意，附图仅示出本发明的典型实施例，故不应视为对其范围的限制，因为本发明可涵盖其他等效实施例。




FIG. 1 is a block diagram illustrating a computer system configured to implement one or more aspects of the present invention;

> 
图1是示出被配置为实现本发明的一个或多个方面的计算机系统的框图；




FIG. 2 is a block diagram of a parallel processing subsystem for the computer system of FIG. 1, according to one embodiment of the present invention;

> 
图2是根据本发明一个实施例的、用于图1计算机系统的并行处理子系统的框图；




FIG. 3A is a block diagram of a GPC within one of the PPUs of FIG. 2, according to one embodiment of the present invention;

> 
图3A是根据本发明的一个实施例的图2的PPU之一内的GPC的框图；




FIG. 3B is a block diagram of a partition unit within one of the PPUs of FIG. 2, according to one embodiment of the present invention;

> 
图3B是根据本发明一个实施例的图2的PPU之一内的分区单元的框图；




FIG. 3C is a block diagram of a portion of the SPM of FIG. 3A, according to one embodiment of the present invention; and

> 
图3C为根据本发明一个实施例的图3A中SPM的一部分的方框图；以及




FIG. 4 is a block diagram of a memory barrier instruction execution unit, according to one embodiment of the present invention;

> 
图4是根据本发明一个实施例的存储器屏障指令执行单元的框图；




FIG. 5A is a block diagram of finite state machine, according to one embodiment of the present invention;

> 
图5A是根据本发明一个实施例的有限状态机的框图；




FIG. 5B is a conceptual diagram illustrating the coalescing of memory barrier instructions, according to one embodiment of the present invention;

> 
图5B是根据本发明一个实施例示出内存屏障指令合并的概念图；




## 4

FIG. 6 is a flow diagram of method steps for coalescing memory barrier instructions, according to one embodiment of the present invention;

> 
图6是根据本发明一个实施例的用于合并内存屏障指令的方法步骤的流程图；




FIG. 7A is another block diagram of a memory barrier instruction execution unit, according to one embodiment of the present invention;

> 
图7A是根据本发明一个实施例的存储器屏障指令执行单元的另一框图；




FIG. 7B is a block diagram of finite state machines, according to one embodiment of the present invention;

> 
FIG. 7B 是有限状态机的框图，根据本发明的一个实施例；




FIG. 7C is another conceptual diagram illustrating the coalescing of memory barrier instructions, according to one embodiment of the present invention;

> 
图7C是示出根据本发明一个实施例的存储屏障指令合并的另一概念图；




FIG. 8A is a conceptual diagram of illustrating the tagged memory command stream buffer of FIG. 7A, according to one embodiment of the present invention;

> 
图8A是说明根据本发明一个实施例的图7A的标记内存命令流缓冲区的概念图；




FIG. 8B is another flow diagram of method steps for processing memory barrier instructions, according to one embodiment of the present invention; and

> 
图8B是根据本发明一个实施例的处理内存屏障指令的方法步骤的另一流程图；以及




FIG. 8C is another flow diagram of method steps for N-way memory barrier instruction coalescing, according to one embodiment of the present invention.

> 
图8C是根据本发明一个实施例的用于N路内存屏障指令合并的方法步骤的另一流程图。




## DETAILED DESCRIPTION

In the following description, numerous specific details are set forth to provide a more thorough understanding of the present invention. However, it will be apparent to one of skill in the art that the present invention may be practiced without one or more of these specific details. In other instances, wellknown features have not been described in order to avoid 0 obscuring the present invention.

> 
在以下描述中，阐述了许多具体细节以提供对本发明的更透彻理解。然而，对于本领域的技术人员来说显而易见的是，本发明可以在没有这些具体细节中的一个或多个的情况下实施。在其他实例中，为了避免0模糊本发明，未对公知特征进行描述。




## System Overview

FIG. 1 is a block diagram illustrating a computer system 35 100 configured to implement one or more aspects of the present invention. Computer system 100 includes a central processing unit (CPU) 102 and a system memory 104 communicating via an interconnection path that may include a memory bridge 105. Memory bridge 105, which may be, e.g.,

> 
图1是示出被配置为实现本发明的一个或多个方面的计算机系统35 100的框图。计算机系统100包括经由互连路径进行通信的中央处理单元（CPU）102和系统存储器104，该互连路径可以包括存储器桥105。存储器桥105，其可以是例如，




a Northbridge chip, is connected via a bus or other communication path 106 (e.g., a HyperTransport link) to an I/O (input/output) bridge 107. I/O bridge 107, which may be, e.g., a Southbridge chip, receives user input from one or more user input devices 108 (e.g., keyboard, mouse) and forwards the

> 
北桥芯片，通过总线或其他通信路径106（例如，HyperTransport链路）连接到I/O（输入/输出）桥107。I/O桥107，例如可以是南桥芯片，从一个或多个用户输入设备108（例如键盘、鼠标）接收用户输入，并转发该




input to CPU 102 via path 106 and memory bridge 105. A parallel processing subsystem 112 is coupled to memory bridge 105 via a bus or other communication path 113 (e.g., a PCI Express, Accelerated Graphics Port, or HyperTransport link); in one embodiment parallel processing subsystem 112

> 
输入经由路径106和内存桥105送至CPU 102。并行处理子系统112通过总线或其他通信路径113（例如，PCI Express、加速图形端口或HyperTransport链路）耦合至内存桥105；在一个实施例中，并行处理子系统112




50 is a graphics subsystem that delivers pixels to a display device 110 (e.g., a conventional CRT or LCD based monitor). A system disk 114 is also connected to I/O bridge 107. A switch 116 provides connections between I/O bridge 107 and other components such as a network adapter 118 and various add-in

> 
50 是一种图形子系统，可将像素传输至显示设备 110（例如，传统的 CRT 或基于 LCD 的监视器）。系统磁盘 114 也连接到 I/O 桥 107。开关 116 在 I/O 桥 107 与其他组件（如网络适配器 118 及各种附加卡）之间提供连接。




cards 120 and 121. Other components (not explicitly shown), including USB or other port connections, CD drives, DVD drives, film recording devices, and the like, may also be connected to I/O bridge 107. Communication paths interconnecting the various components in FIG. 1 may be imple-

> 
卡120和121。其他组件（未明确显示），包括USB或其他端口连接、CD驱动器、DVD驱动器、胶片记录设备等，也可连接到I/O桥107。图1中互连各种组件的通信路径可能imple-




mented using any suitable protocols, such as PCI (Peripheral Component Interconnect), PCI-Express, AGP (Accelerated Graphics Port), HyperTransport, or any other bus or point-to-point communication protocol(s), and connections between different devices may use different protocols as is known in 65 the art.

> 
可使用任何合适的协议来实现，例如 PCI（外设组件互连）、PCI-Express、AGP（加速图形端口）、HyperTransport 或任何其他总线或点对点通信协议，并且不同设备之间的连接可使用如本领域已知的不同协议（65）。




In one embodiment, the parallel processing subsystem 112 incorporates circuitry optimized for graphics and video processing, including, for example, video output circuitry, and constitutes a graphics processing unit (GPU). In another embodiment, the parallel processing subsystem 112 incorporates circuitry optimized for general purpose processing, while preserving the underlying computational architecture, described in greater detail herein. In yet another embodiment, the parallel processing subsystem 112 may be integrated with one or more other system elements, such as the memory bridge 105, CPU 102, and I/O bridge 107 to form a system on chip (SoC).

> 
在一个实施例中，并行处理子系统112结合了针对图形和视频处理优化的电路，例如包括视频输出电路，并构成图形处理单元（GPU）。在另一个实施例中，并行处理子系统112结合了针对通用处理优化的电路，同时保留了底层计算架构，本文对此有更详细的描述。在又一个实施例中，并行处理子系统112可以与一个或多个其他系统元件集成，例如存储器桥105、CPU 102和I/O桥107，以形成片上系统（SoC）。




It will be appreciated that the system shown herein is illustrative and that variations and modifications are possible. The connection topology, including the number and arrangement of bridges, the number of CPUs 102, and the number of parallel processing subsystems 112, may be modified as desired. For instance, in some embodiments, system memory 104 is connected to CPU 102 directly rather than through a bridge, and other devices communicate with system memory 104 via memory bridge 105 and CPU 102. In other alternative topologies, parallel processing subsystem 112 is connected to I/O bridge 107 or directly to CPU 102, rather than to memory bridge 105. In still other embodiments, I/O bridge 107 and memory bridge 105 might be integrated into a single chip. Large embodiments may include two or more CPUs 102 and two or more parallel processing systems 112. The particular components shown herein are optional; for instance, any number of add-in cards or peripheral devices might be supported. In some embodiments, switch 116 is eliminated, and network adapter 118 and add-in cards 120, 121 connect directly to I/O bridge 107.

> 
应当理解，本文所示的系统是说明性的，并且可以有变化和修改。连接拓扑结构，包括桥接器的数量和布局、CPU 102的数量以及并行处理子系统112的数量，均可按需修改。例如，在一些实施例中，系统内存104直接连接到CPU 102而非通过桥接器，而其他设备则通过内存桥接器105和CPU 102与系统内存104通信。在其他替代拓扑中，并行处理子系统112连接到I/O桥接器107或直接连接到CPU 102，而不是连接到内存桥接器105。在另一些实施例中，I/O桥接器107和内存桥接器105可以集成到单一的芯片中。大型实施例可包括两个或更多个CPU 102以及两个或更多个并行处理系统112。本文所示的具体组件均为可选的；例如，可以支持任意数量的附加卡或外围设备。在一些实施例中，交换机116被省略，网络适配器118和附加卡120、121直接连接到I/O桥接器107。




FIG. 2 illustrates a parallel processing subsystem 112, according to one embodiment of the present invention. As shown, parallel processing subsystem 112 includes one or more parallel processing units (PPUs) 202, each of which is coupled to a local parallel processing (PP) memory 204. In general, a parallel processing subsystem includes a number U of PPUs, where U≥1. (Herein, multiple instances of like objects are denoted with reference numbers identifying the object and parenthetical numbers identifying the instance where needed.) PPUs 202 and parallel processing memories 204 may be implemented using one or more integrated circuit devices, such as programmable processors, application specific integrated circuits (ASICs), or memory devices, or in any other technically feasible fashion.

> 
图2展示了根据本发明一个实施例的并行处理子系统112。如图所示，并行处理子系统112包括一个或多个并行处理单元（PPU）202，每个PPU耦合到本地并行处理（PP）存储器204。通常，并行处理子系统包括U个PPU，其中U≥1。（在本文中，类似对象的多个实例用标识该对象的参考编号和括号内的实例编号表示，必要时使用。）PPU 202和并行处理存储器204可以使用一个或多个集成电路器件实现，例如可编程处理器、专用集成电路（ASIC）或存储器器件，或以任何其他技术上可行的方式实现。




Referring again to FIG. 1, in some embodiments, some or all of PPUs 202 in parallel processing subsystem 112 are graphics processors with rendering pipelines that can be configured to perform various tasks related to generating pixel data from graphics data supplied by CPU 102 and/or system memory 104 via memory bridge 105 and bus 113, interacting with local parallel processing memory 204 (which can be used as graphics memory including, e.g., a conventional frame buffer) to store and update pixel data, delivering pixel data to display device 110, and the like. In some embodiments, parallel processing subsystem 112 may include one or more PPUs 202 that operate as graphics processors and one or more other PPUs 202 that are used for general-purpose computations. The PPUs may be identical or different, and each PPU may have its own dedicated parallel processing memory device(s) or no dedicated parallel processing memory device(s). One or more PPUs 202 may output data to display device 110 or each PPU 202 may output data to one or more display devices 110.

> 
再次参考图1，在一些实施例中，并行处理子系统112中的部分或全部PPU 202是图形处理器，其渲染管线可配置为执行与从CPU 102和/或系统内存104经内存桥105和总线113提供的图形数据生成像素数据相关的各种任务、与本地并行处理内存204（可用作图形内存，例如包括传统帧缓冲器）交互以存储和更新像素数据、向显示设备110传送像素数据等。在一些实施例中，并行处理子系统112可包含一个或多个作为图形处理器运行的PPU 202，以及一个或多个用于通用计算的其他PPU 202。这些PPU可相同或不同，且每个PPU可有自己的专用并行处理内存设备，或没有专用并行处理内存设备。一个或多个PPU 202可向显示设备110输出数据，或者每个PPU 202可向一个或多个显示设备110输出数据。




In operation, CPU 102 is the master processor of computer system 100, controlling and coordinating operations of other system components. In particular, CPU 102 issues commands that control the operation of PPUs 202. In some embodiments, CPU 102 writes a stream of commands for each PPU 202 to a pushbuffer (not explicitly shown in either FIG. 1 or FIG. 2) that may be located in system memory 104, parallel processing memory 204, or another storage location accessible to both CPU 102 and PPU 202. PPU 202 reads the command stream from the pushbuffer and then executes commands asynchronously relative to the operation of CPU 102.

> 
在操作中，CPU 102 是计算机系统 100 的主处理器，负责控制和协调其他系统组件的运行。具体而言，CPU 102 发出的命令控制着 PPU 202 的操作。在某些实施例中，CPU 102 为每个 PPU 202 将命令流写入一个推送缓冲区（未在图 1 或图 2 中明确示出），该缓冲区可位于系统内存 104、并行处理内存 204，或 CPU 102 与 PPU 202 均可访问的其他存储位置。PPU 202 从推送缓冲区读取命令流，随后相对于 CPU 102 的操作异步执行这些命令。




Referring back now to FIG. 2, each PPU 202 includes an I/O (input/output) unit 205 that communicates with the rest of 10 computer system 100 via communication path 113, which connects to memory bridge 105 (or, in one alternative embodiment, directly to CPU 102). The connection of PPU 202 to the rest of computer system 100 may also be varied. In some embodiments, parallel processing subsystem 112 is implemented as an add-in card that can be inserted into an expansion slot of computer system 100. In other embodiments, a PPU 202 can be integrated on a single chip with a bus bridge, such as memory bridge 105 or I/O bridge 107. In still other embodiments, some or all elements of PPU 202 may be integrated on a single chip with CPU 102.

> 
现在回到图2，每个PPU 202包含一个I/O（输入/输出）单元205，该单元通过通信路径113与计算机系统100的其余部分进行通信，该路径连接到内存桥105（或在一个替代实施例中，直接连接到CPU 102）。PPU 202与计算机系统100其余部分的连接方式也可以有所变化。在一些实施例中，并行处理子系统112被实现为可插入计算机系统100扩展槽的附加卡。在其他实施例中，PPU 202可以与总线桥（如内存桥105或I/O桥107）集成在单个芯片上。在另一些实施例中，PPU 202的部分或全部元件可以与CPU 102集成在单个芯片上。




In one embodiment, communication path 113 is a PCI-EXPRESS link, in which dedicated lanes are allocated to each PPU 202, as is known in the art. Other communication paths may also be used. An I/O unit 205 generates packets (or other 25 signals) for transmission on communication path 113 and also receives all incoming packets (or other signals) from communication path 113, directing the incoming packets to appropriate components of PPU 202. For example, commands related to processing tasks may be directed to a host 30 interface 206, while commands related to memory operations (e.g., reading from or writing to parallel processing memory 204) may be directed to a memory crossbar unit 210. Host interface 206 reads each pushbuffer and outputs the work specified by the pushbuffer to a front end 212.

> 
在一个实施例中，通信路径113是一条PCI-EXPRESS链路，其中为每个PPU 202分配了专用通道，如本领域已知的那样。也可以使用其他通信路径。I/O单元205生成用于在通信路径113上传输的数据包（或其他25信号），同时也从通信路径113接收所有传入的数据包（或其他信号），并将传入的数据包导向PPU 202的适当组件。例如，与处理任务相关的命令可被导向主机接口206，而与存储器操作（例如，读取或写入并行处理存储器204）相关的命令可被导向存储器交叉开关单元210。主机接口206读取每个推送缓冲区，并将推送缓冲区所指定的工作输出到前端212。




Each PPU 202 advantageously implements a highly parallel processing architecture. As shown in detail, PPU 202(0) includes a processing cluster array 230 that includes a number C of general processing clusters (GPCs) 208, where $\mathrm{C} \geq  1$ . Each GPC 208 is capable of executing a large number (e.g., hundreds or thousands) of threads concurrently, where each thread is an instance of a program. In various applications, different GPCs 208 may be allocated for processing different types of programs or for performing different types of computations. For example, in a graphics application, a first set of GPCs 208 may be allocated to perform tessellation operations and to produce primitive topologies for patches, and a second set of GPCs 208 may be allocated to perform tessellation shading to evaluate patch parameters for the primitive topologies and to determine vertex positions and other per-vertex 50 attributes. The allocation of GPCs 208 may vary dependent on the workload arising for each type of program or computation.

> 
每个PPU 202均有利地实现了高度并行的处理架构。如图所示，PPU 202(0)包括一个处理簇阵列230，该阵列包含C个通用处理簇（GPC）208，其中 $\mathrm{C} \geq  1$。每个GPC 208能够并发执行大量（例如数百或数千）线程，其中每个线程是程序的一个实例。在各种应用中，可以分配不同的GPC 208来处理不同类型的程序或执行不同类型的计算。例如，在图形应用中，第一组GPC 208可被分配用于执行曲面细分操作并为面片生成图元拓扑，第二组GPC 208可被分配用于执行曲面细分着色，以评估图元拓扑的面片参数并确定顶点位置及其他每顶点50个属性。GPC 208的分配可根据每种程序或计算类型产生的工作负载而变化。




GPCs 208 receive processing tasks to be executed via a work distribution unit 200, which receives commands defining processing tasks from front end unit 212. Processing tasks include indices of data to be processed, e.g., surface (patch) data, primitive data, vertex data, and/or pixel data, as well as state parameters and commands defining how the data is to be processed (e.g., what program is to be executed). Work distribution unit 200 may be configured to fetch the indices corresponding to the tasks, or work distribution unit 200 may receive the indices from front end 212. Front end 212 ensures that GPCs 208 are configured to a valid state before the processing specified by the pushbuffers is initiated.

> 
GPC 208 通过工作分配单元 200 接收待执行的处理任务，该工作分配单元从前端单元 212 接收定义处理任务的命令。处理任务包括待处理数据的索引，例如表面（面片）数据、图元数据、顶点数据和/或像素数据，以及定义如何处理数据的状态参数和命令（例如要执行哪个程序）。工作分配单元 200 可被配置为获取与任务对应的索引，或者工作分配单元 200 也可从前端 212 接收索引。前端 212 确保在启动由推缓冲区指定的处理之前，GPC 208 被配置为有效状态。




When PPU 202 is used for graphics processing, for example, the processing workload for each patch is divided into approximately equal sized tasks to enable distribution of the tessellation processing to multiple GPCs 208. A work distribution unit 200 may be configured to produce tasks at a frequency capable of providing tasks to multiple GPCs 208 for processing. By contrast, in conventional systems, processing is typically performed by a single processing engine, while the other processing engines remain idle, waiting for the single processing engine to complete its tasks before beginning their processing tasks. In some embodiments of the present invention, portions of GPCs 208 are configured to perform different types of processing. For example a first portion may be configured to perform vertex shading and topology generation, a second portion may be configured to perform tessellation and geometry shading, and a third portion may be configured to perform pixel shading in screen space to produce a rendered image. Intermediate data produced by GPCs 208 may be stored in buffers to allow the intermediate data to be transmitted between GPCs 208 for further processing.

> 
当 PPU 202 用于图形处理时，例如，每个面片的处理工作负载被划分为大致相等大小的任务，以实现将曲面细分处理分布到多个 GPC 208。工作分配单元 200 可被配置为以能够向多个 GPC 208 提供任务进行处理的频率产生任务。相比之下，在常规系统中，处理通常由单个处理引擎执行，而其他处理引擎则处于空闲状态，等待该单个处理引擎完成其任务后才开始它们自己的处理任务。在本发明的一些实施例中，GPC 208 的各部分被配置为执行不同类型的处理。例如，第一部分可被配置为执行顶点着色和拓扑生成，第二部分可被配置为执行曲面细分和几何着色，第三部分可被配置为在屏幕空间中执行像素着色以产生渲染图像。由 GPC 208 产生的中间数据可存储在缓冲区中，以允许该中间数据在 GPC 208 之间传输以进行进一步处理。




Memory interface 214 includes a number D of partition units 215 that are each directly coupled to a portion of parallel processing memory 204, where D≥1. As shown, the number of partition units 215 generally equals the number of DRAM 220. In other embodiments, the number of partition units 215 may not equal the number of memory devices. Persons skilled in the art will appreciate that DRAM 220 may be replaced 2 with other suitable storage devices and can be of generally conventional design. A detailed description is therefore omitted. Render targets, such as frame buffers or texture maps may be stored across DRAMs 220, allowing partition units 215 to write portions of each render target in parallel to efficiently use the available bandwidth of parallel processing memory 204.

> 
内存接口214包括D个分区单元215，每个分区单元直接耦合到并行处理存储器204的一部分，其中D≥1。如图所示，分区单元215的数量通常等于DRAM 220的数量。在其它实施例中，分区单元215的数量可能不等于存储器件的数量。本领域技术人员将理解，DRAM 220可被替换2为其它适当的存储设备，并且可以是常规设计。因此省略详细描述。渲染目标（例如帧缓冲器或纹理映射）可跨DRAM 220存储，从而允许分区单元215并行写入每个渲染目标的部分，以高效利用并行处理存储器204的可用带宽。




Any one of GPCs 208 may process data to be written to any of the DRAMs 220 within parallel processing memory 204. Crossbar unit 210 is configured to route the output of each GPC 208 to the input of any partition unit 215 or to another GPC 208 for further processing. GPCs 208 communicate with memory interface 214 through crossbar unit 210 to read from or write to various external memory devices. In one embodiment, crossbar unit 210 has a connection to memory interface 214 to communicate with I/O unit 205, as well as a connection to local parallel processing memory 204, thereby enabling the processing cores within the different GPCs 208 to communicate with system memory 104 or other memory that is not local to PPU 202. In the embodiment shown in FIG. 2, crossbar unit 210 is directly connected with I/O unit 205. Crossbar unit 210 may use virtual channels to separate traffic streams between the GPCs 208 and partition units 215.

> 
任何一个 GPC 208 均可处理要写入并行处理存储器 204 内任意 DRAM 220 的数据。交叉开关单元 210 被配置为将各 GPC 208 的输出路由至任意分区单元 215 的输入或另一 GPC 208 以供进一步处理。GPC 208 通过交叉开关单元 210 与存储器接口 214 通信，以读取或写入各种外部存储设备。在一个实施例中，交叉开关单元 210 具有与存储器接口 214 的连接以便与 I/O 单元 205 通信，并具有与局部并行处理存储器 204 的连接，从而使不同 GPC 208 内的处理核心能够与系统存储器 104 或非 PPU 202 本地的其他存储器进行通信。在图 2 所示的实施例中，交叉开关单元 210 直接与 I/O 单元 205 连接。交叉开关单元 210 可使用虚拟通道来分隔 GPC 208 与分区单元 215 之间的流量流。




Again, GPCs 208 can be programmed to execute processing tasks relating to a wide variety of applications, including but not limited to, linear and nonlinear data transforms, filtering of video and/or audio data, modeling operations (e.g., applying laws of physics to determine position, velocity and other attributes of objects), image rendering operations (e.g., tessellation shader, vertex shader, geometry shader, and/or pixel shader programs), and so on. PPUs 202 may transfer data from system memory 104 and/or local parallel processing memories 204 into internal (on-chip) memory, process the data, and write result data back to system memory 104 and/or local parallel processing memories 204, where such data can be accessed by other system components, including CPU 102 or another parallel processing subsystem 112.

> 
同样，GPC 208 可被编程以执行涉及多种应用的处理任务，包括但不限于线性与非线性数据变换、视频和/或音频数据的滤波、建模操作（例如，应用物理定律来确定对象的位置、速度及其他属性）、图像渲染操作（例如，曲面细分着色器、顶点着色器、几何着色器和/或像素着色器程序）等。PPU 202 可将数据从系统内存 104 和/或本地并行处理内存 204 传输到内部（片上）内存，处理数据，并将结果数据写回系统内存 104 和/或本地并行处理内存 204，这些数据随后可被包括 CPU 102 或另一个并行处理子系统 112 在内的其他系统组件访问。




A PPU 202 may be provided with any amount of local parallel processing memory 204, including no local memory, and may use local memory and system memory in any combination. For instance, a PPU 202 can be a graphics processor in a unified memory architecture (UMA) embodiment. In such embodiments, little or no dedicated graphics (parallel processing) memory would be provided, and PPU 202 would use system memory exclusively or almost exclusively. In UMA embodiments, a PPU 202 may be integrated into a bridge chip or processor chip or provided as a discrete chip with a high-speed link (e.g., PCI-EXPRESS) connecting the PPU 202 to system memory via a bridge chip or other communication means.

> 
PPU 202 可以配备任意数量的本地并行处理内存 204，包括没有本地内存，并且可以以任意组合使用本地内存和系统内存。例如，PPU 202 可以是统一内存架构（UMA）实施例中的图形处理器。在此类实施例中，将提供很少或没有专用的图形（并行处理）内存，PPU 202 将专门或几乎专门使用系统内存。在 UMA 实施例中，PPU 202 可以集成到桥接芯片或处理器芯片中，或者作为独立芯片提供，通过高速链路（例如 PCI-EXPRESS）经由桥接芯片或其他通信方式将 PPU 202 连接到系统内存。




As noted above, any number of PPUs 202 can be included in a parallel processing subsystem 112. For instance, multiple PPUs 202 can be provided on a single add-in card, or multiple add-in cards can be connected to communication path 113, or one or more of PPUs 202 can be integrated into a bridge chip. PPUs 202 in a multi-PPU system may be identical to or 5 different from one another. For instance, different PPUs 202 might have different numbers of processing cores, different amounts of local parallel processing memory, and so on. Where multiple PPUs 202 are present, those PPUs may be operated in parallel to process data at a higher throughput than 20 is possible with a single PPU 202. Systems incorporating one or more PPUs 202 may be implemented in a variety of configurations and form factors, including desktop, laptop, or handheld personal computers, servers, workstations, game consoles, embedded systems, and the like.

> 
如上所述，并行处理子系统112中可以包含任意数量的PPU 202。例如，多个PPU 202可以布置在单个附加卡上，或者多个附加卡可以连接到通信路径113，或者一个或多个PPU 202可以集成到桥接芯片中。多PPU系统中的PPU 202可以彼此相同，也可以5不同。例如，不同的PPU 202可能具有不同数量的处理核心、不同大小的本地并行处理内存等。在存在多个PPU 202的情况下，这些PPU可以并行操作，以比单个PPU 202可能达到的20更高的吞吐量处理数据。包含一个或多个PPU 202的系统可以以各种配置和外形规格实现，包括台式机、笔记本电脑或手持个人计算机、服务器、工作站、游戏机、嵌入式系统等。




## Processing Cluster Array Overview

FIG. 3A is a block diagram of a GPC 208 within one of the PPUs 202 of FIG. 2, according to one embodiment of the present invention. Each GPC 208 may be configured to execute a large number of threads in parallel, where the term "thread" refers to an instance of a particular program executing instructions on a particular set of input data. In some embodiments, single-instruction, multiple-data (SIMD) instruction issue techniques are used to support parallel execution of a large number of threads without providing multiple independent instruction units. In other embodiments, single-instruction, multiple-thread (SIMT) techniques are used to support parallel execution of a large number of generally synchronized threads, using a common instruction unit configured to issue instructions to a set of processing engines within each one of the GPCs 208. Unlike a SIMD execution regime, where all processing engines typically execute identical instructions, SIMT execution allows different threads to more readily follow divergent execution paths through a given thread program. Persons skilled in the art will understand that a SIMD processing regime represents a functional subset of a SIMT processing regime.

> 
图3A是根据本发明一个实施例的、图2中PPU 202之一内的GPC 208的框图。每个GPC 208可被配置为并行执行大量线程，其中术语“线程”指的是特定程序在特定输入数据集上执行指令的实例。在一些实施例中，采用单指令多数据（SIMD）指令发射技术来支持大量线程的并行执行，而无需提供多个独立的指令单元。在其他实施例中，采用单指令多线程（SIMT）技术来支持大量基本同步的线程的并行执行，使用一个公共指令单元，该指令单元被配置为向每个GPC 208内的一组处理引擎发射指令。与所有处理引擎通常执行相同指令的SIMD执行模式不同，SIMT执行允许不同线程更容易地沿着给定线程程序的发散执行路径执行。本领域技术人员将理解，SIMD处理机制代表SIMT处理机制的一个功能子集。




Operation of GPC 208 is advantageously controlled via a 50 pipeline manager 305 that distributes processing tasks to multithreaded SIMT processors called streaming multiprocessors (SPMs) 310 or simply parallel thread processors. Pipeline manager 305 may also be configured to control a work distribution crossbar 330 by specifying destinations for processed data output by SPMs 310.

> 
GPC 208的操作有利地通过管线管理器305来控制，该管线管理器将处理任务分发给被称为流多处理器（SPM）310的多线程SIMT处理器或简称为并行线程处理器。管线管理器305还可配置为通过指定SPM 310输出的处理后数据的目的地来控制工作分发交叉开关330。




In one embodiment, each GPC 208 includes a number M of SPMs 310, where M≥1, each SPM 310 configured to process one or more thread groups. Also, each SPM 310 advantageously includes an identical set of functional execution units

> 
在一个实施例中，每个GPC 208包含M个SPM 310，其中M≥1，每个SPM 310被配置为处理一个或多个线程组。此外，每个SPM 310有利地包含一组相同的功能执行单元




(e.g., arithmetic logic units, and load-store units, shown as Exec units 302 and LSUs 303 in FIG. 3C) that may be pipelined, allowing a new instruction to be issued before a previous instruction has finished, as is known in the art. Any combination of functional execution units may be provided.

> 
（例如，算术逻辑单元和加载存储单元，在图3C中显示为执行单元302和LSU 303）可以采用流水线方式，允许在前一条指令完成之前发出新指令，如本领域已知的那样。可以提供功能执行单元的任何组合。




In one embodiment, the functional units support a variety of operations including integer and floating point arithmetic (e.g., addition and multiplication), comparison operations,

> 
在一个实施方案中，功能单元支持多种运算，包括整数和浮点算术运算（例如加法和乘法）、比较运算、




9

> 
9




Boolean operations (AND, OR, XOR), bit-shifting, and computation of various algebraic functions (e.g., planar interpolation, trigonometric, exponential, and logarithmic functions, etc.); and the same functional-unit hardware can be leveraged to perform different operations.

> 
布尔运算（AND、OR、XOR）、位移操作，以及各种代数函数的计算（例如平面插值、三角函数、指数函数和对数函数等）；并且可以利用相同的功能单元硬件来执行不同的操作。




As previously defined herein, a thread is an instance of a particular program executing instructions on a particular set of input data. The collection of a certain number of concurrently executing threads across the parallel processing engines (not shown) within an SPM 310 is referred to herein as a "warp" or "thread group." As used herein, a "thread group" refers to a group of threads concurrently executing the same program on different input data, with one thread of the group being assigned to a different processing engine within an SPM 310. A thread group may include fewer threads than the number of processing engines within the SPM 310, in which case some processing engines will be idle during cycles when that thread group is being processed. A thread group may also include more threads than the number of processing engines within the SPM 310, in which case processing will take place over consecutive clock cycles. Since each SPM 310 can support up to G thread groups concurrently, it follows that up to G*M thread groups can be executing in GPC 208 at any given time.

> 
如前所述，线程是指特定程序在特定输入数据集上执行指令的一个实例。在单个SPM 310内的各个并行处理引擎（未示出）上，一定数量的并发执行线程的集合，在本文中称为“束（warp）”或“线程组”。本文中，“线程组”是指在不同的输入数据上并发执行同一程序的一组线程，该组中的每个线程分别分配到SPM 310内的一个不同处理引擎。一个线程组所包含的线程数可能少于SPM 310内的处理引擎数量，此时在该线程组被处理的时钟周期内，部分处理引擎将处于空闲状态。一个线程组包含的线程数也可能多于SPM 310内的处理引擎数量，此时处理过程将跨越多个连续的时钟周期完成。由于每个SPM 310最多可同时支持G个线程组，因此，在任意给定时刻，GPC 208中最多可同时有G*M个线程组在执行。




Additionally, a plurality of related thread groups may be active (in different phases of execution) at the same time within an SPM 310. This collection of thread groups is referred to herein as a "cooperative thread array" ("CTA") or "thread array." The size of a particular CTA is equal to m*k, where $\mathrm{k}$ is the number of concurrently executing threads in a thread group and is typically an integer multiple of the number of parallel processing engines within the SPM 310, and m is the number of thread groups simultaneously active within the SPM 310. The size of a CTA is generally determined by the programmer and the amount of hardware resources, such as memory or registers, available to the CTA.

> 
此外，多个相关的线程组可以同时在一个 SPM 310 内处于活动状态（处于不同的执行阶段）。本文中将这种线程组的集合称为“协作线程阵列”（“CTA”）或“线程阵列”。一个特定 CTA 的大小等于 m*k，其中 $\mathrm{k}$ 是一个线程组中并发执行的线程数，且通常是 SPM 310 内并行处理引擎数量的整数倍，而 m 是 SPM 310 内同时活动的线程组数量。CTA 的大小通常由程序员以及 CTA 可用的硬件资源（如内存或寄存器）数量决定。




Each SPM 310 contains an L1 cache (320 in FIG. 3C) or uses space in a corresponding L1 cache outside of the SPM 310 that is used to perform load and store operations. Each SPM 310 also has access to L2 caches within the partition units 215 that are shared among all GPCs 208 and may be used to transfer data between threads. Finally, SPMs 310 also have access to off-chip "global" memory, which can include, e.g., parallel processing memory 204 and/or system memory 104. It is to be understood that any memory external to PPU 202 may be used as global memory. Additionally, an L1.5 cache 335 may be included within the GPC 208, configured to receive and hold data fetched from memory via memory interface 214 requested by SPM 310, including instructions, uniform data, and constant data, and provide the requested data to SPM 310. Embodiments having multiple SPMs 310 in GPC 208 beneficially share common instructions and data cached in L1.5 cache 335.

> 
每个 SPM 310 包含一个 L1 缓存（图 3C 中的 320），或使用 SPM 310 外部相应 L1 缓存中用于执行加载和存储操作的空间。每个 SPM 310 还可以访问分区单元 215 内的 L2 缓存，这些 L2 缓存在所有 GPC 208 之间共享，并可用于在线程之间传输数据。最后，SPM 310 还可以访问片外“全局”内存，例如可包括并行处理内存 204 和/或系统内存 104。应当理解，PPU 202 外部的任何内存都可用作全局内存。此外，GPC 208 内可包含一个 L1.5 缓存 335，配置为接收并保存 SPM 310 通过内存接口 214 从内存请求获取的数据，包括指令、统一数据和常量数据，并将所请求的数据提供给 SPM 310。在 GPC 208 中具有多个 SPM 310 的实施例有益地共享 L1.5 缓存 335 中缓存的公共指令和数据。




Each GPC 208 may include a memory management unit (MMU) 328 that is configured to map virtual addresses into physical addresses. In other embodiments, MMU(s) 328 may reside within the memory interface 214. The MMU 328 includes a set of page table entries (PTEs) used to map a virtual address to a physical address of a tile and optionally a cache line index. The MMU 328 may include address translation lookaside buffers (TLB) or caches which may reside within multiprocessor SPM 310 or the L1 cache or GPC 208. The physical address is processed to distribute surface data access locality to allow efficient request interleaving among partition units. The cache line index may be used to determine whether of not a request for a cache line is a hit or miss.

> 
每个GPC 208可包括存储器管理单元（MMU）328，其被配置为将虚拟地址映射为物理地址。在其他实施例中，MMU 328可位于存储器接口214内。MMU 328包括一组页表条目（PTE），用于将虚拟地址映射到图块的物理地址以及可选地映射到缓存行索引。MMU 328可包括地址转换旁路缓冲器（TLB）或缓存，这些可位于多处理器SPM 310、一级缓存或GPC 208内。处理物理地址以分配表面数据访问局部性，从而允许在分区单元之间进行高效的请求交织。缓存行索引可用于确定对缓存行的请求是命中还是未命中。




## 10

In graphics and computing applications, a GPC 208 may be configured such that each SPM 310 is coupled to a texture unit 315 for performing texture mapping operations, e.g., determining texture sample positions, reading texture data, and 5 filtering the texture data. Texture data is read from an internal texture L1 cache (not shown) or in some embodiments from the L1 cache within SPM 310 and is fetched from an L2 cache, parallel processing memory 204, or system memory 104, as needed. Each SPM 310 outputs processed tasks to

> 
在图形和计算应用中，GPC 208可以被配置为使得每个SPM 310耦合到纹理单元315，用于执行纹理映射操作，例如确定纹理样本位置、读取纹理数据以及5过滤纹理数据。纹理数据从内部纹理L1缓存（未示出）读取，或者在一些实施例中从SPM 310内的L1缓存读取，并根据需要从L2缓存、并行处理存储器204或系统存储器104获取。每个SPM 310将处理后的任务输出到




10 work distribution crossbar 330 in order to provide the processed task to another GPC 208 for further processing or to store the processed task in an L2 cache, parallel processing memory 204, or system memory 104 via crossbar unit 210. A preROP (pre-raster operations) 325 is configured to receive 15 data from SPM 310, direct data to ROP units within partition units 215, and perform optimizations for color blending, organize pixel color data, and perform address translations.

> 
10 工作分发交叉开关330，用于将处理后的任务提供给另一个GPC 208进行进一步处理，或将该处理后的任务经由交叉开关单元210存储在L2缓存、并行处理存储器204或系统存储器104中。预光栅操作单元（preROP）325被配置为从SPM 310接收15数据，将数据引导至分区单元215内的ROP单元，并执行颜色混合优化、组织像素颜色数据以及进行地址转换。




It will be appreciated that the core architecture described herein is illustrative and that variations and modifications are 20 possible. Any number of processing units, e.g., SPMs 310 or texture units 315, preROPs 325 may be included within a GPC 208. Further, while only one GPC 208 is shown, a PPU 202 may include any number of GPCs 208 that are advantageously functionally similar to one another so that execution

> 
应当理解，本文所描述的核心架构是说明性的，并且可以进行变化和修改。任何数量的处理单元，例如 SPM 310 或纹理单元 315、preROP 325，都可以包含在 GPC 208 内。此外，虽然仅示出了一个 GPC 208，但 PPU 202 可以包括任意数量的 GPC 208，这些 GPC 有利地在功能上彼此相似，从而使得执行




25 behavior does not depend on which GPC 208 receives a particular processing task. Further, each GPC 208 advantageously operates independently of other GPCs 208 using separate and distinct processing units, L1 caches, and so on.

> 
25 行为并不取决于哪个 GPC 208 接收特定处理任务。此外，每个 GPC 208 有优势地独立于其他 GPC 208 运行，使用分开且不同的处理单元、L1 缓存等。




FIG. 3B is a block diagram of a partition unit 215 within 30 one of the PPUs 202 of FIG. 2, according to one embodiment of the present invention. As shown, partition unit 215 includes a L2 cache 350, a frame buffer (FB) DRAM interface 355, and a raster operations unit (ROP) 360. L2 cache 350 is a read/ write cache that is configured to perform load and store opera-

> 
图3B是图2的一个PPU 202内的分区单元215的框图，其中30一个PPU，根据本发明的一个实施例。如图所示，分区单元215包括L2缓存350、帧缓冲区（FB）DRAM接口355和光栅操作单元（ROP）360。L2缓存350是一个读/写缓存，被配置为执行加载和存储操作-




tions received from crossbar unit 210 and ROP 360. Read

> 
来自纵横开关单元210和ROP 360的请求。读取




misses and urgent writeback requests are output by L2 cache

> 
L2 缓存输出未命中和紧急写回请求




350 to FB DRAM interface 355 for processing. Dirty updates

> 
350到FB DRAM接口355进行处理。脏更新




are also sent to FB 355 for opportunistic processing. FB 355

> 
也被发送到 FB 355 进行机会性处理。FB 355




interfaces directly with DRAM 220, outputting read and write

> 
直接与 DRAM 220 接口，输出读和写




requests and receiving data read from DRAM 220.

> 
请求和接收从 DRAM 220 读取的数据。




In graphics applications, ROP 360 is a processing unit that performs raster operations, such as stencil, z test, blending, and the like, and outputs pixel data as processed graphics data for storage in graphics memory. In some embodiments of the present invention, ROP 360 is included within each GPC 208 instead of partition unit 215, and pixel read and write requests are transmitted over crossbar unit 210 instead of pixel fragment data.

> 
在图形应用程序中，ROP 360 是一个处理单元，执行光栅操作，例如模板测试、深度测试、混合等，并输出像素数据作为处理后的图形数据，以存储在图形内存中。在本发明的一些实施例中，ROP 360 包含在每个 GPC 208 而不是分区单元 215 中，并且像素读取和写入请求通过交叉开关单元 210 传输，而不是像素片段数据。




The processed graphics data may be displayed on display 50 device 110 or routed for further processing by CPU 102 or by one of the processing entities within parallel processing subsystem 112. Each partition unit 215 includes a ROP 360 in order to distribute processing of the raster operations. In some embodiments, ROP 360 may be configured to compress z or 5 color data that is written to memory and decompress z or color data that is read from memory.

> 
处理后的图形数据可以显示在显示设备110上，或者被路由到CPU 102或并行处理子系统112内的一个处理实体进行进一步处理。每个分区单元215包含一个ROP 360，以便分配光栅化操作的处理。在一些实施例中，ROP 360可被配置为压缩写入内存的z或颜色数据，并解压从内存读取的z或颜色数据。




Persons skilled in the art will understand that the architecture described in FIGS. 1, 2, 3A, and 3B in no way limits the scope of the present invention and that the techniques taught herein may be implemented on any properly configured processing unit, including, without limitation, one or more CPUs, one or more multi-core CPUs, one or more PPUs 202, one or more GPCs 208, one or more graphics or special purpose processing units, or the like, without departing the 5 scope of the present invention.

> 
本领域技术人员将理解，图1、图2、图3A和图3B中描述的架构决不限制本发明的范围，并且本文教导的技术可以在任何适当配置的处理单元上实现，包括但不限于一个或多个CPU、一个或多个多核CPU、一个或多个PPU 202、一个或多个GPC 208、一个或多个图形或专用处理单元等，而不脱离本发明的5范围。




In embodiments of the present invention, it is desirable to use PPU 122 or other processor(s) of a computing system to

> 
在本发明的实施例中，期望使用计算系统的PPU 122或其他处理器来




11

> 
本专利提出了一种技术，用于提高并行处理架构中内存屏障操作的效率。主要研究问题是如何降低代价高昂的内存屏障指令对性能的影响，这些指令用于在多个协作线程组之间对内存事务进行排序。

关键贡献在于一种**N路内存屏障操作合并**方法。当某个线程组接收到第一条内存屏障指令时，该组后续内存操作的执行将被暂停。系统不会立即将该内存屏障发送到内存系统，而是开启一个合并窗口。在此时间窗口内到达的、来自不同线程组的后续内存屏障指令会与第一条进行组合（合并），生成单一的合并内存屏障。系统构造一个带标记的内存命令流，其中每个内存事务及合并屏障都与一个合并索引关联。这使得多个事务集合及其合并屏障可以同时处于执行中。在处理一个合并屏障时，仅会阻塞受影响的线程组，而其他独立线程组的内存操作可以继续执行。

主要结论是，该技术显著降低了内存屏障的开销。通过合并来自多个线程组的屏障，并重叠处理不同的合并集合，系统将单个屏障的长延迟分摊到多个线程上，同时防止未参与同步的线程组不必要的停顿，从而提高了整体指令处理吞吐量。




execute general-purpose computations using thread arrays. Each thread in the thread array is assigned a unique thread identifier ("thread ID") that is accessible to the thread during its execution. The thread ID, which can be defined as a one-dimensional or multi-dimensional numerical value controls various aspects of the thread's processing behavior. For instance, a thread ID may be used to determine which portion of the input data set a thread is to process and/or to determine which portion of an output data set a thread is to produce or write.

> 
使用线程数组执行通用计算。线程数组中的每个线程都被分配一个唯一的线程标识符（"线程 ID"），该标识符在线程执行期间可被线程访问。线程 ID 可以定义为一维或多维数值，控制着线程处理行为的各个方面。例如，线程 ID 可用于确定线程要处理输入数据集的哪一部分，和/或确定线程要生成或写入输出数据集的哪一部分。




A sequence of per-thread instructions may include at least one instruction that defines a cooperative behavior between the representative thread and one or more other threads of the thread array. For example, the sequence of per-thread instructions might include an instruction to suspend execution of operations for the representative thread at a particular point in the sequence until such time as one or more of the other threads reach that particular point, an instruction for the representative thread to store data in a shared memory to which one or more of the other threads have access, an instruction for the representative thread to atomically read and update data stored in a shared memory to which one or more of the other threads have access based on their thread IDs, or the like. The CTA program can also include an instruction to compute an address in the shared memory from which data is to be read, with the address being a function of thread ID. By defining suitable functions and providing synchronization techniques, data can be written to a given location in shared memory by one thread of a CTA and read from that location by a different thread of the same CTA in a predictable manner. Consequently, any desired pattern of data sharing among threads can be supported, and any thread in a CTA can share data with any other thread in the same CTA. The extent, if any, of data sharing among threads of a CTA is determined by the CTA program; thus, it is to be understood that in a particular application that uses CTAs, the threads of a CTA might or might not actually share data with each other, depending on the CTA program, and the terms "CTA" and "thread array" are used synonymously herein.

> 
每个线程的指令序列可能包含至少一条指令，该指令定义了代表线程与线程阵列中的一个或多个其他线程之间的协作行为。例如，该每线程指令序列可能包括：在序列的特定点暂停执行代表线程的操作，直到一个或多个其他线程到达该特定点的指令；代表线程将数据存储到可供一个或多个其他线程访问的共享内存中的指令；代表线程基于线程ID原子地读取和更新存储在共享内存中的数据（该共享内存可供一个或多个其他线程访问）的指令，或类似指令。CTA程序还可包含一条指令，用于计算共享内存中待读取数据的地址，该地址是线程ID的函数。通过定义合适的函数并提供同步技术，数据可由CTA的一个线程写入共享内存的给定位置，并由同一CTA的另一线程以可预测的方式从该位置读取。因此，可以支持线程间任意所需的数据共享模式，且CTA中的任何线程都可以与同一CTA中的任何其他线程共享数据。CTA中线程间数据共享的程度（如果有的话）由CTA程序决定；因此，应理解，在使用CTA的特定应用中，根据CTA程序，CTA中的线程实际上可能共享数据，也可能不共享数据，而术语“CTA”和“线程阵列”在本文中可互换使用。




FIG. 3C is a block diagram of the SPM 310 of FIG. 3A, according to one embodiment of the present invention. The SPM 310 includes an instruction L1 cache 370 that is configured to receive instructions and constants from memory via L1.5 cache 335. A warp scheduler and instruction unit 312 receives instructions and constants from the instruction L1 cache 370 and controls local register file 304 and SPM 310 functional units according to the instructions and constants. The SPM 310 functional units include N exec (execution or processing) units 302 and P load-store units (LSU) 303.

> 
图3C是根据本发明一个实施例的图3A的SPM 310的框图。SPM 310包括指令L1高速缓存370，其被配置为经由L1.5高速缓存335从存储器接收指令和常量。线程束调度器和指令单元312从指令L1高速缓存370接收指令和常量，并根据这些指令和常量控制本地寄存器文件304和SPM 310的功能单元。SPM 310的功能单元包括N个exec（执行或处理）单元302和P个加载存储单元（LSU）303。




SPM 310 provides on-chip (internal) data storage with different levels of accessibility. Special registers (not shown) are readable but not writeable by LSU 303 and are used to store parameters defining each CTA thread's "position." In one embodiment, special registers include one register per CTA thread (or per exec unit 302 within SPM 310) that stores a thread ID; each thread ID register is accessible only by a respective one of the exec unit 302. Special registers may also include additional registers, readable by all CTA threads (or by all LSUs 303) that store a CTA identifier, the CTA dimensions, the dimensions of a grid to which the CTA belongs, and an identifier of a grid to which the CTA belongs. Special registers are written during initialization in response to commands received via front end 212 from device driver 103 and do not change during CTA execution.

> 
SPM 310 提供具有不同访问级别的片上（内部）数据存储。特殊寄存器（未示出）可由 LSU 303 读取但不可写入，用于存储定义每个 CTA 线程的“位置”的参数。在一个实施例中，特殊寄存器包括每个 CTA 线程（或 SPM 310 内的每个执行单元 302）一个寄存器，该寄存器存储线程 ID；每个线程 ID 寄存器只能由相应的一个执行单元 302 访问。特殊寄存器还可以包括附加寄存器，可由所有 CTA 线程（或所有 LSU 303）读取，这些寄存器存储 CTA 标识符、CTA 维度、CTA 所属网格的维度，以及 CTA 所属网格的标识符。特殊寄存器在初始化期间响应于经由前端 212 从设备驱动程序 103 接收的命令而被写入，并且在 CTA 执行期间不改变。




A parameter memory (not shown) stores runtime parameters (constants) that can be read but not written by any CTA thread (or any LSU 303). In one embodiment, device driver

> 
参数存储器（未示出）存储可由任何 CTA 线程（或任何 LSU 303）读取但不能写入的运行时参数（常量）。在一个实施例中，设备驱动程序




## 12

103 provides parameters to the parameter memory before directing SPM 310 to begin execution of a CTA that uses these parameters. Any CTA thread within any CTA (or any exec unit 302 within SPM 310) can access global memory through a memory interface 214. Portions of global memory may be stored in the L1 cache 320.

> 
103 先将参数提供给参数内存，然后引导 SPM 310 开始执行使用这些参数的 CTA。任何 CTA 内的任何 CTA 线程（或 SPM 310 内的任何执行单元 302）都可以通过内存接口 214 访问全局内存。部分全局内存可以存储在 L1 缓存 320 中。




Local register file 304 is used by each CTA thread as scratch space; each register is allocated for the exclusive use of one thread, and data in any of local register file 304 is 10 accessible only to the CTA thread to which it is allocated. Local register file 304 can be implemented as a register file that is physically or logically divided into P lanes, each having some number of entries (where each entry might store, e.g., a 32-bit word). One lane is assigned to each of the N exec

> 
每个 CTA 线程将本地寄存器文件 304 用作暂存空间；每个寄存器被分配供一个线程专用，本地寄存器文件 304 中的任何数据仅可由 10 分配给它的 CTA 线程访问。本地寄存器文件 304 可实现为在物理或逻辑上划分为 P 个通道的寄存器文件，每个通道具有若干条目（其中每个条目可存储例如 32 位字）。每个 N exec 分配一个通道。




units 302 and P load-store units LSU 303, and corresponding entries in different lanes can be populated with data for different threads executing the same program to facilitate SIMT or SIMD execution. Different portions of the lanes can be allocated to different ones of the G concurrent thread groups, so that a given entry in the local register file 304 is accessible only to a particular thread. In one embodiment, certain entries within the local register file 304 are reserved for storing thread identifiers, implementing one of the special registers.

> 
单元302和P个加载存储单元LSU 303，并且不同通道中的对应条目可以填充执行相同程序的不同线程的数据，以促进SIMT或SIMD执行。通道的不同部分可以分配给G个并发线程组中的不同组，使得本地寄存器文件304中的给定条目仅对特定线程可访问。在一个实施例中，本地寄存器文件304中的某些条目被保留用于存储线程标识符，实现其中一个特殊寄存器。




Shared memory 306 is accessible to all CTA threads 5 (within a single CTA); any location in shared memory 306 is accessible to any CTA thread within the same CTA (or to any processing engine within SPM 310). Shared memory 306 can be implemented as a shared register file or shared on-chip cache memory with an interconnect that allows any processing engine to read from or write to any location in the shared memory. In other embodiments, shared state space might map onto a per-CTA region of off-chip memory, and be cached in L1 cache 320. The parameter memory can be implemented as a designated section within the same shared register file or shared cache memory that implements shared memory 306, or as a separate shared register file or on-chip cache memory to which the LSUs 303 have read-only access. In one embodiment, the area that implements the parameter memory is also used to store the CTA ID and grid ID, as well as CTA and grid dimensions, implementing portions of the special registers. Each LSU 303 in SPM 310 is coupled to a unified address mapping unit 352 that converts an address provided for load and store instructions that are specified in a unified memory space into an address in each distinct memory space. Consequently, an instruction may be used to access any of the local, shared, or global memory spaces by specifying an address in the unified memory space.

> 
共享内存 306 可被（单个 CTA 内的）所有 CTA 线程 5 访问；共享内存 306 中的任何位置都可被同一 CTA 内的任何 CTA 线程（或 SPM 310 内的任何处理引擎）访问。共享内存 306 可以实现为共享寄存器堆或共享片上缓存，并配有一个允许任何处理引擎读取或写入共享内存中任何位置的互连网络。在其他实施例中，共享状态空间可以映射到片外内存的每个 CTA 区域，并缓存在 L1 缓存 320 中。参数内存可以实现为实现了共享内存 306 的同一共享寄存器堆或共享缓存内的指定部分，或者作为一个单独的共享寄存器堆或片上缓存，LSU 303 对此具有只读访问权限。在一个实施例中，实现了参数内存的区域也用于存储 CTA ID 和网格 ID，以及 CTA 和网格维度，从而实现了部分特殊寄存器。SPM 310 中的每个 LSU 303 都耦合到一个统一地址映射单元 352，该单元将针对在统一内存空间中指定的加载和存储指令所提供的地址，转换为每个不同内存空间中的地址。因此，可以通过在统一内存空间中指定地址，使用一条指令来访问本地、共享或全局内存空间中的任何一个。




The L1 Cache 320 in each SPM 310 can be used to cache private per-thread local data and also per-application global 50 data. In some embodiments, the per-CTA shared data may be cached in the L1 cache 320. The LSUs 303 are coupled to a uniform L1 cache 371, the shared memory 306, and the L1 cache 320 via a memory and cache interconnect 380. The uniform L1 cache 371 is configured to receive read-only data and constants from memory via the L1.5 Cache 335. In one embodiment the L1 cache 320 is included within the LSUs 303.

> 
每个 SPM 310 中的 L1 Cache 320 可用于缓存线程私有的本地数据以及每个应用程序的全局50数据。在一些实施例中，每个 CTA 的共享数据可能被缓存在 L1 cache 320 中。LSUs 303 通过内存与缓存互连 380 连接到统一 L1 cache 371、共享内存 306 和 L1 cache 320。统一 L1 cache 371 被配置为通过 L1.5 Cache 335 从内存接收只读数据和常量。在一个实施例中，L1 cache 320 包含在 LSUs 303 内。




## Coalescing Memory Barrier Operations

The computing system 100 provides a many-core high performance compute platform for academic research, commercial, and consumer applications across a broad range of problem spaces. Among key components of the architecture are the memory hierarchy that supports accesses to parallel processing memory (DRAM) and system memory and the SPM 310 that supports the simultaneous scheduling and

> 
计算系统 100 提供了一个众核高性能计算平台，适用于学术研究、商业和消费应用等广泛的问题领域。该架构的关键组件包括支持访问并行处理内存（DRAM）和系统内存的内存层次结构，以及支持同时调度和……的 SPM 310。




13 execution of multiple threads in a CTA. In one embodiment, up to 1024 threads are included in a CTA, where 32 threads are collected into an execution unit called a warp, as previously defined herein. All active threads within the warp execute the same instruction but with independent address, data, register, and control state. Memory operations must be managed carefully in this SIMT environment to ensure correct program behavior.

> 
13 在一个CTA中执行多个线程。在一个实施例中，一个CTA最多包含1024个线程，其中32个线程被组织成一个称为warp的执行单元，如本文先前所定义。warp中的所有活跃线程执行相同的指令，但具有独立的地址、数据、寄存器和控制状态。在这种SIMT环境下，必须谨慎管理内存操作，以确保正确的程序行为。




A relaxed memory ordering model is used that allows flexibility in how memory operations are issued, accepted, and ordered throughout the system. More specifically, memory operations can be performed in any order except with respect to LOAD and STORE operations from the same thread to the same memory address. LOAD and STORE operations from any one thread to the same memory address must be performed with respect to just that thread in program order of those LOAD and STORE operations. This flexibility allows for increased performance in general, but correct program execution may require certain points in memory transactions around which sequential order is guaranteed. In these cases, a memory barrier (MEMBAR) instruction is used to ensure that all memory transactions issued before the MEM-BAR instruction are sufficiently performed so that their results are visible to any memory transactions issued after the MEMBAR instruction.

> 
采用一种宽松的内存排序模型，使得内存操作在整个系统中的发出、接受和排序方式具有灵活性。更具体地说，除了同一线程对同一内存地址的 LOAD 和 STORE 操作外，内存操作可以按任意顺序执行。来自任一线程对同一内存地址的 LOAD 和 STORE 操作，必须仅就该线程而言，按照这些 LOAD 和 STORE 操作的程序顺序执行。这种灵活性通常会带来性能提升，但正确的程序执行可能需要在内存事务的某些点确保顺序性。在这些情况下，使用内存屏障（MEMBAR）指令来确保在 MEM-BAR 指令之前发出的所有内存事务都已充分执行，使得它们的结果对 MEMBAR 指令之后发出的任何内存事务可见。




From the standpoint of a single thread running alone, memory operations to a given address must appear to be performed in program order. This matches normal C program semantics, and is necessary for the CUDA programming model. Once multiple threads are involved, memory ordering becomes more complex, and must be defined in terms of when a memory transaction is "performed", and thus visible to other threads and memory clients.

> 
从单个线程独立运行的角度来看，对给定地址的内存操作必须看起来按程序顺序执行。这与普通的 C 程序语义一致，也是 CUDA 编程模型所必需的。一旦涉及多个线程，内存排序就变得更加复杂，必须根据内存事务何时被“执行”并因此对其他线程和内存客户端可见来定义。




A memory transaction, such as a load or store operation is defined as being performed based on the following definition, 35 "A request is initiated when a processor has sent the request and the completion of the request is out of its control. An initiated request is issued when it has left the processor environment and is in transit in the memory system. A LOAD by processor I is considered performed with respect to processor K at a point in time when the issuing of a STORE to the same address by processor K cannot affect the value returned to processor I. A STORE by processor I is considered performed with respect to processor $\mathrm{K}$ , at a point in time when an issued LOAD to the same address by processor $\mathrm{K}$ returns the value defined by the STORE. An access by processor I is performed when it is performed with respect to all processors." (see Dubois, M., Scheurich, C., & Briggs, F. (1986). Memory access buffering in multiprocessors. In Proceedings of the 13th annual International Symposium on Computer Architecture (pp. 434-442). ACM)

> 
诸如加载或存储操作的内存事务，其执行定义基于以下定义[35]：“当处理器发送请求且请求的完成已超出其控制时，即发起请求。发起的请求在离开处理器环境并进入内存系统传输时被发出。处理器 I 的加载操作，当处理器 K 对同一地址的存储操作发出后不再影响返回给处理器 I 的值时，即视为相对于处理器 K 已执行。处理器 I 的存储操作，当处理器 $\mathrm{K}$ 对同一地址的加载操作发出后返回该存储操作所定义的值时，即视为相对于处理器 $\mathrm{K}$ 已执行。处理器 I 的访问操作，当其相对于所有处理器均已执行时，即视为已执行。”（参见 Dubois, M., Scheurich, C., & Briggs, F. (1986). Memory access buffering in multiprocessors. In Proceedings of the 13th annual International Symposium on Computer Architecture (pp. 434-442). ACM）




The programming model used by the computer system100 recognizes three levels of affinity for memory clients: threads in the same CTA ("CTA" level), threads and other clients in the same PPU 202 ("global" level), and all threads and clients with access to the same memory in the computer system 100, including the host CPU 102 and peer PPUs 202 ("system" level). Other embodiments may support other affinity levels for MEMBAR instructions, including a thread (self) affinity level, and a warp affinity level (the set of threads that execute a SIMT or SIMD instruction together). In the context of the computer system 100, a memory transaction is considered "performed" when it has been committed to memory order and is visible to all other threads and clients at the indicated level of affinity. For example, a load (LD) by a first thread in a CTA is considered "performed" at the CTA level with respect to other threads in a CTA at a point in time when the

> 
计算机系统 100 所使用的编程模型为内存客户端识别三个层次的亲和性：同一 CTA 内的线程（“CTA”级别），同一 PPU 202 内的线程和其他客户端（“全局”级别），以及计算机系统 100 中可访问同一内存的所有线程和客户端，包括主机 CPU 102 和对等 PPU 202（“系统”级别）。其他实施例可能支持 MEMBAR 指令的其他亲和性级别，包括线程（自身）亲和性级别和 warp 亲和性级别（一同执行 SIMT 或 SIMD 指令的线程集合）。在计算机系统 100 的语境中，当一次内存事务已提交到内存排序并对所指示的亲和性级别上的所有其他线程和客户端可见时，该事务即被视为“已执行”。例如，CTA 中第一个线程发出的一条加载（LD）指令，在相对于该 CTA 中其他线程而言被认为在 CTA 级别上“已执行”的时间点是当




## 14

issuing of a store (ST) to the same address by one of the other threads in the CTA cannot affect the value returned to the first thread. In another example, a store (ST) by the first thread in a CTA is considered "performed" at the CTA level at a point in time when an issued LD to the same address by another thread in the CTA returns the value defined by the STORE; threads that are not in the same CTA may or may not see the result of the store by the first thread. In general, it is faster and less expensive to perform memory operations at the lower 10 affinity levels of visibility. In one embodiment, the CTA affinity level is the lowest level is the lowest affinity level and the system affinity level is the highest affinity level. In other embodiments, the thread or warp affinity level is the lowest level.

> 
同一CTA中的其他线程对同一地址发出存储（ST）不会影响返回给第一个线程的值。在另一个示例中，当CTA中的第一个线程执行一条存储（ST）指令后，在某个时间点，若该CTA内另一线程对同一地址发出的加载（LD）返回了该存储所定义的值，便认为此存储在该CTA层级“已完成”；而不在同一CTA中的线程可能会、也可能不会观察到该存储的结果。一般而言，在较低的10级亲和性可见性层级上执行内存操作更快且开销更低。在一个实施例中，CTA亲和性层级是最低的亲和性层级，而系统亲和性层级是最高的亲和性层级。在其他实施例中，线程或经线（warp）亲和性层级为最低层级。




In this discussion, the term "load" is used to describe a class of instructions that read and return a value from memory, while "store" describes instructions that write a value to memory. Some instructions, such as atomic and locking operations, read and modify memory and return val- 20 ues, and thus should be considered to have both load and store semantics, and thus follow both load and store ordering rules.

> 
在本讨论中，术语“load”（加载）用于描述一类从内存读取并返回值的指令，而“store”（存储）则描述将值写入内存的指令。某些指令，例如原子操作和锁定操作，会读取并修改内存并返回 val- 20 ues，因此应被视为同时具有加载和存储语义，从而需要遵循加载和存储两者的排序规则。




There are many definitions and ordering rules for the overall memory consistency model. Memory ordering rules specific to MEMBAR operations are defined in terms of two 25 orders: program order and dependence order. Program order requires that memory operations are performed in the exact sequential order as the instructions are in the program. Dependence order is a partial ordering that describes the constraints that hold between instructions in a thread that access the same register or memory location. This covers data dependencies, such as values passed through scoreboarded resources such as the register file, condition code register, or predicate registers; and also includes control dependencies, such as a store following a conditional branch.

> 
针对整体内存一致性模型，存在诸多定义与排序规则。具体到 MEMBAR 操作的排序规则，则依据两种顺序加以定义：程序顺序与依赖顺序。程序顺序要求，访存操作须严格按程序中指令的顺序执行。依赖顺序是一种偏序关系，描述了同一线程内访问相同寄存器或内存位置的指令之间的约束。这涵盖了数据依赖，例如通过记分板资源（如寄存器文件、条件码寄存器或谓词寄存器）传递的值；也包括控制依赖，比如条件分支之后的存储操作。




Within the relaxed memory ordering rules, MEMBAR instructions order the performance of memory transactions within a given thread or warp. Memory transactions that occur prior to the MEMBAR instruction in program order will be performed in memory prior to any memory transactions that occur after the MEMBAR instruction in program order.

> 
在宽松内存排序规则下，MEMBAR 指令对给定线程或线程束内的内存事务的执行进行排序。在程序顺序上位于 MEMBAR 指令之前的内存事务，将在内存中先于在程序顺序上位于 MEMBAR 指令之后的任何内存事务执行。




The relaxed memory ordering rules have implications for memory transactions. For example, if one thread stores to two different addresses, another thread could see those stores in any order. To enforce an inter-thread or inter-address order on memory transactions, the program must execute a MEMBAR instruction. MEMBAR effectively inserts a fence in the stream of memory operations, such that operations executed by this thread prior to the MEMBAR are guaranteed to be 0 performed before memory operations executed after the MEMBAR. It is also the responsibility of the reading thread to execute a MEMBAR between load operations that it expects to be performed in a specific order, unless this order is established via other ordering rules such as dependency.

> 
放宽的内存排序规则对内存事务存在影响。例如，若一个线程向两个不同地址执行存储，另一个线程可能以任意顺序观察到这些存储。为了对内存事务强制施加线程间或地址间顺序，程序必须执行 MEMBAR 指令。MEMBAR 实质上在内存操作流中插入了一道栅栏，使得该线程在 MEMBAR 之前执行的操作保证 0 在 MEMBAR 之后执行的内存操作之前完成。读取线程也有责任在其期望按特定顺序执行的加载操作之间执行 MEMBAR，除非该顺序已通过其他排序规则（如依赖关系）建立。




There are multiple levels of MEMBAR instructions that

> 
有多种级别的 MEMBAR 指令，




differ in the scope of other threads that are affected. MEMB-

> 
在受到影响的其他线程的范围上有所不同。MEMB-




AR.CTA enforces memory ordering among threads in the

> 
AR.CTA 在线程间强制实施内存排序，在




CTA, MEMBAR.GL enforces ordering at the global level

> 
CTA，MEMBAR.GL 在全局级别强制执行排序。




(e.g. among the memory interface 214 clients), and MEM-

> 
(例如在内存接口214的客户端中)，和MEM-




BAR.SYS enforces ordering at the system level (e.g. including system and peer memory). The MEMBAR.CTA ensures that all prior memory transactions are committed at a CTA level such that they are visible to all threads in the same CTA, such as the L1 cache 320 level. The MEMBAR.GL ensures

> 
BAR.SYS 在系统级别强制排序（例如包括系统内存和对等内存）。MEMBAR.CTA 确保所有先前的内存事务在 CTA 级别提交，从而对同一 CTA 中的所有线程可见，例如在 L1 缓存 320 级别。MEMBAR.GL 确保




that all prior memory transactions are committed at a global level such that that they are visible to all threads in the same PPU, such as the L2.cache 350 level. The MEMBAR.SYS

> 
确保所有先前的内存事务在全局级别上提交，使得它们对同一 PPU 中的所有线程可见，例如 L2.cache 350 级别。MEMBAR.SYS




## 15

ensures that all prior memory transactions are committed at a system level such that they are visible to all threads and clients in the system.

> 
确保所有先前的内存事务在系统级别被提交，从而使系统中的所有线程和客户端均可对其可见。




These three levels form a hierarchy, and a MEMBAR at any level implies ordering at the lower levels. Thus, MEM-BAR.GL effectively implies a MEMBAR.CTA, and a MEM-BAR.SYS implies MEMBAR.GL. Note that these orderings are defined in terms of threads, and not in terms of memory spaces (e.g., local, shared, and global memory, where global memory includes the DRAM 220 and system memory 104). Specifically, threads within a CTA can communicate through global memory using MEMBAR.CTA to order their transactions, which is typically lower latency than using MEM-BAR.GL. Other embodiments may include additional affinity levels, including MEMBAR.THREAD for ordering transactions within a thread without regard to other threads, and MEMBAR.WARP for ordering transactions among the threads comprising a warp.

> 
这三个级别构成一个层次结构，任何级别的 MEMBAR 都暗示着更低级别的顺序。因此，MEM-BAR.GL 实际上暗示了 MEMBAR.CTA，而 MEM-BAR.SYS 则暗示了 MEMBAR.GL。请注意，这些顺序是根据线程定义的，而不是根据内存空间（例如本地、共享和全局内存，其中全局内存包括 DRAM 220 和系统内存 104）定义的。具体来说，CTA 内的线程可以通过全局内存使用 MEMBAR.CTA 来对其事务进行排序，这通常比使用 MEM-BAR.GL 具有更低的延迟。其他实施例可能包括额外的亲和级别，包括 MEMBAR.THREAD（用于在单个线程内对事务进行排序，而不考虑其他线程）以及 MEMBAR.WARP（用于对构成一个线程束的线程之间的事务进行排序）。




The following memory barrier instruction implements the concepts described above:

> 
以下的内存屏障指令实现了上述概念：




MEMBAR.lvl

> 
MEMBAR.lvl




.1vl: \{.CTA, .GL, .SYS\} CTA, Global, System level MEMBAR Levels:

> 
.1vl: {.CTA, .GL, .SYS} CTA、全局、系统级 MEMBAR 级别:




.CTA CTA thread level

> 
.CTA 线程级别




Waits until all prior memory writes are visible to other 25 threads in the same CTA.

> 
等待直到所有先前的内存写入对同一 CTA 中的其他 25 个线程可见。




Waits until prior memory reads have been performed with respect to other threads in the CTA.

> 
等待，直到先前的内存读取相对于 CTA 中的其他线程已执行完毕。




For communication within a CTA, MEMBAR.CTA is the appropriate type of MEMBAR.

> 
对于 CTA 内部的通信，MEMBAR.CTA 是合适的 MEMBAR 类型。




.GL Global level

> 
.GL 全局级别




Waits until all prior memory requests have been performed with respect to all other threads in the PPU.

> 
等待，直到所有先前的内存请求相对于 PPU 中的所有其他线程都已完成。




For communication between threads in different CTAs, MEMBAR.GL is the appropriate type of MEMBAR.

> 
对于不同 CTA 中的线程间通信，MEMBAR.GL 是适用的内存屏障类型。




MEMBAR.GL will typically be more expensive (longer latency) than MEMBAR.CTA

> 
MEMBAR.GL 通常比 MEMBAR.CTA 开销更大（延迟更长）。




.SYS System level

> 
.SYS 系统级别




Waits until all prior memory requests have been performed with respect to all threads and clients, including the DRAM and those communicating via communication path 113, such as system and peer-to-peer memory.

> 
一直等待，直到所有先前的内存请求针对所有线程和客户端都已执行完毕，包括 DRAM 以及那些通过通信路径 113 进行通信的（如系统内存和对等内存）。




This level of MEMBAR is required to insure performance with respect to a host CPU thread or other system level peers that are coupled to the host CPU 102 via the memory bridge 105.

> 
需要这一级别的内存屏障来确保相对于主机CPU线程或通过内存桥105耦合到主机CPU 102的其他系统级对等体的性能。




Writes to system memory pass through the L2 cache to communication path 113.

> 
对系统内存的写入会经过二级缓存，传递到通信路径 113。




MEMBAR.SYS will typically be much more expensive 5 (longer latency) than MEMBAR.GL

> 
MEMBAR.SYS 通常比 MEMBAR.GL 开销大得多⁵（更长的延迟）




Given the scalable nature of the PPU 202 architecture, the total number of threads on a given SPM 310, and the total number of memory transactions that may be simultaneously in flight across an entire implementation, an MEMBAR operation can be costly to perform. As such, MEMBAR requests from a given SPM 310 are coalesced to reduce to impact to the rest of the system.

> 
鉴于PPU 202架构的可扩展特性、特定SPM 310上的线程总数，以及整个实现中可能同时进行的内存事务总量，执行MEMBAR操作可能代价高昂。因此，来自特定SPM 310的MEMBAR请求会被合并，以减少对系统其余部分的影响。




At the lowest level, MEMBAR operations may be partly coalesced due to per-warp grouping of threads. In other words, the threads within a warp will execute a MEMBAR synchronously. The warp scheduler and instruction unit 312 ultimately knows the execution state of all threads within a CTA and in particular within a warp. For a MEMBAR.CTA, the warp scheduler and instruction unit 312 ensures all prior load/store requests or requests which could otherwise affect the state of memory have been accepted for execution (and

> 
在最底层，MEMBAR 操作可能因每个线程束内的线程分组而部分合并。换句话说，线程束内的线程将同步执行 MEMBAR。线程束调度器和指令单元 312 最终了解 CTA 内所有线程的执行状态，特别是线程束内的线程。对于 MEMBAR.CTA，线程束调度器和指令单元 312 确保所有先前的加载/存储请求或可能以其他方式影响内存状态的请求都已为执行所接受（并




## 16

their order of performance established within the CTA) before allowing subsequent requests. Execution of a MEM-BAR.CTA may be accomplished within the SPM 310. The higher levels of MEMBAR.GL and MEMBAR.SYS require architecture-wide checks, and coalescing and ordering is implemented outside of SPM 310.

> 
在允许后续请求之前，建立其在 CTA 内的执行顺序）。MEM-BAR.CTA 的执行可以在 SPM 310 内完成。更高级别的 MEMBAR.GL 和 MEMBAR.SYS 需要架构范围的检查，并且合并与排序在 SPM 310 外部实现。




The L1 Cache 320 is a first level data cache responsible for memory requests from SM where much of the coalescing is done. The threads within a CTA may naturally execute MEM-BAR.GL or MEMBAR.SYS in a temporally coherent manner. The L1 cache 320 takes advantage of this by coalescing MEMBARs that arrive within a configurable temporal window before sending a single MEMBAR request to the rest of the memory system. The warp scheduler and instruction unit

> 
L1 缓存 320 是负责处理来自 SM 的内存请求的一级数据缓存，大部分合并操作在此完成。一个 CTA 内的线程可能会自然地以时间相干的方式执行 MEM-BAR.GL 或 MEMBAR.SYS。L1 缓存 320 利用这一点，在将单个 MEMBAR 请求发送到内存系统的其余部分之前，合并在可配置的时间窗口内到达的 MEMBAR。warp 调度器和指令单元




312 and the L1 cache 320 communicate throughout this process to establish the following:

> 
312 和 L1 缓存 320 在整个过程中进行通信以建立以下内容：




1) which MEMBARs are accepted for coalescing

> 
1) 哪些 MEMBAR 被接受进行合并




2) which MEMBARs missed the coalescing window and must be deferred to be retried in the future

> 
2) 哪些内存屏障指令未进入合并窗口因而必须推迟到将来重新尝试




3) which MEMBARs are deferred for reasons other than not being accepted as part of a prior MEMBAR.

> 
3）哪些MEMBAR是因为除了未被接受为先前MEMBAR的一部分之外的原因而被延迟的。




4) when all MEMBARs accepted for coalescing are done being executed

> 
4) 当所有被接受用于合并的内存屏障执行完毕时




5) when any MEMBAR that was deferred can be retried

> 
5）当任何被推迟的 MEMBAR 可以重试时




6) hints from the L1 cache 320 to the warp scheduler and instruction unit 312 to indicate preferred types of memory requests that will increase the coalescing efficiency

> 
6) 来自L1高速缓存320的提示，发送给warp调度器与指令单元312，以指示能够提高合并效率的首选存储器请求类型




FIG. 4 is a block diagram of a memory barrier instruction

> 
图4 是内存屏障指令的框图。




30 execution unit 500, according to one embodiment of the present invention. In some embodiments the memory barrier instruction execution unit500is within each L1 cache 320 to coalesce MEMBAR.CTA, MEMBAR.GL, and MEMBAR- .SYS instructions across multiple threads. Another memory

> 
30 执行单元500，根据本发明的一个实施例。在一些实施例中，内存屏障指令执行单元500位于每个L1缓存320内，以跨多个线程合并MEMBAR.CTA、MEMBAR.GL和MEMBAR- .SYS指令。另一个内存




barrier instruction execution unit 500 may be included within each L2 cache 350 to coalesce MEMBAR.GL and MEM-BAR.SYS instructions across multiple threads and yet another memory barrier instruction execution unit 500 may be included within each I/O unit 205 to coalesce MEMBAR- .SYS instructions across multiple threads.

> 
屏障指令执行单元500可以被包括在每个L2缓存350内，以跨多个线程合并MEMBAR.GL和MEM-BAR.SYS指令，而另一个内存屏障指令执行单元500可以被包括在每个I/O单元205内，以跨多个线程合并MEMBAR- .SYS指令。




The warp scheduler and instruction unit 312 includes selection logic 510 that selects a next instruction to issue. Selection logic 510 may be of generally conventional design, and a detailed description is omitted as not being critical to understanding the present invention. A MEMBAR detection circuit 512 that is also included in the warp scheduler and instruction unit 312 receives each selected instruction. The selected instruction may be a MEMBAR instruction that specifies a memory barrier level, e.g., CTA, GL, or SYS.

> 
warp 调度器与指令单元 312 包含选择逻辑 510，用于选择要发射的下一条指令。选择逻辑 510 可采用常规设计，因其对理解本发明并非关键，故省略详细描述。warp 调度器与指令单元 312 中还包含一个 MEMBAR 检测电路 512，它接收每条被选中的指令。所选指令可能是一条 MEMBAR 指令，该指令指定内存屏障级别，例如 CTA、GL 或 SYS。




When the selected instruction is a MEMBAR instruction, MEMBAR detection circuit 512 directs the instruction to the memory barrier instruction execution unit 500; otherwise, MEMBAR detection circuit 512 forwards the instruction to the next stage for eventual delivery to execution units 302. In 55 one embodiment, the MEMBAR.CTA is executed by the MEMBAR detection circuit 512 since a memory barrier at the CTA level commits memory transactions at the L1 cache 320.

> 
当所选指令为 MEMBAR 指令时，MEMBAR 检测电路 512 将该指令定向到内存屏障指令执行单元 500；否则，MEMBAR 检测电路 512 将指令转发至下一阶段，以便最终送达执行单元 302。在 55 一个实施例中，MEMBAR.CTA 由 MEMBAR 检测电路 512 执行，因为 CTA 级别的内存屏障在 L1 缓存 320 处提交内存事务。




Memory barrier instruction execution unit 500 includes a MEMBAR accept/retry unit 505, a coalesce window unit 503, and a MEMBAR tracking unit 515. The MEMBAR accept/ retry unit 505 receives the MEMBAR instruction and determines if the MEMBAR instruction can be accepted. If not, then the MEMBAR accept/retry unit 505 negates an accept signal and discards the MEMBAR instruction. The MEM-

> 
内存屏障指令执行单元500包含一个MEMBAR接受/重试单元505、一个合并窗口单元503以及一个MEMBAR跟踪单元515。MEMBAR接受/重试单元505接收MEMBAR指令并确定该MEMBAR指令是否可被接受。若不可接受，则MEMBAR接受/重试单元505否定接受信号并丢弃该MEMBAR指令。MEM-




BAR accept/retry unit 505 signals via the retry to indicate when a MEMBAR instruction that was not accepted should be retried. When the MEMBAR instruction can be accepted,

> 
BAR 接受/重试单元 505 通过重试信号指示未被接受的 MEMBAR 指令何时应重试。当 MEMBAR 指令可以被接受时，




## 17

the MEMBAR accept/retry unit 505 asserts the accept signal. When a coalesce (temporal) window is closed, then the accepted MEMBAR instruction is the first MEMBAR instruction since either a reset or a previous MEMBAR instruction was executed, and the coalesce window is opened by the coalesce window unit 503. Subsequent MEMBAR instructions are accepted while the coalesce window remains open.

> 
MEMBAR 接受/重试单元 505 断言接受信号。当合并（时间）窗口关闭后，被接受的 MEMBAR 指令即为自复位或前一条 MEMBAR 指令执行以来的第一条 MEMBAR 指令，且合并窗口由合并窗口单元 503 打开。在合并窗口保持打开期间，后续 MEMBAR 指令均被接受。




The duration of the coalescing window is configurable from 0 clocks (disabled) to on the order 1000's of clocks. This allows tuning in the amount of target MEMBAR instructions from typical CTAs to coalesce given the cost for the rest of the system to implement the .SYS or .GL MEMBAR once it leaves the L1 cache 320. When the coalesce window closes, a transition window opens, as described in greater detail in conjunction with FIGS. 5A and 5B. The duration of the transition window is not configurable, since it is governed by communication latencies between the L1 cache 320 and the warp scheduler and instruction unit 312.

> 
合并窗口的持续时间可在0个时钟周期（禁用）到数千个时钟周期的数量级之间进行配置。这允许根据系统其他部分在合并后的.SYS或.GL MEMBAR离开L1缓存320后的执行成本，对来自典型CTA的目标MEMBAR指令数量进行调节。当合并窗口关闭时，转换窗口会打开，具体细节结合图5A和图5B进一步描述。转换窗口的持续时间不可配置，因为它由L1缓存320与warp调度器及指令单元312之间的通信延迟决定。




When the coalesced MEMBAR instruction can be issued by the memory barrier instruction execution unit 500, the MEMBAR tracking unit 515 outputs the MEMBAR command to the MMU 328. Note that MEMBAR.CTA instructions (coalesced or not) are not output to the MMU 328 since execution of a MEMBAR.CTA instruction is completed for the threads within a SPM 310 when the memory transactions before the MEMBAR.CTA are committed to the L1 cache 320.

> 
当合并后的 MEMBAR 指令可以被存储器屏障指令执行单元 500 发出时，MEMBAR 跟踪单元 515 将 MEMBAR 命令输出到 MMU 328。注意，MEMBAR.CTA 指令（无论是否合并）不会输出到 MMU 328，因为当 MEMBAR.CTA 之前的存储器事务被提交到 L1 缓存 320 时，SPM 310 内的线程即完成了 MEMBAR.CTA 指令的执行。




The MEMBAR tracking unit 515 receives a MEMBAR ACK (acknowledgement) signal from the MMU 328 when each coalesced MEMBAR.GL and MEMBAR.SYS is completed. The MEMBAR.GL instructions are completed when all of the memory transactions before the MEMBAR .GL are committed to the L2 cache 350. The MEMBAR.SYS instructions are completed when all of the memory transactions before the MEMBAR.SYS are committed to system memory 104 which is typically considered to be when the memory transactions are output by the parallel processing subsystem 112 to the communication path 113.

> 
当每个合并的 MEMBAR.GL 和 MEMBAR.SYS 完成时，MEMBAR 跟踪单元 515 会从 MMU 328 接收 MEMBAR ACK（确认）信号。MEMBAR.GL 指令在 MEMBAR.GL 之前的所有内存事务均已提交至L2缓存350后完成。MEMBAR.SYS 指令则在 MEMBAR.SYS 之前的所有内存事务均已提交至系统内存104时完成，通常认为此时内存事务已由并行处理子系统112输出到通信路径113。




The global pending ACK count is used by the MEMBAR tracking unit 515 to determine when the coalesced MEMBAR command can be output and is described in further detail in conjunction with FIGS. 5A and 5B. When the MEMBAR ACK is received by the MEMBAR tracking unit 515, the MEMBAR accept/retry unit 505 outputs the MEMBAR done to the warp scheduler and instruction unit 312 and execution of the MEMBAR instruction is complete. Execution of any threads that were blocked waiting for a MEMBAR instruction to be done that is included in the coalesced MEMBAR instruction resumes.

> 
全局未决 ACK 计数被 MEMBAR 跟踪单元 515 用来确定何时可以输出合并后的 MEMBAR 命令，具体细节将结合图 5A 和图 5B 进一步描述。当 MEMBAR 跟踪单元 515 收到 MEMBAR ACK 时，MEMBAR 接受/重试单元 505 会将 MEMBAR 完成信号输出给线程组调度器和指令单元 312，此时 MEMBAR 指令的执行便告完成。所有因等待合并的 MEMBAR 指令中包含的 MEMBAR 指令完成而被阻塞的线程，其执行也随之恢复。




The MEMBAR accept/retry unit 505 also receives an external MEMBAR request signal that is output by the work distribution unit 200. An external MEMBAR request is used to enforce ordering between separately issued but dependent grids that are composed of multiple CTAs and distributed across the processing cluster array 230 for processing. As such, multiple grids can be in flight at any one time, but only if there are no dependencies among them. Before a grid Gn that depends on results from a grid Gk can begin, (1) all CTAs launched from Gk needs to be complete, and (2) all memory operations from Gk must be committed. However, multiple grids which do not depend on Gk but otherwise meet their respective dependency criteria, if any, may be launched while Gk is still running.

> 
MEMBAR 接受/重试单元 505 还接收由工作分配单元 200 输出的外部 MEMBAR 请求信号。外部 MEMBAR 请求用于在分开发出但存在依赖关系的网格之间强制顺序，这些网格由多个 CTA 组成，并分布到处理集群阵列 230 进行处理。因此，任意时刻可以有多个网格在并发执行，但前提是它们之间没有依赖关系。在依赖于网格 Gk 结果的网格 Gn 能够开始之前，(1) 从 Gk 启动的所有 CTA 必须完成，且 (2) 来自 Gk 的所有内存操作必须被提交。然而，不依赖于 Gk 但满足其各自依赖条件（如果有的话）的多个网格，可以在 Gk 仍在运行时启动。




To implement condition (2) above once condition (1) is established, the L1 cache 320 receives a request in the form of a memory flush bundle for a MEMBAR.GL or

> 
为了在条件(1)建立后实现上述条件(2)，L1高速缓存320以内存刷新包的形式接收针对MEMBAR.GL的请求，或者




## 18

MEMBAR.SYS operation. The memory flush bundle is provided external to the instruction stream and is shown as the external MEMBAR signal input to the MEMBAR accept/ retry unit 505. The external MEMBAR ensures all memory operations of a grid are committed, even though each CTA may not have issued a MEMBAR instruction via the warp scheduler and instruction unit 312 within the SPM 310. The work distribution unit 200 can issue requests (bundles) to the L1 cache 320 asynchronous to current requests received by 0 the SPM 310, so the external MEMBAR may be coalesced with any MEMBAR instructions issued by currently active and non-dependent grids. The external MEMBAR instruction can also be used for graphics work to similarly ensure results from running shaders on the SPM 310 (e.g., vertex shaders, geometry shaders, pixel shaders, . . . ) will be visible, either within the PPU 202 or if accessed by the CPU 102.

> 
MEMBAR.SYS 操作。内存刷新束在指令流外部提供，并作为外部 MEMBAR 信号输入到 MEMBAR 接受/重试单元 505。外部 MEMBAR 确保一个网格的所有内存操作均被提交，即使每个 CTA 可能未通过 SPM 310 内的 warp 调度器和指令单元 312 发出 MEMBAR 指令。工作分发单元 200 可以异步于 SPM 310 收到的当前请求向 L1 缓存 320 发出请求（束），因此外部 MEMBAR 可以与当前活跃且非依赖的网格发出的任何 MEMBAR 指令合并。外部 MEMBAR 指令也可用于图形工作，类似地确保在 SPM 310 上运行的着色器（例如，顶点着色器、几何着色器、像素着色器等）的结果可见，无论是在 PPU 202 内部还是由 CPU 102 访问。




In one embodiment, only one coalesced and pending MEMBAR instruction is supported. In another embodiment, multiple coalesced MEMBAR instructions may be in flight, 20 and each coalesced MEMBAR that is issued is assigned a unique identifier. When a thread reaches a MEMBAR instruction, the thread is blocked until execution of the MEMBAR is done. Importantly, threads that have not reached a MEMBAR instruction continue executing while other threads may be 25 blocked.

> 
在一个实施例中，仅支持一个合并且待处理的内存屏障指令。在另一个实施例中，多个合并的内存屏障指令可以同时处于执行中，20 并且每个发出的合并内存屏障都被分配一个唯一标识符。当线程到达内存屏障指令时，该线程被阻塞直到内存屏障执行完成。重要的是，未到达内存屏障指令的线程在其他线程可能被阻塞 25 时继续执行。




Additionally, the MMU 328 may be configured to perform an optimization for the MEMBAR.SYS and MEMBAR.GL instructions. Specifically, following a reset or execution of a MEMBAR.SYS or MEMBAR.GL instruction (coalesced or not), the MMU 328 may track if a memory transaction is received that accesses either the system memory or global memory. Since the MMU 328 performs address translation for each of the memory transactions that are received by the MMU 328, the MMU 328 is able to determine which portion

> 
此外，MMU 328可配置为对MEMBAR.SYS和MEMBAR.GL指令执行优化。具体而言，在复位或执行MEMBAR.SYS或MEMBAR.GL指令（无论是否合并）之后，MMU 328可追踪是否接收到访问系统内存或全局内存的内存事务。由于MMU 328为其接收的每个内存事务执行地址转换，因此能够确定哪个部分




35 of memory (system or global) a transaction accesses. When a MEMBAR.GL is received by the MMU 328 and a memory transaction accessing the global memory has not been received since a reset or execution of the previous MEMBAR instruction (MEMBAR.GL or MEMBAR.SYS), the MMU

> 
35 事务访问的内存（系统或全局）。当 MMU 328 接收到 MEMBAR.GL，且自复位或执行前一条 MEMBAR 指令（MEMBAR.GL 或 MEMBAR.SYS）以来未收到访问全局内存的内存事务时，MMU




328 is configured to discard the MEMBAR instruction and assert the MEMBAR ACK signal to the MEMBAR tracking unit 515 in the L1 cache 320. Similarly, when a MEMBAR.SYS is received by the MMU 328 and a memory transaction accessing either the system memory or the global

> 
328被配置为丢弃MEMBAR指令并向L1缓存320中的MEMBAR跟踪单元515声明MEMBAR ACK信号。类似地，当MMU 328接收到MEMBAR.SYS并且有一个内存事务访问系统内存或全局




5 memory has not been received since a reset or execution of the previous MEMBAR.SYS instruction, the MMU 328 is configured to discard the MEMBAR.SYS instruction and assert the MEMBAR ACK signal to the MEMBAR tracking unit 515 in the L1 cache 320. Finally, when a MEMBAR.SYS is

> 
5 自复位或执行前一条 MEMBAR.SYS 指令以来未收到存储器请求，MMU 328 被配置为丢弃该 MEMBAR.SYS 指令，并向 L1 高速缓存 320 中的 MEMBAR 跟踪单元 515 发出 MEMBAR ACK 信号。最后，当 MEMBAR.SYS 是




50 received by the MMU 328 and only a memory transaction accessing the global memory has been received since a reset or execution of the previous MEMBAR.SYS instruction, the MMU 328 is configured to demote the MEMBAR.SYS instruction to a MEMBAR.GL instruction.

> 
当 MMU 328 接收到 50，且自复位或执行前一条 MEMBAR.SYS 指令以来仅收到访问全局内存的内存事务时，MMU 328 被配置为将该 MEMBAR.SYS 指令降级为 MEMBAR.GL 指令。




FIG. 5A is a block diagram of finite state machine 520, according to one embodiment of the present invention. A deferring mechanism is used by the warp scheduler and instruction unit 312 and the L1 cache 320 to coalesce MEM-BAR.SYS and MEMBAR.GL requests. The scheme has three windows: coalescing, transitioning, and blocking. After a "triggering" MEMBAR instruction (a MEMBAR instruction that opens the coalescing window) is received by the memory barrier instruction execution unit 500 within the L1 cache 320, the MEMBAR accept/retry unit 505 records the MEM-BAR instruction and enters a coalescing window.

> 
图5A是根据本发明一个实施例的有限状态机520的框图。warp调度器和指令单元312以及L1高速缓存320使用延迟机制来合并MEMBAR.SYS和MEMBAR.GL请求。该方案有三个窗口：合并窗口、转换窗口和阻塞窗口。在L1高速缓存320内的内存屏障指令执行单元500接收到“触发”MEMBAR指令（打开合并窗口的MEMBAR指令）后，MEMBAR接受/重试单元505记录该MEMBAR指令并进入合并窗口。




While coalescing, the memory barrier instruction execution unit 500 in the L1 cache 320 will continue to process

> 
在合并期间，L1缓存320中的内存屏障指令执行单元500将继续处理。




19 requests from threads that have not reached a MEMBAR instruction including subsequent MEMBAR instructions. The duration of the coalescing window may be defined by a configurable register setting. At the end of the coalescing window, the MEMBAR accept/retry unit 505 enters a transition period, and requests the SPM 310 to not send any more global memory transaction requests to the L1 cache 320 via the hints signal. The L1 cache 320 still processes memory transaction requests while waiting for the hint to take effect. The transition window is based on nominal pipeline depth. After the transition period, the MEMBAR accept/retry unit 505 will enter a blocking window until the global pending ACK count is zero. Memory accesses to global and system memory are queued in the MMU 328 for output to the L2 Caches 350. In one embodiment, the MEMBAR command bypasses the queue, so the transition period is needed to allow the queued memory accesses to drain. In particular, the queued memory store transactions need to be drained before the MEMBAR command is output to the MMU 328. The global pending ACK count is updated each time a memory access is output from the queue. In an embodiment that does not allow the MEMBAR command to bypass the queue, the global pending ACK count is not needed and can be considered to have a value of zero.

> 
来自尚未到达 MEMBAR 指令（包括后续 MEMBAR 指令）的线程的 19 个请求。合并窗口的持续时间可由可配置的寄存器设置定义。在合并窗口结束时，MEMBAR 接受/重试单元 505 进入过渡期，并通过提示信号请求 SPM 310 不再向 L1 缓存 320 发送任何全局内存事务请求。L1 缓存 320 在等待提示生效期间仍会处理内存事务请求。过渡窗口基于标称流水线深度。过渡期之后，MEMBAR 接受/重试单元 505 将进入阻塞窗口，直到全局待处理 ACK 计数为零。对全局和系统内存的访问在 MMU 328 中排队，以输出到 L2 缓存 350。在一个实施例中，MEMBAR 命令绕过队列，因此需要过渡期来让排队的访存请求排空。具体来说，排队的存储事务需要在 MEMBAR 命令输出到 MMU 328 之前排空。每次从队列输出一个访存请求时，全局待处理 ACK 计数都会更新。在不允许 MEMBAR 命令绕过队列的实施例中，不需要全局待处理 ACK 计数，可将其视为零。




When the global pending ACK count equals zero, the MEMBAR tracking unit 515 sends the MEMBAR command to the MMU 328 and resumes processing of all requests except MEMBAR instructions for threads that have not reached a MEMBAR instruction. The blocking window ends when the MEMBAR ACK is returned. In one embodiment, the memory barrier instruction execution unit one MEMBAR outstanding to the MMU 328 at a time.

> 
当全局待处理 ACK 计数等于零时，MEMBAR 跟踪单元 515 向 MMU 328 发送 MEMBAR 命令，并恢复处理除未到达 MEMBAR 指令的线程的 MEMBAR 指令之外的所有请求。当 MEMBAR ACK 返回时，阻塞窗口结束。在一个实施例中，存储器屏障指令执行单元一次仅有一个发往 MMU 328 的未完成 MEMBAR。




The MEMBAR detection circuit 512 is configured to wait for "ACCEPT" from the L1 cache 320 for all previous LD/ST instructions in the same warp before outputting a MEMBAR instruction to the memory barrier instruction execution unit 500. Note that it is easiest to simply check for acceptance of all types of LD/ST, regardless of whether the LD/ST are for shared, local, or global memory. Therefore, the MEMB-AR.CTA is effectively executed by the MEMBAR detection circuit 512. After sending a MEMBAR, the MEMBAR detection circuit 512 will not send a further request for a warp until the MEMBAR is acknowledged as done, i.e., until MEMBAR done is output by the memory barrier instruction execution unit 500.

> 
MEMBAR 检测电路 512 被配置为，在向内存屏障指令执行单元 500 发出 MEMBAR 指令之前，等待同一 warp 中所有先前的 LD/ST 指令都收到来自 L1 缓存 320 的“接受（ACCEPT）”确认。请注意，最简单的方法是直接检查所有类型的 LD/ST 是否已被接受，而无需区分这些 LD/ST 访问的是共享内存、局部内存还是全局内存。因此，MEMBAR.CTA 实际上由 MEMBAR 检测电路 512 来执行。在发出 MEMBAR 之后，MEMBAR 检测电路 512 将不会为某个 warp 发出后续请求，直到该 MEMBAR 被确认为完成，即直到内存屏障指令执行单元 500 输出 MEMBAR done 信号为止。




When the MEMBAR accept/retry unit 505 receives and accepts a first MEMBAR instruction, the MEMBAR accept/ retry unit 505 transitions from the IDLE state 521 to the coalesce state 522 and opens the coalesce window. A counter in the coalesce window unit 503 is initialized and a minimum coalesce window delay period begins. While in the coalesce state 522, the MEMBAR accept/retry unit 505 accepts subsequent MEMBAR instructions received from the MEMBAR detection circuit 512 and coalesces the subsequent MEM-BAR instructions with the first MEMBAR instruction. The type of the coalesced MEMBAR instruction is promoted to MEMBAR.SYS if a MEMBAR.SYS type of MEMBAR instruction is received and the coalesced MEMBAR instruction is a MEMBAR.GL instruction. Alternatively, in another embodiment, when a higher level MEMBAR instruction is received, the existing coalesced MEMBAR instruction is output and the received higher level MEMBAR instruction is deferred, so that only MEMBAR instructions at the same level are combined together into a coalesced MEMBAR instruction.

> 
当 MEMBAR 接受/重试单元 505 接收并接受第一条 MEMBAR 指令时，MEMBAR 接受/重试单元 505 从 IDLE 状态 521 转换到 coalesce 状态 522，并打开合并窗口。合并窗口单元 503 中的计数器被初始化，最小合并窗口延迟期开始。处于 coalesce 状态 522 时，MEMBAR 接受/重试单元 505 接受从 MEMBAR 检测电路 512 接收到的后续 MEMBAR 指令，并将这些后续 MEMBAR 指令与第一条 MEMBAR 指令合并。如果收到 MEMBAR.SYS 类型的 MEMBAR 指令且已合并的 MEMBAR 指令为 MEMBAR.GL 指令，则合并后的 MEMBAR 指令类型提升为 MEMBAR.SYS。或者，在另一个实施例中，当接收到更高级别的 MEMBAR 指令时，现有的已合并 MEMBAR 指令被输出，而接收到的更高级别 MEMBAR 指令被推迟处理，从而仅将同一级别的 MEMBAR 指令合并为一个合并后的 MEMBAR 指令。




When the coalesce window unit 503 determines that the minimum coalesce delay period has expired, then the coa-

> 
当合并窗口单元503确定最小合并延迟周期已到期时，则合——




## 20

lesce window is closed and the MEMBAR accept/retry unit 505 transitions from the coalesce state 522 to the transition state 523. In the transition state 523 the MEMBAR accept/ retry unit 505 updates the hints/grants signal to discourage requests for accessing global and system memory (requests accessing local and shared memory are not discouraged). In one embodiment, only requests for stores to global and system memory are discouraged. The transition window duration allows time for the hints to take effect. When the transition window duration has expired, the MEMBAR accept/ retry unit 505 transitions from the transition state 523 to either the block state 524 or the block_issue state 525.

> 
合并窗口关闭，MEMBAR接受/重试单元505从合并状态522转换到过渡状态523。在过渡状态523下，MEMBAR接受/重试单元505更新提示/授权信号，以抑制访问全局和系统内存的请求（访问本地和共享内存的请求不受抑制）。在一个实施例中，仅抑制对全局和系统内存的存储请求。过渡窗口的持续时间允许提示信号生效。当过渡窗口持续时间到期后，MEMBAR接受/重试单元505从过渡状态523转换到阻塞状态524或阻塞_发出状态525。




When the global pending ACK count is non-zero, the MEMBAR accept/retry unit 505 enters the block state 524. Otherwise, the MEMBAR accept/retry unit 505 enters the block_issue state 525. When the MEMBAR accept/retry unit 505 is in the block state 524, the MEMBAR accept/retry unit 505 is in a blocking window and requests (possibly only store requests) that miss in the L1 cache 320 are deferred as "Backoff, write (or read) limit exceeded" as appropriate. When the MEMBAR accept/retry unit 505 is in the block state 524, the MEMBAR accept/retry unit 505 waits for a count of the global pending ACKs to go to zero and continues processing local and shared requests. Requests that hit in the L1 cache 320 are also accepted and processed while the MEMBAR accept/retry unit 505 is in the block state 524. When the count of the global pending ACKs equals zero, the MEMBAR accept/retry unit 505 transitions from the block state 524 to the block_issue state 525.

> 
当全局待处理 ACK 计数非零时，MEMBAR 接受/重试单元 505 进入阻塞状态 524。否则，MEMBAR 接受/重试单元 505 进入阻塞_发出状态 525。当 MEMBAR 接受/重试单元 505 处于阻塞状态 524 时，该单元处于阻塞窗口，在 L1 缓存 320 中未命中的请求（可能仅为存储请求）将视情况被推迟为"回退，写（或读）限制超出"。当 MEMBAR 接受/重试单元 505 处于阻塞状态 524 时，该单元等待全局待处理 ACK 的计数变为零，并继续处理本地和共享请求。在 MEMBAR 接受/重试单元 505 处于阻塞状态 524 期间，在 L1 缓存 320 中命中的请求也会被接受和处理。当全局待处理 ACK 的计数等于零时，MEMBAR 接受/重试单元 505 从阻塞状态 524 转换到阻塞_发出状态 525。




In the block_issue state 525, the MEMBAR accept/retry

> 
在 block_issue 状态 525 下，执行 MEMBAR 接受/重试




unit 505 outputs the MEMBAR command via the MEMBAR

> 
单元 505 通过 MEMBAR 输出内存屏障命令




tracking unit 515 and transitions to the block_wait state 526.

> 
跟踪单元515，并转换到block_wait状态526。




In the block-wait state 526, the MEMBAR accept/retry unit

> 
在阻塞等待状态 526 下，MEMBAR 接受/重试单元




505 indicates to the SPM 310 that read/write limits are no longer exceeded to allow requests that were deferred during the blocking window to be rescheduled. While in the block_wait state 526, the MEMBAR accept/retry unit 505 defers any subsequent MEMBAR instructions as "backoff,

> 
505向SPM 310指示读写限制已不再超出，从而允许在阻塞窗口期间被推迟的请求得到重新调度。在block_wait状态526下，MEMBAR接受/重试单元505将任何后续的MEMBAR指令推迟为“退避”状态，




MEMBAR pending" and continues to process other requests. When the MEMBAR tracking unit 515 receives the MEM-BAR ACK signal, the MEMBAR accept/retry unit 505 transitions from the block_wait state 526 to the ack state 527 and outputs the MEMBAR done signal before transitioning to the

> 
“MEMBAR pending”并继续处理其他请求。当 MEMBAR 跟踪单元 515 接收到 MEM-BAR ACK 信号时，MEMBAR 接受/重试单元 505 从 block_wait 状态 526 转换到 ack 状态 527，并输出 MEMBAR done 信号，然后转换到




45 idle state 521. At state ack 527 any MEMBAR instructions that were included in the coalesced MEMBAR have been executed. Any MEMBAR instructions deferred while the MEMBAR accept/retry unit 505 was in the block_wait state 526 may be reissued.

> 
45 空闲状态 521。在确认状态 527 下，包含在合并后的 MEMBAR 中的所有 MEMBAR 指令都已执行完毕。在 MEMBAR 接受/重试单元 505 处于阻塞等待状态 526 期间被延迟的任何 MEMBAR 指令可以重新发出。




FIG. 5B is a conceptual diagram illustrating the coalescing

> 
图5B是示出合并的概念图




of memory barrier instructions, according to one embodiment

> 
根据一个实施例，内存屏障指令的




of the present invention. A GST is a global store request that

> 
本发明的一个全局存储事务（GST）是一种全局存储请求，其




accesses global or system memory and is received by the L1

> 
访问全局或系统内存并由L1接收




cache 320 from a first warp. The L1 cache 320 receives a

> 
来自第一个线程束的缓存320。L1缓存320接收一个




sequence of three GST requests from the first warp. At a time 530, the memory barrier instruction execution unit 500 receives a triggering MEMBAR instruction from the warp and the coalesce window is opened. The memory barrier instruction execution unit 500 receives two additional MEM-

> 
第一个 warp 发出了三个 GST 请求序列。在时刻 530，内存屏障指令执行单元 500 从该 warp 接收到一条触发 MEMBAR 指令，合并窗口随即打开。内存屏障指令执行单元 500 又收到了两个额外的 MEM-




BAR instructions from second and third warps and coalesces the two additional MEMBAR instructions with the triggering MEMBAR instruction to produce a coalesced MEMBAR instruction. At time 535 the coalesce window closes and the transition window opens. During the transition window a

> 
来自第二和第三线程束的 BAR 指令，并将这两个额外的 MEMBAR 指令与触发性的 MEMBAR 指令合并，生成一个合并后的 MEMBAR 指令。在时间 535，合并窗口关闭，过渡窗口打开。在过渡窗口期间，一个




65 sequence of three global store requests from other warps are processed. At time 540 the transition window closes and the blocking window opens.

> 
65 来自其他线程束的三个全局存储请求序列被处理。在时刻540，转换窗口关闭，阻塞窗口打开。




## 21

During a first portion of the blocking window the global pending ACK count is reduced to zero and at time 545 the coalesced MEMBAR instruction is issued by the memory barrier instruction execution unit 500. While the blocking window is open subsequent MEMBAR instructions are deferred. While the blocking window is open and the global pending ACK count is not zero, global store requests that miss in the L1 cache 320 are deferred. Local and shared requests are processed during the blocking window. At time 550 the MEMBAR ACK is received by the memory barrier instruction execution unit 500 and at time 555 the MEMBAR done signal is output by the memory barrier instruction execution unit 500, closing the blocking window. At time 560 a second triggering MEMBAR instruction is received by the memory barrier instruction execution unit 500.

> 
在阻塞窗口的第一部分期间，全局待处理确认计数降至零，且在时刻545，合并后的MEMBAR指令由内存屏障指令执行单元500发出。当阻塞窗口开启时，后续的MEMBAR指令将被延迟。当阻塞窗口开启且全局待处理确认计数不为零时，在L1缓存320中未命中的全局存储请求将被延迟。局部和共享请求在阻塞窗口期间得到处理。在时刻550，内存屏障指令执行单元500接收到MEMBAR确认，并在时刻555输出MEMBAR完成信号，关闭阻塞窗口。在时刻560，内存屏障指令执行单元500接收到第二个触发MEMBAR指令。




When a halt condition occurs the coalescing window is closed and the state of the memory barrier instruction execution unit 500 is preserved. This ensures that the coalesced MEMBAR is output before new global stores after the halted condition is removed. Before the L1 cache 320 returns to the run state, the L1 cache 320 replays any deferred global stores, waits for the global ACKs (global pending ACK count to reach zero), and then sends the coalesced MEMBAR to the MMU 328.

> 
当发生停顿条件时，合并窗口关闭，内存屏障指令执行单元500的状态得到保留。这确保了在停顿条件解除后，合并后的MEMBAR先于新的全局存储输出。在L1缓存320返回运行状态之前，L1缓存320会重放任何被推迟的全局存储，等待全局ACK（全局待处理ACK计数降为零），然后将合并后的MEMBAR发送给MMU 328。




FIG. 6 is a flow diagram 600 of method steps for coalescing memory barrier instructions, according to one embodiment of the present invention. At step 605 the memory barrier instruction execution unit 500 receives a memory barrier instruction (MEMBAR). The MEMBAR instruction may be received from the MEMBAR detection circuit 512 or as an external MEMBAR instruction. At step 610 the MEMBAR accept/ retry unit 505 determines if the MEMBAR instruction can be accepted. Whether or not the MEMBAR instruction can be accepted depends on the state in which the MEMBAR accept/ retry unit 505 is operating in. In particular, MEMBAR instructions can be accepted when the MEMBAR accept/ retry unit 505 is in the idle state 521 or the coalesce state 522. If, at step 610 the MEMBAR instruction is not accepted, then at step 615 the MEMBAR instruction is discarded by the MEMBAR accept/retry unit 505 and the MEMBAR accept output signal is negated.

> 
根据本发明的一个实施例，图6是用于合并内存屏障指令的方法步骤的流程图600。在步骤605，内存屏障指令执行单元500接收一条内存屏障指令（MEMBAR）。该MEMBAR指令可从MEMBAR检测电路512接收，也可作为外部MEMBAR指令接收。在步骤610，MEMBAR接受/重试单元505判断该MEMBAR指令能否被接受。MEMBAR指令能否被接受取决于MEMBAR接受/重试单元505所处的运行状态。具体而言，当MEMBAR接受/重试单元505处于空闲状态521或合并状态522时，MEMBAR指令可被接受。如果在步骤610该MEMBAR指令未被接受，则在步骤615，该MEMBAR指令被MEMBAR接受/重试单元505丢弃，且MEMBAR接受输出信号被否定。




If, at step 610 the MEMBAR instruction is accepted by the MEMBAR accept/retry unit 505, then at step 617 the MEM-BAR instruction determines if the coalesce MEMBAR instruction should be promoted because the MEMBAR instruction received in step 605 is a higher level MEMBAR instruction (MEMBAR.SYS is considered higher level compared with MEMBAR.GL and MEMBAR.GL is considered higher level compared with MEMBAR.CTA). If, at step 617 the MEMBAR instruction determines that the coalesce MEMBAR instruction should be promoted, then at step 618 the coalesced MEMBAR is updated to the higher level.

> 
如果在步骤610中，MEMBAR指令被MEMBAR接受/重试单元505接受，那么在步骤617中，MEMBAR指令会判断是否需要提升合并后的MEMBAR指令，因为步骤605中接收到的MEMBAR指令是更高级别的MEMBAR指令（MEMBAR.SYS被认为比MEMBAR.GL级别更高，MEMBAR.GL被认为比MEMBAR.CTA级别更高）。如果在步骤617中，MEMBAR指令判断出应当提升合并后的MEMBAR指令，则在步骤618中，将合并后的MEMBAR更新为更高级别。




At step 620 the MEMBAR accept/retry unit 505 determines if the coalesced MEMBAR instruction can be issued, and, if not, the MEMBAR accept/retry unit 505 repeats step 620. The coalesced MEMBAR instruction may be issued when the MEMAR accept/retry unit 505 is in the blocking window and the global pending ACK count equals zero. When the coalesced MEMBAR instruction can be issued at step 620, then at step 625 the MEMBAR tracking unit 515 outputs the coalesced MEMBAR instruction to the MMU 328. At step 630 the MEMBAR tracking unit 515 waits for the MEMBAR ACK signal that is received when all of the memory transactions have been committed to memory at the level specified by the coalesced MEMBAR instruction (global or system level). When the MEMBAR ACK signal is received, then at step 635 the MEMBAR tracking unit 515

> 
在步骤620，MEMBAR接受/重试单元505判断合并后的MEMBAR指令是否能够发出，如果不能，则重复步骤620。当MEMBAR接受/重试单元505处于阻塞窗口且全局待处理确认计数等于零时，合并后的MEMBAR指令可以发出。当合并后的MEMBAR指令在步骤620可以发出时，则在步骤625，MEMBAR跟踪单元515将合并后的MEMBAR指令输出到MMU 328。在步骤630，MEMBAR跟踪单元515等待MEMBAR确认信号，该信号在所有内存事务已按照合并后的MEMBAR指令所指定的级别（全局或系统级别）提交到内存后接收。当收到MEMBAR确认信号时，则在步骤635，MEMBAR跟踪单元515……




## 22

informs the MEMBAR accept/retry unit 505 and the MEM-BAR is released by outputting the MEMBAR done signal.

> 
通知MEMBAR接收/重试单元505，并通过输出MEMBAR完成信号来释放MEMBAR。




Although the method steps are described in conjunction with the systems of FIGS. 1, 2, 3A, 3B, 3C, and 4, persons skilled in the art will understand that any system configured to perform the method steps, in any order, is within the scope of the inventions.

> 
尽管这些方法步骤是结合图1、图2、图3A、图3B、图3C和图4的系统来描述的，但本领域技术人员将理解，被配置为以任何顺序执行这些方法步骤的任何系统均落入本发明的范围内。




Using a hierarchical memory barrier instruction may improve performance since the different levels of memory barrier instruction have different execution latencies.

> 
由于不同级别的内存屏障指令具有不同的执行延迟，使用层次化内存屏障指令可能提升性能。




When threads execute independently, it may be advantageous to coalesce two or more memory barrier instructions for different threads that fall within a temporal window to execute a coalesced memory barrier instruction rather than 15 executing separate memory barrier instructions for each thread.

> 
当线程独立执行时，将在时间窗口内针对不同线程的两个或更多内存屏障指令合并，以执行合并后的内存屏障指令，而不是为每个线程执行单独的内存屏障指令，这可能是有利的。




## N-Way Memory Barrier Operation Coalescing

While coalescing multiple memory barrier instructions amortizes the latency of executing a single memory barrier instruction over multiple memory barrier instructions, only a single coalesced memory barrier instruction may be pending at a time. After the coalescing window is closed, any subsequent memory barrier instruction and instructions that follow that subsequent memory barrier instruction are not accepted by the memory barrier instruction execution unit 500.

> 
虽然合并多个内存屏障指令可以将执行单个内存屏障指令的延迟摊销到多个内存屏障指令上，但一次只能有一个待处理的合并内存屏障指令。合并窗口关闭后，后续的任何内存屏障指令以及跟随在该后续内存屏障指令之后的指令都不会被内存屏障指令执行单元 500 所接受。




N-way buffering of memory barrier instruction coalescing constructs a tagged memory transaction stream including sets of memory commands where each memory command is tagged with a coalescing index. A coalesced memory barrier instruction follows each set of memory transactions and has the same coalescing index as the preceeding set of memory transactions. A first phase of coalescing the memory barrier instructions may be overlapped with a second phase of transmitting the tagged memory command stream to the MMU 328. In other words, a first coalesced memory barrier instruction may be transmitted to the MMU 328 while subsequent memory commands are accepted and while subsequent memory barrier instructions are coalesced to generate a second coalesced memory barrier instruction.

> 
N路缓冲的内存屏障指令合并构建了一个带标记的内存事务流，其中包含多组内存命令，每个内存命令都被标记了一个合并索引。一个合并后的内存屏障指令跟随在每组内存事务之后，且具有与前面的那组内存事务相同的合并索引。合并内存屏障指令的第一阶段可以与将带标记的内存命令流传输到MMU 328的第二阶段重叠。换句话说，可以在接受后续内存命令并且合并后续内存屏障指令以生成第二个合并的内存屏障指令的同时，将第一个合并的内存屏障指令传输到MMU 328。




FIG. 7A is another block diagram of a memory barrier instruction execution unit 700, according to one embodiment of the present invention. In some embodiments the memory barrier instruction execution unit 500 is within each L1 cache 320 to coalesce memory barrier instructions, e.g., MEMB-AR.CTA, MEMBAR.GL, and MEMBAR.SYS, across multiple thread groups.

> 
图7A是根据本发明的一个实施例的存储器屏障指令执行单元700的另一框图。在一些实施例中，存储器屏障指令执行单元500位于每个L1高速缓存320内，以跨多个线程组合并存储器屏障指令，例如MEMB-AR.CTA、MEMBAR.GL和MEMBAR.SYS。




Memory barrier instruction execution unit 700 includes a 50 MEMBAR accept unit 705, a coalesce window unit 703, and a MEMBAR tracking unit 715. The warp scheduler and instruction unit 312 outputs a stream of memory commands including memory transactions (load and store instructions) and memory barrier instructions to a MEMBAR accept unit

> 
内存屏障指令执行单元700包括一个MEMBAR接收单元705、一个合并窗口单元703以及一个MEMBAR跟踪单元715。线程束调度器与指令单元312将包含内存事务（加载与存储指令）和内存屏障指令的内存命令流输出至MEMBAR接收单元




55 705 within the memory barrier instruction execution unit 700. Each memory transaction is executed in parallel by the threads in a particular thread group and the warp scheduler and instruction unit 312 may interleave instructions for different thread groups for output while ensuring that the pro- 60 gram instruction order is maintained for each individual thread group.

> 
55 705 位于内存屏障指令执行单元 700 内。每个内存事务由特定线程组中的线程并行执行，而 warp 调度器和指令单元 312 可交错输出不同线程组的指令，同时确保每个独立线程组的程序指令顺序得以保持。




Before dispatching a memory barrier instruction for a thread group to the memory barrier instruction execution unit 700, the warp scheduler and instruction unit 312 waits for all 5 memory transactions dispatched to the memory barrier instruction execution unit 700 for the thread group to be accepted. Unlike the memory barrier instruction execution

> 
在将线程组的存储器屏障指令分派到存储器屏障指令执行单元700之前，warp调度器和指令单元312会等待分派到存储器屏障指令执行单元700的该线程组的所有5个存储器事务被接受。与存储器屏障指令执行




23

> 
本专利提出了一种用于提高并行处理架构中内存屏障操作效率的技术。主要研究问题是如何降低开销高昂的内存屏障指令对性能的影响，这些指令用于在多个协同线程组间对内存事务进行排序。

其关键贡献在于一种N路内存屏障操作合并方法。当接收到某个线程组的第一条内存屏障指令时，该组后续内存操作的执行会被暂停。系统并不会立即将该内存屏障发送至内存系统，而是开启一个合并窗口。在此时间窗口内，来自不同线程组的后续内存屏障指令会与第一个屏障合并（coalesced），产生一条单一的合并内存屏障。系统会构建一个带标记的内存命令流，其中每个内存事务及合并屏障都与一个合并索引相关联。这使得多组事务及其合并屏障可以同时处于执行状态。当某个合并屏障正在处理时，仅阻塞受影响的线程组，而其他无关线程组的内存操作可以继续执行。

主要结论是，该技术显著降低了内存屏障的开销。通过将多个线程组的屏障合并，并使不同合并组的处理过程相互重叠，系统将单个屏障的长延迟分摊到了多个线程上，并避免了未参与同步的线程组产生不必要的停顿，从而提高了整体指令处理的吞吐量。




unit 500, the memory barrier instruction execution unit 700 does not defer memory barrier instructions. Once a memory barrier instruction is dispatched for a thread group, the warp scheduler and instruction unit 312 cannot dispatch any new memory transactions for the same thread group. In one embodiment, the warp scheduler and instruction unit 312 marks the thread group as "waiting" or "sleeping" and saves the coalescing index as well. When processing of the memory barrier instruction is complete, the warp scheduler and instruction unit 312 wakes up the thread group and resumes dispatching of memory commands for the thread group.

> 
单元 500 中，内存屏障指令执行单元 700 不会延迟内存屏障指令。一旦为某个线程组调度了一条内存屏障指令，线程束调度器与指令单元 312 不能为同一线程组再调度任何新的内存事务。在一种实施例中，线程束调度器与指令单元 312 将该线程组标记为“等待”或“休眠”，并保存合并索引。当内存屏障指令的处理完成时，线程束调度器与指令单元 312 唤醒该线程组，并恢复为该线程组调度内存命令。




The MEMBAR accept unit 705 accepts the memory commands dispatched by the warp scheduler and instruction unit 312 and when a triggering MEMBAR instruction is received, the MEMBAR accept unit 705 outputs a MEMBAR accept signal to the warp scheduler and instruction unit 312. The MEMBAR accept signal indicates that the triggering MEM-BAR instruction is accepted by the MEMBAR accept unit 705. The MEMBAR accept includes a coalescing index having a value between 0 and N, inclusive, that is provided by a coalescing index counter 706. All of the memory commands received after the triggering MEMBAR instruction are tagged with the coalescing index that was output with the MEMBAR accept signal in response to the triggering MEMBAR instruction. MEMBAR instructions received by the MEMBAR accept unit 705 after the triggering MEMBAR instruction may be coalesced during a coalescing window to generate a coalesced MEMBAR instruction that is associated with the coalescing index that was output.

> 
MEMBAR 接收单元 705 接受由线程束调度器与指令单元 312 派发的内存命令，当接收到一条触发 MEMBAR 指令时，MEMBAR 接收单元 705 向线程束调度器与指令单元 312 输出一个 MEMBAR 接受信号。该 MEMBAR 接受信号表示 MEMBAR 接收单元 705 已接受该触发 MEM-BAR 指令。该 MEMBAR 接受信号包含一个由合并索引计数器 706 提供的合并索引，其值在 0 到 N 之间（含边界）。在触发 MEMBAR 指令之后收到的所有内存命令，均用响应触发 MEMBAR 指令时随 MEMBAR 接受信号一同输出的合并索引进行标记。在触发 MEMBAR 指令之后由 MEMBAR 接收单元 705 收到的 MEMBAR 指令，可在合并窗口期内被合并，从而生成一条与所输出的合并索引相关联的合并 MEMBAR 指令。




The tagged memory commands and tagged coalesced MEMBAR instruction are combined to construct a tagged memory command stream that is stored in the tagged memory command stream buffer 718 within the MEMBAR tracking unit 715. Any memory commands that are received by the MEMBAR accept unit 705 before the first triggering MEM-BAR instruction are also stored in the tagged memory command stream buffer 718.

> 
带标记的存储器命令与带标记的合并 MEMBAR 指令组合在一起，构建成一个带标记的存储器命令流，该命令流存储在 MEMBAR 跟踪单元 715 内的带标记存储器命令流缓冲器 718 中。在第一个触发 MEMBAR 指令之前由 MEMBAR 接受单元 705 接收的任何存储器命令，也存储在带标记存储器命令流缓冲器 718 中。




A coalesce (temporal) window is opened by the coalesce window unit 703 when the triggering MEMBAR is received. The minimum duration of the coalescing window controlled by a coalesce timeout counter and is configurable from 0 clocks (disabled) to on the order 1000's of clocks. This allows tuning in the amount of memory barrier instructions that are coalesced. The coalesce window resulting in the generation of MEMBAR instruction $\mathrm{n}$ , where $\mathrm{n}$ is the coalescing index associated with the MEMBAR instruction, is closed when the coalesce timeout counter has expired and a MEMBAR ACK for the MEMBAR instruction corresponding to the coalescing index of $\left( {n - \left( {N - 1}\right) }\right) \% N$ is received from the MMU 328. For example, when $\mathrm{N} = 3$ the coalesce window unit 703 may wait until MEMBAR n-2 is ACKed to close the coalesce window associated with coalescing index n.

> 
当接收到触发 MEMBAR 时，合并窗口单元 703 会打开一个合并（时间）窗口。合并窗口的最短持续时间由合并超时计数器控制，并可在 0 个时钟周期（禁用）到约 1000 个时钟周期之间配置。这样可以调整被合并的内存屏障指令数量。产生 MEMBAR 指令 $\mathrm{n}$ 的合并窗口（其中 $\mathrm{n}$ 是与该 MEMBAR 指令关联的合并索引）在合并超时计数器到期，并且从 MMU 328 收到对应于合并索引 $\left( {n - \left( {N - 1}\right) }\right) \% N$ 的 MEMBAR 指令的 MEMBAR ACK 时关闭。例如，当 $\mathrm{N} = 3$ 时，合并窗口单元 703 可能会等待 MEMBAR n-2 被确认，以关闭与合并索引 n 关联的合并窗口。




The MEMBAR tracking unit 515 receives a MEMBAR ACK (acknowledgement) signal from the MMU 328 when processing of each coalesced MEMBAR is completed. When the MEMBAR ACK is received by the MEMBAR tracking unit 715, the MEMBAR accept unit 705 outputs the MEM-BAR done to the warp scheduler and instruction unit 312 and execution of the MEMBAR instruction is complete. The MEMBAR done signal includes the coalescing index n associated with the coalesced MEMBAR. Execution of any thread groups that were blocked waiting for a MEMBAR instruction to be done that is included in the coalesced MEMBAR index n resumes. In other words, the MEMBAR DONE to 312 provides feedback to the warp scheduler and instruction unit

> 
MEMBAR 跟踪单元 515 在完成每个合并的 MEMBAR 处理后，从 MMU 328 接收 MEMBAR ACK（确认）信号。当 MEMBAR 跟踪单元 715 收到 MEMBAR ACK 时，MEMBAR 接受单元 705 向 warp 调度器和指令单元 312 输出 MEMBAR done，MEMBAR 指令执行完成。MEMBAR done 信号包含与合并 MEMBAR 相关的合并索引 n。任何因等待包含在合并 MEMBAR 索引 n 中的 MEMBAR 指令完成而被阻塞的线程组恢复执行。换句话说，到 312 的 MEMBAR DONE 为 warp 调度器和指令单元提供反馈。




## 24

312 that the coalesced MEMBAR associated with index n has completed and warps waiting for the coalesced MEMBAR index n may be released.

> 
312 表明与索引 n 关联的合并 MEMBAR 已完成，等待该合并 MEMBAR 索引 n 的线程束可以被释放。




The MEMBAR tracking unit 715 includes N ACK 5 counters, ACK count0 713 through ACK countN-1 714 that track the acknowledge signals received in response to each memory transaction that is output from the tagged memory command stream buffer 718. The acknowledge signals for the different coalescing indices are tracked separately. Specifi-

> 
MEMBAR跟踪单元715包括N个ACK计数器，即ACK计数0 713至ACK计数N-1 714，这些计数器追踪响应带标签的内存命令流缓冲器718所输出的每个内存事务而接收的确认信号。不同合并索引的确认信号被分别追踪。具体-




10 cally, when a memory transaction tagged with a particular coalescing index is transmitted to the MMU 328 by the MEMBAR tracking unit 715, the ACK counter associated with the particular coalescing index is incremented. When an acknowledgement (ACK) for the particular coalescing index

> 
具体来说，当带有特定合并索引标记的内存事务被MEMBAR跟踪单元715传输至MMU 328时，与该特定合并索引关联的ACK计数器递增。当针对该特定合并索引的确认（ACK）




15 is received by the MEMBAR tracking unit 715 from the MMU 328, the ACK counter associated with the particular coalescing index is decremented. The memory barrier instruction tagged with the particular coalescing index can only by output by the MEMBAR tracking unit 715 when the

> 
当MMU 328向MEMBAR跟踪单元715发送15时，与该特定合并索引关联的ACK计数器递减。只有当




20 ACK counter associated with the particular coalescing index

> 
与特定合并索引关联的20个ACK计数器




has a value of zero, indicating that all of the memory trans-

> 
为零（零值），这表示所有内存事务




actions for the particular coalescing index have been

> 
针对特定合并索引的操作已经




acknowledged by the MMU 328.

> 
由 MMU 328 确认。




FIG. 7B is a block diagram of a coalescer finite state 25 machine 720 and a tracking finite state machine 730, according to one embodiment of the present invention. The coa-lescer finite state machine 720 and the tracking finite state machine 730 illustrate the functionality provided by the MEMBAR accept unit 705 and the MEMBAR tracking unit

> 
图7B是根据本发明一个实施例的合并器有限状态机720和跟踪有限状态机730的框图。合并器有限状态机720和跟踪有限状态机730展示了MEMBAR接受单元705和MEMBAR跟踪单元所提供的功能。




30 715, respectively. The coalescer finite state machine 720 controls the interaction between the warp scheduler and instruction unit 312 and the MEMBAR coalescing window. The tracking finite state machine 730 controls the transmission of the coalesced MEMBAR instructions and the corresponding

> 
分别为 30 715。合并器有限状态机 720 控制 warp 调度器与指令单元 312 以及 MEMBAR 合并窗口之间的交互。跟踪有限状态机 730 控制合并后的 MEMBAR 指令及相应的




35 memory transactions. Note that for the following description, $\mathrm{N} = 2$ and all memory transactions are assumed to the to the same memory space (global address space) and tracked by the MEMBAR operations.

> 
35 次内存事务。请注意，对于以下描述，$\mathrm{N} = 2$，并且所有内存事务都假定至至相同的内存空间（全局地址空间），并由 MEMBAR 操作跟踪。




An IDLE0 721 state of the coalescer finite state machine

> 
合并器有限状态机的 IDLE0 721 状态




40 720 is the reset state. While in the IDLE0 721 state, the

> 
40 720 是复位状态。当处于 IDLE0 721 状态时，




MEMBAR accept unit 705 accepts memory commands.

> 
MEMBAR 接收单元 705 接收内存命令。




Memory transactions that are received are tagged with the

> 
接收到的内存事务被标记为所述




coalescing index $\mathrm{n}$ ( $\mathrm{n} = 0$ after reset) and output to the tagged

> 
合并索引 $\mathrm{n}$（复位后 $\mathrm{n}=0$）并输出到带标签的




memory command stream buffer 718. When a triggering

> 
内存命令流缓冲区718。当触发




MEMBAR instruction is received, the MEMBAR accept unit 705 transitions from the IDLE0 721 state to a COALESCE0 722 state. A first coalescing window is opened and the MEM-BAR accept unit 705 provides the warp scheduler and instruction unit 312 with the coalescing index 0 . Memory transac-

> 
接收到 MEMBAR 指令时，MEMBAR 接受单元 705 从 IDLE0 721 状态转换到 COALESCE0 722 状态。第一个合并窗口打开，MEMBAR 接受单元 705 向 warp 调度器和指令单元 312 提供合并索引 0。内存事务




tions that are received after the triggering memory barrier instruction are also tagged with the coalescing index $\mathrm{n}(\mathrm{n} = 0$ after reset) and output to the tagged memory command stream buffer 718. The memory transactions that are tagged with the coalescing index 0 are a first set of memory transac-

> 
在触发内存屏障指令之后接收的事务也被标记为合并索引 $\mathrm{n}$（复位后 $\mathrm{n} = 0$），并输出到标记内存命令流缓冲区 718。标记有合并索引 0 的内存事务是第一组内存事务——




tions. Memory barrier instructions that are received while the MEMBAR accept unit 705 is in the COALESCE0 722 state are accepted and coalesced with the triggering memory barrier instruction to generate a coalesced memory barrier instruction associated with the coalescing index 0 .

> 
tions. 当MEMBAR接受单元705处于COALESCE0 722状态时，接收到的内存屏障指令会被接受并与触发内存屏障指令合并，生成一个与合并索引0关联的合并内存屏障指令。




When the coalescing window closes, the MEMBAR accept unit 705 transitions from the COALESCE0 722 state to the MEMBAR0 to M-stage 723 state. As previously explained, the coalescing window is closed when the coalesce timeout counter has expired and a MEMBAR ACK for the MEMBAR

> 
当合并窗口关闭时，MEMBAR 接受单元 705 从 COALESCE0 722 状态转换到 MEMBAR0 to M-stage 723 状态。如前所述，当合并超时计数器到期并接收到针对该 MEMBAR 的 MEMBAR ACK 时，合并窗口关闭。




5 instruction corresponding to the coalescing index of (n-(N- 1)) % N is received from the MMU 328. In the MEMBAR0 to M-stage 723 state the MEMBAR accept unit 705 tags the

> 
与合并索引(n-(N-1))%N对应的5指令从MMU 328接收。在MEMBAR0到M阶段723状态下，MEMBAR接受单元705标记




25 coalesced memory barrier instruction with the coalescing index 0 and inserts the coalesced memory barrier instruction into the tagged memory command stream buffer 718 immediately following the last memory transaction in the first set of memory transactions. The MEMBAR accept unit 705 then increments (modulo $\mathrm{N}$ so that the counter wraps) the coalescing index and transitions from the MEMBAR0 to M-stage 723 state to an IDLE1 724 state.

> 
25 合并后的内存屏障指令的合并索引为 0，并将该合并后的内存屏障指令插入标记内存命令流缓冲器 718 中，紧跟在第一组内存事务中的最后一个内存事务之后。然后，MEMBAR 接受单元 705 将合并索引递增（模 $\mathrm{N}$ 以使计数器回绕），并从 MEMBAR0 至 M 级 723 状态转换到 IDLE1 724 状态。




While in the IDLE1 724 state, the MEMBAR accept unit 705 accepts memory commands. Memory transactions that are received are tagged with the coalescing index 1 and output to the tagged memory command stream buffer 718. When a second triggering MEMBAR instruction is received, the MEMBAR accept unit 705 transitions from the IDLE1 724 state to a COALESCE1 725 state. A second coalescing window is opened and the MEMBAR accept unit 705 acknowledges that the second triggering MEMBAR instruction is accepted and provides the warp scheduler and instruction unit 312 with the coalescing index 1. Memory transactions that are received after the second triggering memory barrier instruction are also tagged with the coalescing index 1 and output to the tagged memory command stream buffer 718. The memory transactions that are tagged with the coalescing index 1 are a second set of memory transactions. Memory barrier instructions that are received while the MEMBAR accept unit 705 is in the COALESCE1 725 state are accepted and coalesced with the second triggering memory barrier instruction to generate a second coalesced memory barrier instruction associated with the coalescing index 1.

> 
当处于IDLE1 724状态时，MEMBAR接受单元705接受内存命令。收到的内存事务被标记上合并索引1，并输出到标记内存命令流缓冲器718。当收到第二个触发MEMBAR指令时，MEMBAR接受单元705从IDLE1 724状态转换到COALESCE1 725状态。第二个合并窗口被打开，MEMBAR接受单元705确认已接受第二个触发MEMBAR指令，并向warp调度器和指令单元312提供合并索引1。在第二个触发内存屏障指令之后收到的内存事务，也被标记上合并索引1并输出到标记内存命令流缓冲器718。标记有合并索引1的内存事务是第二组内存事务。当MEMBAR接受单元705处于COALESCE1 725状态时收到的内存屏障指令被接受，并与第二个触发内存屏障指令合并，生成一个关联合并索引1的第二合并内存屏障指令。




When the second coalescing window closes, the MEM-BAR accept unit 705 transitions from the COALESCE1 725 state to the MEMBAR1 to M-stage 726 state. In the MEM-BAR1 to M-stage 726 state the MEMBAR accept unit 705 tags the second coalesced memory barrier instruction with the coalescing index 1 and inserts the second coalesced memory barrier instruction into the tagged memory command stream buffer 718 immediately following the last memory transaction in the second set of memory transactions. The MEMBAR accept unit 705 then increments (modulo $\mathrm{N}$ so that the counter wraps) the coalescing index and transitions from the MEM-BAR1 to M-stage 726 state to the IDLE0 721 state. When N is greater than 2, additional sets of IDLE, COALESCE, and MEMBAR to M-stage states are included between the MEM-BAR1 to M-stage 726 and the IDLE0 721 states to implement the N-way memory barrier operation coalescing.

> 
当第二个合并窗口关闭时，MEM-BAR 接受单元 705 从 COALESCE1 725 状态转换到 MEMBAR1 to M-stage 726 状态。在 MEM-BAR1 to M-stage 726 状态下，MEMBAR 接受单元 705 使用合并索引 1 来标记第二个合并后的内存屏障指令，并将该第二个合并后的内存屏障指令插入到带标记的内存命令流缓冲区 718 中，紧跟在第二组内存事务中的最后一个内存事务之后。然后，MEMBAR 接受单元 705 以 $\mathrm{N}$ 为模递增合并索引（以使计数器回绕），并从 MEM-BAR1 to M-stage 726 状态转换到 IDLE0 721 状态。当 N 大于 2 时，在 MEM-BAR1 to M-stage 726 和 IDLE0 721 状态之间会包含额外的 IDLE、COALESCE 和 MEMBAR to M-stage 状态集，以实现 N 路内存屏障操作合并。




A STREAM MCMD0 731 state of the tracking finite state machine 730 is the reset state. While in the STREAM MCMD0 731, the MEMBAR tracking unit 715 streams tagged memory transactions from the tagged memory command stream buffer 718 to the MMU 328. The ACK Count0 713 is incremented for each memory transaction having a coalescing index of 0 that is transmitted. The ACK Count0 713 is decremented for each memory transaction ACK having a coalescing index of 0 that is received by the MEMBAR tracking unit 715. When a MEMBAR instruction is reached in the tagged memory command stream buffer 718, the MEM-BAR tracking unit 715 transitions from the STREAM MCMD0 731 state to a WAIT_PEND_ACK0 732 state. When the ACK count the transmitted memory transactions tagged with the coalescing index 0 have been acknowledged by the MMU 328, the MEMBAR tracking unit 715 transitions from the WAIT_PEND_ACK0 732 state to the TRANSMIT MEM-BAR0 733 state.

> 
跟踪有限状态机 730 的 STREAM MCMD0 731 状态是复位状态。当处于 STREAM MCMD0 731 状态时，MEMBAR 跟踪单元 715 将来自标记内存命令流缓冲区 718 的标记内存事务流式传输到 MMU 328。对于每个传输的、凝聚索引为 0 的内存事务，ACK Count0 713 递增。对于 MEMBAR 跟踪单元 715 接收到的每个凝聚索引为 0 的内存事务 ACK，ACK Count0 713 递减。当在标记内存命令流缓冲区 718 中遇到 MEMBAR 指令时，MEMBAR 跟踪单元 715 从 STREAM MCMD0 731 状态转换到 WAIT_PEND_ACK0 732 状态。当标记为凝聚索引 0 的已传输内存事务的 ACK 计数已由 MMU 328 确认时，MEMBAR 跟踪单元 715 从 WAIT_PEND_ACK0 732 状态转换到 TRANSMIT MEM-BAR0 733 状态。




While in the WAIT_PEND_ACK0 732 state, the MEM-BAR tracking unit 715 may be configured to transmit tagged memory transactions having a tag that does not equal 0 while

> 
当处于WAIT_PEND_ACK0 732状态时，MEM-BAR跟踪单元715可被配置为传输具有不等于0的标签的带标签的内存事务，而




## 26

also updating the corresponding ACK count for transmissions and ACKs. The order of the memory transaction ACKs returned from MMU 328 does not necessarily match the order in which the memory transactions were sent to the MMU 328.

> 
同时更新传输和 ACK 对应的 ACK 计数。从 MMU 328 返回的内存事务 ACK 的顺序不一定与发送到 MMU 328 的内存事务顺序一致。




In the TRANSMIT MEMBAR0 733 state the MEMBAR tracking unit 715 transmits the tagged coalesced memory barrier instruction (n=0). When the MMU 328 completes processing of the tagged coalesced memory barrier instruction, the MMU 328 asserts the MEMBAR ACK input to the MEMBAR tracking unit 715 and the coalesce window unit 703 closes a coalescing window corresponding to the coalescing index n+1 for $\mathrm{N} = 2$ or the coalescing index $\left( {\mathrm{n} + \left( {\mathrm{N} - 1}\right) }\right) \; \% \mathrm{\;N}$ for any $\mathrm{N}$ (assuming that the coalescing timeout counter has expired). The MEMBAR tracking unit 715 transitions from the TRANSMIT MEMBAR0 733 state to the STREAM MCMD1 734 state.

> 
在 TRANSMIT MEMBAR0 733 状态下，MEMBAR 跟踪单元 715 发送带标记的合并内存屏障指令（n=0）。当 MMU 328 完成处理该带标记的合并内存屏障指令时，MMU 328 向 MEMBAR 跟踪单元 715 断言 MEMBAR ACK 输入，并且合并窗口单元 703 关闭与合并索引 n+1（当 $\mathrm{N} = 2$ 时）或合并索引 $\left( {\mathrm{n} + \left( {\mathrm{N} - 1}\right) }\right) \; \% \mathrm{\;N}$（对于任意 $\mathrm{N}$，假设合并超时计数器已到期）对应的合并窗口。MEMBAR 跟踪单元 715 从 TRANSMIT MEMBAR0 733 状态转换到 STREAM MCMD1 734 状态。




While in the STREAM MCMD1 734, the MEMBAR tracking unit 715 streams tagged memory transactions in the second set of memory transactions from the tagged memory 20 command stream buffer 718 to the MMU 328. The ACK Count1 is incremented for each memory transaction having a coalescing index of 1 that is transmitted. The ACK Count1 is decremented for each memory transaction ACK having a coalescing index of 1 that is received by the MEMBAR track- 25 ing unit 715.

> 
当处于 STREAM MCMD1 734 中时，MEMBAR 跟踪单元 715 将来自带标记的内存命令流缓冲区 718 的第二组内存事务中的带标记内存事务流式传输到 MMU 328。每传输一个合并索引为 1 的内存事务，ACK Count1 递增。MEMBAR 跟踪单元 715 每收到一个合并索引为 1 的内存事务 ACK，ACK Count1 递减。




When a MEMBAR instruction is reached in the tagged memory command stream buffer 718, the MEMBAR tracking unit 715 transitions from the STREAM MCMD1 734 state to a WAIT_PEND_ACK1 736 state. When the ACK 30 count 1 equals zero, indicating that all of the transmitted memory transactions tagged with the coalescing index 1 have been acknowledged by the MMU 328, the MEMBAR tracking unit 715 transitions from the WAIT_PEND_ACK1 736 state to the TRANSMIT MEMBAR1 738 state. While in the WAIT_PEND_ACK1 736 state, the MEMBAR tracking unit 715 may be configured to transmit tagged memory transactions having a tag that does not equal 1 while also updating the corresponding ACK count for transmissions and ACKs.

> 
当在带标记的内存命令流缓冲区 718 中遇到 MEMBAR 指令时，MEMBAR 跟踪单元 715 从 STREAM MCMD1（流式 MCMD1）状态 734 转换为 WAIT_PEND_ACK1（等待待处理确认 1）状态 736。当 ACK 30 计数 1 等于零，表明所有标记为合并索引 1 的已传输内存事务均已由 MMU 328 确认，则 MEMBAR 跟踪单元 715 从 WAIT_PEND_ACK1 状态 736 转换为 TRANSMIT MEMBAR1（发送 MEMBAR1）状态 738。在 WAIT_PEND_ACK1 状态 736 期间，MEMBAR 跟踪单元 715 可被配置为传输标记不等于 1 的内存事务，同时为传输和确认更新相应的 ACK 计数。




In the TRANSMIT MEMBAR1 738 state the MEMBAR 40 tracking unit 715 transmits the tagged coalesced memory barrier instruction (n=1). When the MMU 328 completes processing of the tagged coalesced memory barrier instruction, the MMU 328 asserts the MEMBAR ACK input to the MEMBAR tracking unit 715 and the coalesce window unit 45 703 closes a coalescing window corresponding to the coalescing index $\mathrm{n} - 1\left( {\mathrm{n} + 1 = 0\text{ when }\mathrm{N} = 2}\right)$ , assuming that the coalescing timeout counter has expired.

> 
在TRANSMIT MEMBAR1 738状态下，MEMBAR 40跟踪单元715发送带标记的合并内存屏障指令（n=1）。当MMU 328完成处理该带标记的合并内存屏障指令后，MMU 328向MEMBAR跟踪单元715置位MEMBAR ACK输入，并且合并窗口单元45 703关闭对应于合并索引$\mathrm{n} - 1\left( {\mathrm{n} + 1 = 0\text{ when }\mathrm{N} = 2}\right)$的合并窗口，前提是合并超时计数器已到期。




FIG. 7C is another conceptual diagram illustrating a timing diagram 750 for the coalescing of memory barrier instruc- 50 tions, according to one embodiment of the present invention. Note that at any one time, there is only one MEMBAR outstanding within the MEMORY subsystem, so the complexity in overlapping MEMBAR operations is contained to the LSU 303. In one embodiment, the coalescing index is output with 55 the MEMBAR and the MMU 328 may process multiple MEMBARs simultaneously.

> 
图7C是示出根据本发明一个实施例的用于合并内存屏障指令的时序图750的另一概念图。注意，在任何时刻，存储器子系统内仅有一个待处理的MEMBAR，因此重叠MEMBAR操作的复杂性被限制在LSU 303内部。在一个实施例中，合并索引与MEMBAR一同输出，而MMU 328可以同时处理多个MEMBAR。




While in the IDLE0 721 state, the MEMBAR accept unit 705 accepts memory commands (shown as global load/store operations) that are tagged with the coalescing index 0 and ) output to the tagged memory command stream buffer 718. The M-stage finite state machine 730 is in the STREAM MCMD0 731 state and the tagged memory commands are read from the tagged memory command stream buffer 718 and transmitted to the MMU 328.

> 
当处于IDLE0 721状态时，MEMBAR接受单元705接受标记有合并索引0的内存命令（显示为全局加载/存储操作），并输出到带标记的内存命令流缓冲器718。M阶段有限状态机730处于STREAM MCMD0 731状态，带标记的内存命令从带标记的内存命令流缓冲器718中读出，并传输到MMU 328。




At time 530 a triggering MEMBAR is received and a coalesce window0is opened and the MEMBAR accept unit 705 provides the warp scheduler and instruction unit 312 with

> 
在时刻530，收到一个触发MEMBAR，并打开合并窗口0，MEMBAR接受单元705向warp调度器和指令单元312提供




## 27

the coalescing index 0 . When the triggering MEMBAR instruction is received, the MEMBAR accept unit 705 transitions from the IDLE0721 state to a COALESCE0 722 state. MEMBAR instructions received during the coalesce window 0 are combined with the triggering MEMBAR instruction and are not output to the tagged memory command stream buffer 718. Memory commands that are received during the coalesce window0are tagged with the coalescing index 0 and are output to the tagged memory command stream buffer 718. The tagged memory commands are read from the tagged memory command stream buffer 718 and transmitted to the MMU 328.

> 
合并索引 0 。当接收到触发的 MEMBAR 指令时，MEMBAR 接受单元 705 从 IDLE0721 状态转换到 COALESCE0 722 状态。在合并窗口 0 期间接收到的 MEMBAR 指令与触发的 MEMBAR 指令合并，并且不输出到带标记的内存命令流缓冲器 718。在合并窗口 0 期间接收到的内存命令被标记合并索引 0 ，并输出到带标记的内存命令流缓冲器 718。带标记的内存命令从带标记的内存命令流缓冲器 718 中读出，并传送给 MMU 328。




At time 535 the coalescing timeout expires and the T-stage finite state machine 720 transitions from the COALESCE0 722 state to the MEMBAR0 to M-stage 723 state and coalescing window0closes. When the coalesced MEMBAR instruction is output by the MEMBAR accept unit 705 at time 535, the coalescing index counter 706 increments the coalescing index and memory commands that are received after the coalesce window0is closed are tagged with the coalescing index 1 and are output to the tagged memory command stream buffer 718. At time 540 the T-stage finite state machine 720 transitions from the MEMBAR0 to M-stage 723 state to the IDLE1 724 and the M-stage finite state machine 730 transitions from the STREAM MCMD0 731 state to the WAIT_PEND_ACK0732 state. The MEMBAR tracking unit 715 waits for the ACK count0 713 to become zero before transitioning to the TRNSMIT MEMBAR0 733 STATE AT TIME 545 and transmitting the coalesced MEMBAR associated with coalescing index 0.

> 
在时间535，合并超时到期，T级有限状态机720从COALESCE0 722状态转换到MEMBAR0至M级723状态，合并窗口0关闭。当合并后的MEMBAR指令由MEMBAR接受单元705在时间535输出时，合并索引计数器706递增合并索引，合并窗口0关闭后收到的内存命令标记为合并索引1，并输出到带标记的内存命令流缓冲区718。在时间540，T级有限状态机720从MEMBAR0至M级723状态转换到IDLE1 724状态，M级有限状态机730从STREAM MCMD0 731状态转换到WAIT_PEND_ACK0732状态。MEMBAR跟踪单元715等待ACK计数0 713变为零，然后在时间545转换到TRNSMIT MEMBAR0 733状态，并发送与合并索引0关联的合并的MEMBAR。




At time 548 a second triggering MEMBAR is received and a coalesce window 1 is opened and the MEMBAR accept unit 705 provides the warp scheduler and instruction unit 312 with the coalescing index 1. The T-stage finite state machine 720 transitions from the IDLE1 724 state to the COALESCE1 725 state. MEMBAR instructions received during the coalesce window 1 are combined with the second triggering MEM-BAR instruction and are not output to the tagged memory command stream buffer 718. Memory commands that are received during the coalesce window1are tagged with the coalescing index 1 and are output to the tagged memory command stream buffer 718. The tagged memory commands are read from the tagged memory command stream buffer 718 and transmitted to the MMU 328.

> 
在时间 548，第二个触发 MEMBAR 被接收，合并窗口 1 被打开，MEMBAR 接受单元 705 向 warp 调度器和指令单元 312 提供合并索引 1。T 阶段有限状态机 720 从 IDLE1 724 状态转换到 COALESCE1 725 状态。在合并窗口 1 期间接收到的 MEMBAR 指令与第二个触发 MEMBAR 指令合并，并不输出到标记的内存命令流缓冲区 718。在合并窗口 1 期间接收到的内存命令被标记为合并索引 1，并输出到标记的内存命令流缓冲区 718。标记的内存命令从标记的内存命令流缓冲区 718 中读取，并传输到 MMU 328。




At time 555 the MMU 328 transmits an acknowledgement (ACK) signal for the first coalesced MEMBAR. When the ACK is transmitted by the MEMBAR tracking unit 715 to the memory barrier instruction execution unit 700 the T-stage finite state machine 720 transitions from the COALESCE1 725 state to the MEMBAR1 to M-stage 726 state and coalescing window 1 closes (the coalescing window timeout had already expired). At time 558 the T-stage finite state machine 720 transitions from the MEMBAR1 to M-stage 726 state to the IDLE0 721 and the M-stage finite state machine 730 transitions from the STREAM MCMD0 731 state to the WAIT_PEND_ACK0 732 state.

> 
在时间555，MMU 328发送针对第一个合并内存屏障的确认（ACK）信号。当确认信号由MEMBAR跟踪单元715传送至内存屏障指令执行单元700时，T阶段有限状态机720从COALESCE1 725状态转换到MEMBAR1至M阶段726状态，并且合并窗口1关闭（合并窗口的超时已经到期）。在时间558，T阶段有限状态机720从MEMBAR1至M阶段726状态转换到IDLE0 721状态，并且M阶段有限状态机730从STREAM MCMD0 731状态转换到WAIT_PEND_ACK0 732状态。




The MEMBAR tracking unit 715 waits for the ACK count1 to become zero before transitioning to the TRANSMIT MEMBAR1 738 STATE at time 562 and transmitting the coalesced MEMBAR associated with coalescing index 1 at time 562. At time 560 a second triggering MEMBAR is received and a second coalesce window0is opened and the MEMBAR accept unit 705 provides the warp scheduler and instruction unit 312 with the coalescing index 0 . The T-stage finite state machine 720 transitions from the IDLE0721 state to the COALESCE0 722 state.

> 
MEMBAR 跟踪单元 715 等待 ACK count1 变为零，然后在时间 562 转换到 TRANSMIT MEMBAR1 738 STATE，并在时间 562 发送与合并索引 1 关联的合并 MEMBAR。在时间 560，接收到第二个触发 MEMBAR，第二个合并窗口 0 被打开，MEMBAR 接受单元 705 向 warp 调度器和指令单元 312 提供合并索引 0。T 阶段有限状态机 720 从 IDLE0721 状态转换到 COALESCE0 722 状态。




## 28

FIG. 8A is a conceptual diagram of illustrating the tagged memory command stream buffer 718 of FIG. 7A, according to one embodiment of the present invention. The MEMBAR accept unit 705 constructs the tagged memory command stream buffer 718 including a first set of memory transactions tagged with a coalescing index of 0, e.g., load 801, store 803, and a first MEMBAR 803. A second set of memory transactions tagged with a coalescing index of 1, e.g., a store 804, store 805, and a MEMBAR 806, follow the first set of 10 memory transactions. A third set of memory transactions tagged with a coalescing index of 0, e.g., a load 807, load 808, and a MEMBAR 809, follow the second set of memory transactions. A fourth set of memory transactions tagged with a coalescing index of 1, e.g., a store 810 and store 811 follow 15 the third set of memory transactions.

> 
图8A是根据本发明一个实施例的、图示说明图7A中的带标签内存命令流缓冲区718的概念图。MEMBAR接受单元705构造了带标签内存命令流缓冲区718，其中包含以合并索引0标记的第一组内存事务，例如加载801、存储803和第一个MEMBAR 803。以合并索引1标记的第二组内存事务，例如存储804、存储805和MEMBAR 806，跟随在第一组10个内存事务之后。以合并索引0标记的第三组内存事务，例如加载807、加载808和MEMBAR 809，跟随在第二组内存事务之后。以合并索引1标记的第四组内存事务，例如存储810和存储811，跟随在第三组内存事务之后。




FIG. 8B is another flowchart 815 of method steps for processing memory barrier instructions, according to one embodiment of the present invention. Although the method steps are described in conjunction with the systems of FIGS. 20 1, 2, 3A, 3B, 3C, 7A, 7B, and 8A, persons of ordinary skill in the art will understand that any system configured to perform the method steps, in any order, is within the scope of the inventions.

> 
图8B是根据本发明一个实施例的处理内存屏障指令的方法步骤的另一流程图815。尽管结合图20 1、2、3A、3B、3C、7A、7B和8A的系统来描述方法步骤，但本领域普通技术人员将理解，任何配置为以任意顺序执行这些方法步骤的系统均在本发明的范围内。




At a step 812 the warp scheduler and instruction unit 312 25 issues memory access transactions for a thread group that are received by the memory barrier instruction execution unit 700. At step 814 the warp scheduler and instruction unit 312 issues a memory barrier instruction (MEMBAR) for a first thread group. The MEMBAR is received by the memory barrier instruction execution unit 700 and the warp scheduler and instruction unit 312 blocks the execution of memory transactions for the first thread group that are after the first memory barrier instruction in program order. However, the warp scheduler and instruction unit 312 may continue issuing memory access transactions for other thread groups.

> 
在步骤812，warp调度器和指令单元312为一个线程组发出内存访问事务，这些事务被内存屏障指令执行单元700接收。在步骤814，warp调度器和指令单元312为第一个线程组发出内存屏障指令（MEMBAR）。该MEMBAR被内存屏障指令执行单元700接收，并且warp调度器和指令单元312阻塞第一个线程组中按程序顺序位于第一条内存屏障指令之后的内存事务的执行。然而，warp调度器和指令单元312可以继续为其他线程组发出内存访问事务。




At step 816 the warp scheduler and instruction unit 312 receives a coalescing index from the memory barrier instruction execution unit 700. At step 818 the warp scheduler and instruction unit 312 determines if a MEMBAR ACK is received for the MEMBAR corresponding to the coalescing index received at step 816. When the MEMBAR ACK is received from the memory barrier instruction execution unit 700, the warp scheduler and instruction unit 312 proceeds to step 820 and releases the MEMBAR for the one or more 45 thread groups waiting for index n. After the MEMBAR is released, the warp scheduler and instruction unit 312 may resume issuing memory access transactions for the thread group. Releasing the first memory barrier allows execution of the memory transactions for the first thread group and any other thread groups coalesced with the first thread group that are after the first memory barrier instruction in program order.

> 
在步骤816，线程束调度器与指令单元312接收来自内存屏障指令执行单元700的合并索引。在步骤818，线程束调度器与指令单元312确定是否收到对应于步骤816所接收合并索引的MEMBAR ACK。当从内存屏障指令执行单元700接收到MEMBAR ACK时，线程束调度器与指令单元312进入步骤820，并为等待索引n的一个或多个线程组释放MEMBAR。释放MEMBAR后，线程束调度器与指令单元312可恢复为该线程组发出内存访问事务。释放第一个内存屏障后，可按程序顺序执行第一个线程组以及与该第一个线程组合并的任何其他线程组在第一个内存屏障指令之后的内存事务。




FIG. 8C illustrates flowcharts 825 and 850 of method steps for N-way memory barrier instruction coalescing, according to one embodiment of the present invention. In one embodiment, the steps in the flowchart 825 are performed by the MEMBAR accept unit 705 and the steps in the flowchart 850 are performed by the MEMBAR tracking unit 715. Although the method steps are described in conjunction with the systems of FIGS. 1, 2, 3A, 3B, 3C, 7A, 7B, and 8A, persons of ordinary skill in the art will understand that any system configured to perform the method steps, in any order, is within the scope of the inventions.

> 
图 8C 示出了根据本发明一个实施例的用于 N 路内存屏障指令合并的方法步骤的流程图 825 和 850。在一个实施例中，流程图 825 中的步骤由 MEMBAR 接受单元 705 执行，流程图 850 中的步骤由 MEMBAR 跟踪单元 715 执行。尽管这些方法步骤是结合图 1、图 2、图 3A、图 3B、图 3C、图 7A、图 7B 和图 8A 的系统来描述的，但本领域普通技术人员将理解，配置为以任何顺序执行这些方法步骤的任何系统都在本发明的范围内。




At step 830 the MEMBAR accept unit 705 receives a first MEMBAR for a thread group. At step 832 the MEMBAR accept unit 705 initializes the coalescing timeout counter and opens a first coalescing window. At step 834 the MEMBAR accept unit 705 outputs the coalescing index to the warp

> 
在步骤830，MEMBAR接受单元705接收针对一个线程组的第一条MEMBAR。在步骤832，MEMBAR接受单元705初始化合并超时计数器并打开第一个合并窗口。在步骤834，MEMBAR接受单元705将合并索引输出到线程束。




29 scheduler and instruction unit 312. At step 836 the MEMBAR accept unit 705 constructs a tagged memory command stream including memory transactions that are not for the first thread group and that are tagged with the coalescing index.

> 
29 调度器和指令单元312。在步骤836，MEMBAR接受单元705构建一个带标签的内存命令流，该流包括非第一个线程组的内存事务，并且这些事务被打上了合并索引的标签。




At step 838 the MEMBAR accept unit 705 determines if the first coalescing window should be closed. In one embodiment, a coalescing window is closed when the coalescing timeout has expired and when a MEMBAR ACK for the transmitted MEMBAR corresponding to the coalescing index n-1 is received from the MMU 328, where the current coalescing index is n. The MMU 328 returns a MEMBAR ACK after determining that all memory transactions that occur prior to the MEMBAR instruction corresponding to a particular coalescing index in the tagged memory command stream are committed to memory.

> 
在步骤838，MEMBAR接受单元705确定第一个合并窗口是否应该关闭。在一个实施例中，当合并超时已到期并且从MMU 328接收到对应于合并索引n-1的已发送MEMBAR的MEMBAR ACK时，合并窗口关闭，其中当前合并索引为n。MMU 328在确定标记内存命令流中与特定合并索引对应的MEMBAR指令之前的所有内存事务都已提交到内存之后，返回MEMBAR ACK。




If, at step 838 the MEMBAR accept unit 705 determines that the coalescing window cannot be closed, then the MEM-BAR accept unit 705 returns to step 836. Otherwise, at step 840, the MEMBAR accept unit 705 inserts a MEMBAR instruction that is tagged with the coalescing index into the memory command stream immediately following the last memory transaction tagged with the same coalescing index. At step 842 the MEMBAR accept unit 705 increments the coalescing index.

> 
如果在步骤838处，MEMBAR接受单元705判定合流窗口无法关闭，则MEMBAR接受单元705返回步骤836。否则，在步骤840，MEMBAR接受单元705将一个标记有合流索引的MEMBAR指令插入内存命令流中，紧跟在标记有相同合流索引的最后一条内存事务之后。在步骤842，MEMBAR接受单元705将合流索引递增。




The steps in the flowchart 825 may be performed by the MEMBAR accept unit 705 simultaneously with the steps in the flowchart 850 that are performed by the MEMBAR tracking unit 715. For example, the MEMBAR accept unit 705 may be coalescing MEMBAR instructions to generate a coalesced MEMBAR instruction to be tagged with a coalescing index n+1 while the MEMBAR tracking unit 715 transmits memory transactions tagged with a coalescing index n.

> 
流程图825中的步骤可由MEMBAR接受单元705与流程图850中由MEMBAR跟踪单元715执行的步骤同时进行。例如，当MEMBAR跟踪单元715传输带有合并索引n的内存事务时，MEMBAR接受单元705可能正在合并MEMBAR指令，以生成待标记合并索引n+1的合并后MEMBAR指令。




At step 852 the MEMBAR tracking unit 715 transmits tagged memory transactions in the tagged memory command stream to the MMU 328. At step 854 the MEMBAR tracking 35 unit 715 determines if the next memory command in the tagged memory command stream is a MEMBAR instruction, and, if not, then the MEMBAR tracking unit 715 returns to step 852. Otherwise, at step 856, the MEMBAR tracking unit 715 determines if all of the memory transactions that have 4 been transmitted have been acknowledged by the MMU 328, i.e., if the ACK count for the coalescing index equals zero. When the MEMBAR tracking unit 715 determines that all of the memory transactions that have been transmitted have been acknowledged by the MMU 328, the MEMBAR track- 4: ing unit 715 transmits the tagged MEMBAR instruction to the MMU 328 at step 858.

> 
在步骤852，MEMBAR跟踪单元715将带标签的内存命令流中的带标签内存事务传输到MMU 328。在步骤854，MEMBAR跟踪 35 单元715判断带标签的内存命令流中的下一条内存命令是否为MEMBAR指令，如果不是，则MEMBAR跟踪单元715返回步骤852。否则，在步骤856，MEMBAR跟踪单元715判断所有已传输 4 的内存事务是否都已被MMU 328确认，即合并索引的ACK计数是否等于零。当MEMBAR跟踪单元715确定所有已传输的内存事务都已被MMU 328确认后，MEMBAR跟踪 4: 单元715在步骤858将带标签的MEMBAR指令传输到MMU 328。




One advantage of the disclosed system and method is that when a first memory barrier is received for a first thread group execution of subsequent memory operations for the first 50 thread group are held until the first memory barrier is executed. Subsequent memory barriers for different thread groups may be coalesced with the first memory barrier to produce a coalesced memory barrier that represents memory barriers for multiple thread groups. When the coalesced 55 memory barrier is being processed by the MMU 328, execution of subsequent memory operations for the first thread group and the different thread groups is suspended. However, memory operations for other thread groups that are not affected by the coalesced memory barrier instruction may be transmitted to the MMU 328 for execution. Additionally, compared with a system that only supports output of a single coalesced memory barrier to the MMU 328, supporting the processing of at least two coalesced memory barriers ensures that the memory barrier instruction is accepted into a coalesc- 6 ing window rather than being deferred, replayed, or otherwise stalled. Acceptance of the memory barrier instructions allows

> 
所公开系统和方法的一个优点在于，当为第一线程组收到第一内存屏障时，该第一线程组后续内存操作的执行将保持，直到第一内存屏障执行完毕。针对不同线程组的后续内存屏障可与第一内存屏障合并，生成一个代表多个线程组内存屏障的合并内存屏障。当该合并内存屏障正由MMU 328处理时，第一线程组及所述不同线程组的后续内存操作执行将被暂停。然而，不受该合并内存屏障指令影响的其他线程组的内存操作可被传送至MMU 328以供执行。此外，与仅支持向MMU 328输出单个合并内存屏障的系统相比，支持处理至少两个合并内存屏障可确保内存屏障指令被纳入合并窗口，而非被推迟、重放或以其他方式停顿。内存屏障指令的接受使得




## 30

subsequent instructions for thread groups that have not issued a memory barrier instruction to be processed.

> 
尚未发出内存屏障指令的线程组的后续指令得以处理。




One embodiment of the invention may be implemented as a program product for use with a computer system. The program(s) of the program product define functions of the embodiments (including the methods described herein) and can be contained on a variety of computer-readable storage media. Illustrative computer-readable storage media include, but are not limited to: (i) non-writable storage media (e.g., 10 read-only memory devices within a computer such as CDROM disks readable by a CD-ROM drive, flash memory, ROM chips or any type of solid-state non-volatile semiconductor memory) on which information is permanently stored; and (ii) writable storage media (e.g., floppy disks within a 15 diskette drive or hard-disk drive or any type of solid-state random-access semiconductor memory) on which alterable information is stored.

> 
本发明的一个实施例可被实现为一种与计算机系统结合使用的程序产品。该程序产品的程序定义了所述实施例（包括本文所述方法）的功能，并可包含于多种计算机可读存储介质上。例示性的计算机可读存储介质包括但不限于：(i) 不可写存储介质（例如，计算机内的只读存储器设备，如可由 CD-ROM 驱动器读取的 CDROM 光盘、闪存、ROM 芯片或任何类型的固态非易失性半导体存储器），信息永久存储于其上；以及 (ii) 可写存储介质（例如，软盘驱动器内的软盘或硬盘驱动器，或任何类型的固态随机存取半导体存储器），可修改信息存储于其上。




The invention has been described above with reference to specific embodiments. Persons skilled in the art, however, 20 will understand that various modifications and changes may be made thereto without departing from the broader spirit and scope of the invention as set forth in the appended claims. The foregoing description and drawings are, accordingly, to be regarded in an illustrative rather than a restrictive sense.

> 
上文已参照具体实施例对本发明进行了描述。然而，本领域技术人员 20 将理解，可以在不脱离所附权利要求中阐述的本发明更广泛的精神和范围的情况下，对其做出各种修改和变更。因此，前述说明和附图应被视为说明性而非限制性的。




What is claimed is:

> 
所要求保护的是：




1. A computer-implemented method for processing memory barrier instructions, the method comprising:

> 
1. 一种用于处理内存屏障指令的计算机实现方法，所述方法包括：




receiving a first memory barrier instruction for a first thread group that includes multiple parallel execution threads;

> 
接收针对第一线程组的第一内存屏障指令，该第一线程组包括多个并行执行线程；




blocking the execution of memory transactions for the first thread group that are subsequent to the first memory barrier instruction in program order;

> 
暂停按程序顺序位于第一个内存屏障指令之后的第一个线程组的内存事务的执行；




receiving, subsequent to the first memory barrier instruction, a first set of memory transactions and a second memory barrier instruction for at least a second thread group that includes multiple execution threads;

> 
在第一个存储器屏障指令之后，接收第一组存储器事务和针对至少第二线程组的第二存储器屏障指令，该第二线程组包括多个执行线程；




coalescing the first memory barrier instruction and the second memory barrier instruction to generate a coalesced memory barrier instruction;

> 
合并第一内存屏障指令与第二内存屏障指令以生成合并内存屏障指令；




tagging each transaction in the first set of memory transactions with a first coalescing index associated with the coalesced memory barrier instruction to generate tagged memory commands;

> 
用与合并的内存屏障指令相关联的第一合并索引标记第一组内存事务中的每个事务，以生成带标记的内存命令；




combining the tagged memory commands and the coalesced memory barrier instruction to generate a tagged memory command stream;

> 
将带标记的内存命令与合并的内存屏障指令组合，以生成带标记的内存命令流；




transmitting the tagged memory command stream and memory transactions for the first thread group that are prior to the first memory barrier instruction in program order to a memory management unit to process the memory transactions for the first thread group that are prior to the first memory barrier instruction in program order, the first set of memory transactions, the first memory barrier instruction, and the second memory barrier instruction;

> 
将标记的内存命令流以及按程序顺序位于第一个内存屏障指令之前的第一线程组的内存事务，传输至内存管理单元，以处理按程序顺序位于第一个内存屏障指令之前的第一线程组的内存事务、第一组内存事务、第一个内存屏障指令以及第二个内存屏障指令；




determining that the memory transactions for the first thread group that are prior to the first memory barrier instruction in program order and the first set of memory transactions are committed to memory; and

> 
确定在程序顺序上先于第一内存屏障指令的针对第一线程组的内存事务以及第一组内存事务已被提交到内存；并且




releasing both the first memory barrier instruction to allow the memory transactions for the first thread group that are subsequent to the first memory barrier instruction in program order to be executed and the second memory barrier instruction to allow the memory transactions for the second thread group that are subsequent to the second memory barrier instruction in program order to be executed.

> 
释放第一个内存屏障指令以允许按程序顺序位于该指令之后的第一线程组的内存事务得以执行，以及释放第二个内存屏障指令以允许按程序顺序位于该指令之后的第二线程组的内存事务得以执行。




## 31

2. The method of claim 1, further comprising, in response to receiving the first memory barrier instruction, outputting a memory barrier accept signal that includes the first coalescing index.

> 
2. 根据权利要求1所述的方法，还包括，响应于接收到所述第一内存屏障指令，输出包含所述第一合并索引的内存屏障接受信号。




3. The method of claim 2, further comprising:

> 
根据权利要求2所述的方法，还包括：




receiving a third memory barrier instruction for a third thread group that includes multiple parallel execution threads;

> 
接收用于第三线程组的第三内存屏障指令，所述第三线程组包括多个并行执行线程；




blocking the execution of memory transactions for the third thread group that are subsequent to the third memory barrier instruction in program order;

> 
在程序顺序上阻塞第三线程组在第三内存屏障指令之后的内存事务执行；




receiving, subsequent to the third memory barrier instruction, a second set of memory transactions and a fourth memory barrier instruction for at least a fourth thread group that includes multiple execution threads;

> 
在第三内存屏障指令之后，接收用于至少一个包括多个执行线程的第四线程组的第二组内存事务和第四内存屏障指令；




coalescing the third memory barrier instruction and the fourth memory barrier instruction to generate a second coalesced memory barrier instruction;

> 
将第三内存屏障指令和第四内存屏障指令合并，以生成第二合并内存屏障指令；




tagging each transaction in the second set of memory transactions with a second coalescing index associated with the second coalesced memory barrier instruction to generate second tagged memory commands;

> 
使用与第二个合并后的内存屏障指令关联的第二个合并索引来标记第二组内存事务中的每个事务，以生成第二组标记内存命令；




combining the second tagged memory commands and the second coalesced memory barrier instruction to generate a second taqqed memory command stream; 25

> 
将第二标记的内存命令与第二合并内存屏障指令相结合，以生成第二 taqqed 内存命令流； 25




transmitting the second tagged memory command stream and memory transactions for the third thread group that are prior to the third memory barrier instruction in program order to the memory management unit to process the memory transactions for the third thread group that are prior to the third memory barrier instruction in program order, the second set of memory transactions, the third memory barrier instruction, and the fourth memory barrier instruction;

> 
将第二带标签的内存命令流以及第三线程组中在程序顺序上位于第三内存屏障指令之前的内存事务传输到内存管理单元，以处理第三线程组中在程序顺序上位于第三内存屏障指令之前的内存事务、第二组内存事务、第三内存屏障指令以及第四内存屏障指令；




determining that the memory transactions for the third 35 thread group that are prior to the third memory barrier instruction in program order and the second set of memory transactions are committed to memory; and

> 
确定第三35线程组的按程序顺序在第三内存屏障指令之前的内存事务以及第二组内存事务被提交到内存；并且




releasing both the third memory barrier instruction to allow the memory transactions for the third thread group that are subsequent to the third memory barrier instruction in program order to be executed and the fourth memory barrier instruction to allow the memory transactions for the fourth thread group that are subsequent to the fourth memory barrier instruction in program order to be executed.

> 
释放第三内存屏障指令和第四内存屏障指令，前者允许第三线程组中在程序顺序上位于该第三内存屏障指令之后的内存事务被执行，后者允许第四线程组中在程序顺序上位于该第四内存屏障指令之后的内存事务被执行。




4. The method of claim 3, further comprising, in response to receiving the third memory barrier instruction, outputting a second memory barrier accept signal that includes the second coalescing index.

> 
4. 根据权利要求3所述的方法，还包括：响应于接收到所述第三内存屏障指令，输出包括所述第二合并索引的第二内存屏障接受信号。




5. The method of claim 3, wherein the second tagged memory command stream and memory transactions for the third thread group that are prior to the third memory barrier instruction in program order are transmitted to the memory management unit prior to determining that the memory trans- 5: actions for the first thread group that are prior to the first memory barrier instruction in program order and the first set of memory transactions are committed to memory.

> 
5. 根据权利要求3所述的方法，其中，在确定所述第一线程组的在程序顺序上位于所述第一内存屏障指令之前的内存事务以及所述第一组内存事务被提交到内存之前，将所述第二带标签的内存命令流以及所述第三线程组的在程序顺序上位于所述第三内存屏障指令之前的内存事务发送到所述内存管理单元。




6. The method of claim 1, wherein the step of determining comprises waiting for a memory barrier acknowledgement signal from the memory management unit that indicates that the memory transactions for the first thread group that are prior to the first memory barrier instruction in program order and the first set of memory transactions are committed to memory.

> 
6. 根据权利要求1所述的方法，其中，所述确定步骤包括等待来自所述内存管理单元的内存屏障确认信号，该信号指示所述第一线程组的在程序顺序上位于所述第一内存屏障指令之前的内存事务以及所述第一组内存事务已被提交至内存。




7. The method of claim 1, further receiving an acknowledgement from the memory management unit once the coa-

> 
7. 根据权利要求1所述的方法，进一步包括一旦存储器管理单元完成该合并




## 32

lesced memory barrier instruction has been processed prior to releasing the first memory barrier instruction and the second memory barrier instruction.

> 
合并后的内存屏障指令已在释放第一个内存屏障指令和第二个内存屏障指令之前被处理。




8. A computing system, comprising:

> 
8. 一种计算系统，包括：




a memory; and

> 
一个存储器；以及




a parallel processing subsystem coupled to the memory and comprising:

> 
一种耦合至所述存储器的并行处理子系统，包括：




an instruction scheduling unit configured to:

> 
一种指令调度单元，其被配置为：




issue for execution a first memory barrier instruction for a first thread group that includes multiple parallel execution threads;

> 
发出以供执行针对第一线程组的第一内存屏障指令，该第一线程组包括多个并行执行线程；




issue for execution, subsequent to the first memory barrier instruction, a first set of memory transactions and a second memory barrier instruction for at least a second thread group that includes multiple execution threads;

> 
继第一条内存屏障指令之后，发出至少一个第二线程组（包括多个执行线程）的第一组内存事务和第二条内存屏障指令以供执行；




block the execution of memory transactions for the first thread group that are subsequent to the first memory barrier instruction in program order;

> 
阻止第一个线程组中在程序顺序上位于第一条内存屏障指令之后的内存事务的执行；




block the execution of memory transactions for the second thread group that are subsequent to the second memory barrier instruction in program order; and

> 
阻止第二线程组中在程序顺序上位于第二内存屏障指令之后的内存事务的执行；以及




release both the first memory barrier instruction to allow execution of the memory transactions for the first thread group that are subsequent to the first memory barrier instruction in program order and the second memory barrier instruction to allow execution of the memory transactions for the second thread group that are subsequent to the second memory barrier instruction in program order when an acknowledgement signal is received;

> 
当收到确认信号时，释放第一个内存屏障指令，以允许程序顺序中位于该指令之后的第一线程组的内存事务执行，并释放第二个内存屏障指令，以允许程序顺序中位于该指令之后的第二线程组的内存事务执行；




a memory management unit configured to process memory transactions and memory barrier instructions; and

> 
内存管理单元，配置为处理内存事务和内存屏障指令；以及




a memory barrier instruction execution unit that is configured to:

> 
一种存储器屏障指令执行单元，其被配置为：




receive the first memory barrier instruction;

> 
接收第一条内存屏障指令；




receive the first set of memory transactions and the second memory barrier instruction;

> 
接收第一组内存事务和第二内存屏障指令；




coalesce the first memory barrier instruction and the second memory barrier instruction to generate a coalesced memory barrier instruction;

> 
将第一个内存屏障指令和第二个内存屏障指令合并，以生成合并后的内存屏障指令；




tag each transaction in the first set of memory transactions with a first coalescing index associated with the coalesced memory barrier instruction to generate tagged memory commands; and

> 
用与合并内存屏障指令相关联的第一合并索引标记第一组内存事务中的每个事务，以生成带标记的内存命令；以及




combine the tagged memory commands and the coalesced memory barrier instruction to generate a tagged memory command stream.

> 
将带标记的内存命令和合并的内存屏障指令组合，以生成带标记的内存命令流。




9. The computing system of claim 8, wherein the memory barrier instruction execution unit is further configured to:

> 
9. 根据权利要求8所述的计算系统，其中，所述内存屏障指令执行单元还被配置为：




transmit the tagged memory command stream and memory transactions for the first thread group that are prior to the first memory barrier instruction in program order to a memory management unit to process the memory transactions for the first thread group that are prior to the first memory barrier instruction in program order, the first set of memory transactions, the first memory barrier instruction, and the second memory barrier instruction;

> 
将带标记的内存命令流以及第一线程组中在程序顺序上位于第一内存屏障指令之前的内存事务传送至内存管理单元，以处理第一线程组中在程序顺序上位于第一内存屏障指令之前的内存事务、第一组内存事务、第一内存屏障指令及第二内存屏障指令；




determine that the memory transactions for the first thread group that are prior to the first memory barrier instruction in program order and the first set of memory transactions are committed to memory; and

> 
确定按程序顺序在第一内存屏障指令之前针对第一个线程组的内存事务以及第一组内存事务被提交到内存；并且




transmit the acknowledgement signal to the instruction scheduling unit.

> 
将确认信号传输至指令调度单元。




10. The computing system of claim 8, wherein the memory barrier instruction execution unit is further configured to, in

> 
10. 根据权利要求8所述的计算系统，其中，所述内存屏障指令执行单元还被配置成，在




## 33

response to receiving the first memory barrier instruction, output a memory barrier accept signal that includes the first coalescing index.

> 
响应于接收到第一条内存屏障指令，输出一个包含第一合并索引的内存屏障接受信号。




11. A processing subsystem comprising:

> 
11. 一种处理子系统，包括：




an instruction scheduling unit configured to:

> 
指令调度单元，被配置为：




issue for execution a first memory barrier instruction for a first thread group that includes multiple parallel execution threads;

> 
开始执行针对第一线程组的第一内存屏障指令，该线程组包含多个并行执行线程；




issue for execution, subsequent to the first memory barrier instruction, a first set of memory transactions and a second memory barrier instruction for at least a second thread group that includes multiple execution threads;

> 
在第一记忆屏障指令之后，发布执行第一组内存事务以及针对至少包括多个执行线程的第二线程组的第二记忆屏障指令；




block the execution of memory transactions for the first thread group that are subsequent to the first memory barrier instruction in program order;

> 
阻止第一个线程组中在程序顺序上位于第一个内存屏障指令之后的内存事务的执行；




block the execution of memory transactions for the second thread group that are subsequent to the second memory barrier instruction in program order; and

> 
阻止对于第二线程组中在程序顺序上位于第二内存屏障指令之后的内存事务的执行；并且




release both the first memory barrier instruction to allow execution of the memory transactions for the first thread group that are subsequent to the first memory barrier instruction in program order and the second memory barrier instruction to allow execution of the memory transactions for the second thread group that are subsequent to the second memory barrier instruction in program order when an acknowledgement signal is received;

> 
当接收到确认信号时，释放第一内存屏障指令以允许执行第一线程组中在程序顺序上位于该第一内存屏障指令之后的内存事务，以及释放第二内存屏障指令以允许执行第二线程组中在程序顺序上位于该第二内存屏障指令之后的内存事务；




a memory management unit configured to process memory transactions and memory barrier instructions; and

> 
一个存储器管理单元，被配置为处理存储器事务和存储器屏障指令；以及




a memory barrier instruction execution unit that is configured to:

> 
一种存储器屏障指令执行单元，其被配置为：




receive the first memory barrier instruction;

> 
接收第一条内存屏障指令；




receive the first set of memory transactions and the second memory barrier instruction;

> 
接收第一组内存事务和第二内存屏障指令；




coalesce the first memory barrier instruction and the second memory barrier instruction to generate a coalesced memory barrier instruction

> 
将第一个内存屏障指令和第二个内存屏障指令合并，以生成一个合并后的内存屏障指令




tag each transaction in the first set of memory transactions with a first coalescing index associated with the coalesced memory barrier instruction to generate tagged memory commands; and

> 
用与合并内存屏障指令相关联的第一合并索引，标记第一组内存事务中的每个事务，以生成带标签的内存命令；以及




combine the tagged memory commands and the coalesced memory barrier instruction to generate a tagged memory command stream.

> 
将带标记的内存命令与合并且的内存屏障指令相结合，生成一个带标记的内存命令流。




12. The processing subsystem of claim 11, wherein the memory barrier instruction execution unit is further configured to, in response to receiving the first memory barrier instruction, output a memory barrier accept signal that includes the first coalescing index.

> 
12. 根据权利要求11所述的处理子系统，其中所述存储器屏障指令执行单元还被配置为，响应于接收到所述第一存储器屏障指令，输出包括所述第一合并索引的存储器屏障接受信号。




13. The processing subsystem of claim 12, wherein the memory barrier instruction execution unit is further configured to:

> 
13. 根据权利要求12所述的处理子系统，其中，所述存储器屏障指令执行单元还被配置为：




receive a third memory barrier instruction for a third thread group that includes multiple execution threads;

> 
接收用于包括多个执行线程的第三线程组的第三内存屏障指令；




receive a second set of memory transactions and a second memory barrier instruction for a fourth thread group that includes multiple execution threads;

> 
接收第四线程组的第二组内存事务和第二内存屏障指令，所述第四线程组包含多个执行线程；




## 34

coalesce the third memory barrier instruction and the fourth second memory barrier instruction to generate a second coalesced memory barrier instruction;

> 
将第三条内存屏障指令与第四条第二个内存屏障指令合并，以生成第二条合并的内存屏障指令；




tag each transaction in the second set of memory transactions with a second coalescing index associated with the second coalesced memory barrier instruction to generate second tagged memory commands; and

> 
用与第二合并内存屏障指令相关联的第二合并索引来标记第二组内存事务中的每个事务，以生成第二带标记内存命令；以及




combine the second tagged memory commands and the second coalesced memory barrier instruction to generate a second tagged memory command stream.

> 
将第二组带标签的内存命令与第二合并内存屏障指令相结合，以生成第二个带标签的内存命令流。




14. The processing subsystem of claim 13, wherein the memory barrier instruction execution unit is further configured to, in response to receiving the third memory barrier instruction, outputting a second memory barrier accept signal that includes the second coalescing index.

> 
14. 根据权利要求13所述的处理子系统，其中，所述内存屏障指令执行单元被进一步配置为，响应于接收到所述第三内存屏障指令，输出包含所述第二合并索引的第二内存屏障接受信号。




15. The processing subsystem of claim 13, wherein the memory barrier instruction execution unit is further configured to transmit the second tagged memory command stream and memory transactions for the third thread group that are prior to the third memory barrier instruction in program order to the memory management unit prior to determining that the memory transactions for the first thread group that are prior to the first memory barrier instruction in program order and the first set of memory transactions are committed to memory.

> 
15. 根据权利要求13所述的处理子系统，其中，所述内存屏障指令执行单元还被配置为：在确定所述第一线程组的在程序顺序上位于所述第一内存屏障指令之前的内存事务以及所述第一组内存事务被提交到内存之前，将所述第二带标签的内存命令流以及所述第三线程组的在程序顺序上位于所述第三内存屏障指令之前的内存事务传输给所述内存管理单元。




16. The processing subsystem of claim 11, wherein the memory barrier instruction execution unit is further configured to:

> 
16. 根据权利要求11所述的处理子系统，其中，所述内存屏障指令执行单元还被配置为：




transmit the tagged memory command stream and memory transactions for the first thread group that are prior to the first memory barrier instruction in program order to a memory management unit to process the memory transactions for the first thread group that are prior to the first memory barrier instruction in program order, the first set of memory transactions, the first memory barrier instruction, and the second memory barrier instruction;

> 
将带标签的内存命令流以及按程序顺序先于第一内存屏障指令的第一线程组的内存事务传输到内存管理单元，以处理按程序顺序先于第一内存屏障指令的第一线程组的内存事务、第一组内存事务、第一内存屏障指令和第二内存屏障指令；




determine that the memory transactions for the first thread group that are prior to the first memory barrier instruction in program order and the first set of memory transactions are committed to memory; and

> 
确定针对第一线程组的、在程序顺序上先于第一内存屏障指令的内存事务以及第一组内存事务被提交至内存；并且




transmit the acknowledgement signal to the instruction scheduling unit.

> 
将确认信号传送至指令调度单元。




17. The processing subsystem of claim 11, wherein the 45 memory barrier instruction execution unit is further configured to wait for a memory barrier acknowledgement signal from the memory management unit that indicates that the memory transactions for the first thread group that are prior to the first memory barrier instruction in program order and the first set of memory transactions are committed to memory.

> 
17. 根据权利要求11所述的处理子系统，其中，所述45内存屏障指令执行单元还被配置为等待来自内存管理单元的内存屏障确认信号，该信号指示在程序顺序上先于第一内存屏障指令的用于第一线程组的内存事务以及第一组内存事务被提交到内存。




18. The processing subsystem of claim 17, the memory barrier instruction execution unit is further configured to receive an acknowledgement from the memory management unit once the coalesced memory barrier instruction has been processed prior to releasing the first memory barrier instruction and the second memory barrier instruction.

> 
18. 根据权利要求17所述的处理子系统，所述内存屏障指令执行单元还被配置为在释放所述第一内存屏障指令和所述第二内存屏障指令之前，一旦所述合并内存屏障指令已被处理，则从所述内存管理单元接收确认。
