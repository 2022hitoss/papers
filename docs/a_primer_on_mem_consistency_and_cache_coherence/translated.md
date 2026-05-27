## CHAPTER 1 Introduction to Consistency and Coherence

Many modern computer systems and most multicore chips (chip multiprocessors) support shared memory in hardware. In a shared memory system, each of the processor cores may read and write to a single shared address space. These designs seek various goodness properties, such as high performance, low power, and low cost. Of course, it is not valuable to provide these goodness properties without first providing correctness. Correct shared memory seems intuitive at a hand-wave level, but, as this lecture will help show, there are subtle issues in even defining what it means for a shared memory system to be correct, as well as many subtle corner cases in designing a correct shared memory implementation. Moreover, these subtleties must be mastered in hardware implementations where bug fixes are expensive. Even academics should master these subtleties to make it more likely that their proposed designs will work.

> 
许多现代计算机系统以及大多数多核芯片（芯片多处理器）在硬件层面支持共享内存。在共享内存系统中，每个处理器核心都可以对单一的共享地址空间进行读写操作。这类设计追求多种优良特性，如高性能、低功耗和低成本。当然，若不首先保证正确性，提供这些优良特性就毫无价值。正确的共享内存从粗略的角度来看似乎很直观，但正如本讲座将揭示的那样，即便是定义共享内存系统正确性的含义，也存在一些微妙的问题，而在设计一个正确的共享内存实现时，更有许多微妙的边界情况。此外，在硬件实现中，错误修复代价高昂，因此必须掌握这些微妙之处。即使是学者，也应精通这些细节，以提高其设计方案成功运行的可能性。




Designing and evaluating a correct shared memory system requires an architect to understand memory consistency and cache coherence, the two topics of this primer. Memory consistency (consistency, memory consistency model, or memory model) is a precise, architecturally visible definition of shared memory correctness. Consistency definitions provide rules about loads and stores (or memory reads and writes) and how they act upon memory. Ideally, consistency definitions would be simple and easy to understand. However, defining what it means for shared memory to behave correctly is more subtle than defining the correct behavior of, for example, a single-threaded processor core. The correctness criterion for a single processor core partitions behavior between one correct result and many incorrect alternatives. This is because the processor's architecture mandates that the execution of a thread transforms a given input state into a single well-defined output state, even on an out-of-order core. Shared memory consistency models, however, concern the loads and stores of multiple threads and usually allow many correct executions while disallowing many (more) incorrect ones. The possibility of multiple correct executions is due to the ISA allowing multiple threads to execute concurrently, often with many possible legal interleavings of instructions from different threads. The multitude of correct executions complicates the erstwhile simple challenge of determining whether an execution is correct. Nevertheless, consistency must be mastered to implement shared memory and, in some cases, to write correct programs that use it.

> 
设计和评估一个正确的共享内存系统要求架构师理解内存一致性和缓存一致性，这正是本入门读物的两个主题。内存一致性（一致性、内存一致性模型或内存模型）是对共享内存正确性的精确定义，且对架构可见。一致性定义提供了有关加载和存储（或内存读和写）以及它们如何作用于内存的规则。理想情况下，一致性定义应当简单且易于理解。然而，定义共享内存的正确行为比定义例如单线程处理器核心的正确行为更为微妙。单处理器核心的正确性标准将行为划分为一个正确结果和许多错误备选。这是因为处理器架构规定，即使是在乱序核心上，线程的执行也会将给定的输入状态转换为单个明确定义的输出状态。然而，共享内存一致性模型关注的是多个线程的加载和存储，通常允许多个正确的执行，同时禁止许多（更多的）错误执行。出现多个正确执行的可能性是因为指令集架构允许多个线程并发执行，且不同线程的指令之间通常有多种合法的交错方式。大量可能的正确执行使原本简单的判断执行是否正确这一挑战变得复杂。尽管如此，要实现在某些情况下，为了编写正确使用共享内存的程序，必须掌握一致性。




The microarchitecture-the hardware design of the processor cores and the shared memory system-must enforce the desired consistency model. As part of this consistency model

> 
微架构——处理器核心和共享内存系统的硬件设计——必须强制执行所需的一致性模型。作为该一致性模型的一部分




## 2 1. INTRODUCTION TO CONSISTENCY AND COHERENCE

support, the hardware provides cache coherence (or coherence). In a shared-memory system with caches, the cached values can potentially become out-of-date (or incoherent) when one of the processors updates its cached value. Coherence seeks to make the caches of a shared-memory system as functionally invisible as the caches in a single-core system; it does so by propagating a processor's write to other processors' caches. It is worth stressing that unlike consistency which is an architectural specification that defines shared memory correctness, coherence is a means to supporting a consistency model.

> 
硬件提供缓存一致性（或简称一致性）。在具有缓存的共享内存系统中，当某个处理器更新其缓存值时，缓存中的值可能会变得过时（或不一致）。一致性旨在使共享内存系统的缓存在功能上像单核系统中的缓存一样不可见；它通过将处理器的写入传播到其他处理器的缓存来实现这一点。值得强调的是，与定义共享内存正确性的架构规范——一致性不同，缓存一致性是支持一致性模型的一种手段。




Even though consistency is the first major topic of this primer, we begin in Chapter 2 with a brief introduction to coherence because coherence protocols play an important role in providing consistency. The goal of Chapter 2 is to explain enough about coherence to understand how consistency models interact with coherent caches, but not to explore specific coherence protocols or implementations, which are topics we defer until the second portion of this primer in Chapters 6-9.

> 
尽管内存一致性是本书的首要主题，但我们从第2章开始简要介绍缓存一致性，因为缓存一致性协议在提供内存一致性方面起着重要作用。第2章的目标是解释足够的缓存一致性知识，以理解内存一致性模型如何与一致性缓存交互，但不探讨具体的缓存一致性协议或实现，这些主题我们将推迟到本书第二部分第6–9章再讨论。




### 1.1 CONSISTENCY (A.K.A., MEMORY CONSISTENCY, MEMORY CONSISTENCY MODEL, OR MEMORY MODEL)

Consistency models define correct shared memory behavior in terms of loads and stores (memory reads and writes), without reference to caches or coherence. To gain some real-world intuition on why we need consistency models, consider a university that posts its course schedule online. Assume that the Computer Architecture course is originally scheduled to be in Room 152. The day before classes begin, the university registrar decides to move the class to Room 252. The registrar sends an e-mail message asking the website administrator to update the online schedule, and a few minutes later, the registrar sends a text message to all registered students to check the newly updated schedule. It is not hard to imagine a scenario-if, say, the website administrator is too busy to post the update immediately—in which a diligent student receives the text message, immediately checks the online schedule, and still observes the (old) class location Room 152. Even though the online schedule is eventually updated to Room 252 and the registrar performed the "writes" in the correct order, this diligent student observed them in a different order and thus went to the wrong room. A consistency model defines whether this behavior is correct (and thus whether a user must take other action to achieve the desired outcome) or incorrect (in which case the system must preclude these reorderings).

> 
一致性模型依据加载和存储（内存读取与写入）来定义正确的共享内存行为，而不涉及缓存或一致性。为了建立关于为何需要一致性模型的直观理解，假设有一所大学在网上发布其课程安排。假定计算机体系结构课程最初安排在152号教室。在开学前一天，大学教务主任决定将这门课移至252号教室。教务主任发送电子邮件，要求网站管理员更新在线课程表；几分钟后，他又发短信通知所有已注册的学生查看新发布的课程表。不难想象这样一种情形——比如，网站管理员因过于繁忙而未能立即发布更新——一位勤勉的学生收到短信后立即查看在线课程表，却仍看到（旧的）上课地点152号教室。尽管在线课程表最终被更新为252号教室，且教务主任按正确顺序执行了“写入”操作，但这位勤勉的学生却按不同的顺序观察到了这些操作，从而走错了教室。一致性模型定义了这种行为是正确（因此用户必须采取其他行动来获得预期结果）还是不正确（在这种情况下系统必须禁止这类重排序）。




Although this contrived example used multiple media, similar behavior can happen in shared memory hardware with out-of-order processor cores, write buffers, prefetching, and multiple cache banks. Thus, we need to define shared memory correctness-that is, which shared memory behaviors are allowed-so that programmers know what to expect and implementors know the limits to what they can provide.

> 
尽管这一人为设计的示例使用了多种媒介，但类似的行为也可能发生在共享内存硬件中，例如，带有乱序处理器核心、写缓冲区、预取机制和多缓存体的系统。因此，我们需要定义共享内存的正确性——即哪些共享内存行为是允许的——这样程序员才能知道预期行为，而实现者也能了解他们能够提供哪些行为的边界。




Shared memory correctness is specified by a memory consistency model or, more simply, a memory model. The memory model specifies the allowed behavior of multithreaded programs

> 
共享内存正确性由内存一致性模型，或更简单地，内存模型来指定。内存模型指定了多线程程序允许的行为。




1.1. CONSISTENCY 3 executing with shared memory. For a multithreaded program executing with specific input data, the memory model specifies what values dynamic loads may return and, optionally, what possible final states of the memory are. Unlike single-threaded execution, multiple correct behaviors are usually allowed, making understanding memory consistency models subtle.

> 
1.1. 一致性 3 在共享内存中执行。对于使用特定输入数据执行的多线程程序，内存模型规定了动态加载可能返回的值，以及（可选地）内存的可能最终状态。与单线程执行不同，通常允许多种正确行为，这使得理解内存一致性模型变得微妙。




Chapter 3 introduces the concept of memory consistency models and presents sequential consistency (SC), the strongest and most intuitive consistency model. The chapter begins by motivating the need to specify shared memory behavior and precisely defines what a memory consistency model is. It next delves into the intuitive SC model, which states that a multithreaded execution should look like an interleaving of the sequential executions of each constituent thread, as if the threads were time-multiplexed on a single-core processor. Beyond this intuition, the chapter formalizes SC and explores implementing SC with coherence in both simple and aggressive ways, culminating with a MIPS R10000 case study.

> 
第三章介绍了内存一致性模型的概念，并展示了顺序一致性（SC）这一最严格且最直观的一致性模型。本章首先阐述了指定共享内存行为的必要性，并精确定义了什么是内存一致性模型。接着深入探讨了直观的 SC 模型，该模型指出，多线程执行看起来应当像是各个组成线程的顺序执行交错进行，就如同这些线程在一个单核处理器上分时复用一样。在此直观理解之外，本章对 SC 进行了形式化描述，并探讨了通过缓存一致性以简单和激进两种方式实现 SC，最后以 MIPS R10000 作为案例研究。




In Chapter 4, we move beyond SC and focus on the memory consistency model implemented by x86 and historical SPARC systems. This consistency model, called total store order (TSO), is motivated by the desire to use first-in-first-out write buffers to hold the results of committed stores before writing the results to the caches. This optimization violates SC, yet promises enough performance benefit to inspire architectures to define TSO, which permits this optimization. In this chapter, we show how to formalize TSO from our SC formalization, how TSO affects implementations, and how SC and TSO compare.

> 
在第4章中，我们不再局限于顺序一致性，而是聚焦于 x86 及历史 SPARC 系统所实现的内存一致性模型。该一致性模型称为全序存储（total store order，TSO），其动机源于希望使用先进先出写缓冲来暂存已提交的存储结果，之后再写入缓存。这一优化违背了顺序一致性，却带来了足够的性能提升，从而促使架构定义出允许该优化的 TSO。在本章中，我们将展示如何从顺序一致性的形式化出发来形式化 TSO，说明 TSO 如何影响实现，并比较顺序一致性与 TSO 的异同。




Finally, Chapter 5 introduces "relaxed" or "weak" memory consistency models. It motivates these models by showing that most memory orderings in strong models are unnecessary. If a thread updates ten data items and then a synchronization flag, programmers usually do not care if the data items are updated in order with respect to each other but only that all data items are updated before the flag is updated. Relaxed models seek to capture this increased ordering flexibility to get higher performance or a simpler implementation. After providing this motivation, the chapter develops an example relaxed consistency model, called XC, wherein programmers get order only when they ask for it with a FENCE instruction (e.g., a FENCE after the last data update but before the flag write). The chapter then extends the formalism of the previous two chapters to handle XC and discusses how to implement XC (with considerable reordering between the cores and the coherence protocol). The chapter then discusses a way in which many programmers can avoid thinking about relaxed models directly: if they add enough FENCEs to ensure their program is data-race free (DRF), then most relaxed models will appear SC. With "SC for DRF," programmers can get both the (relatively) simple correctness model of SC with the (relatively) higher performance of XC. For those who want to reason more deeply, the chapter concludes by distinguishing acquires from releases, discussing write atomicity and causality, pointing to commercial examples (including an IBM Power case study), and touching upon high-level language models ( Java and C++).

> 
最后，第5章引入了“宽松”或“弱”内存一致性模型。通过展示强模型中的大多数内存排序是不必要的，来阐明这些模型的动机：如果一个线程更新了十个数据项，然后更新一个同步标志，程序员通常不关心这些数据项之间的更新顺序，而只关心在更新标志之前所有数据项都已更新。宽松模型试图捕捉这种增加的排序灵活性，以获得更高的性能或更简单的实现。在阐述这一动机之后，本章开发了一个示例宽松一致性模型XC，其中程序员仅在通过FENCE指令显式请求时才会获得顺序（例如，在最后一次数据更新之后、但在标志写入之前使用FENCE）。接着，本章扩展了前两章的形式化方法来处理XC，并讨论了如何实现XC（在核心与一致性协议之间进行大量重排序）。然后本章讨论了一种许多程序员可以避免直接思考宽松模型的方法：如果他们添加足够的FENCE来确保程序是无数据竞争的（DRF），那么大多数宽松模型将表现得如同SC。借助“针对DRF的SC”，程序员可以同时获得SC（相对）简单的正确性模型和XC（相对）更高的性能。对于想要更深入推理的读者，本章最后区分了获取和释放，讨论了写原子性和因果性，指出了商业实例（包括一个IBM Power案例研究），并涉及了高级语言模型（ Java和C++）。




Returning to the real-world consistency example of the class schedule, we can observe that the combination of an email system, a human web administrator, and a text-messaging system

> 
回到课堂时间表这一现实世界的一致性示例，我们可以观察到，电子邮件系统、人工网络管理员和短信系统的组合




## 4 1. INTRODUCTION TO CONSISTENCY AND COHERENCE

represents an extremely weak consistency model. To prevent the problem of a diligent student going to the wrong room, the university registrar needed to perform a FENCE operation after her email to ensure that the online schedule was updated before sending the text message.

> 
代表了一种极弱的存储一致性模型。为了防止勤奋的学生走错教室，大学注册员需要在发送邮件后执行一个FENCE操作，以确保在线日程在发送短信之前已更新。




### 1.2 COHERENCE (A.K.A., CACHE COHERENCE)

Unless care is taken, a coherence problem can arise if multiple actors (e.g., multiple cores) have access to multiple copies of a datum (e.g., in multiple caches) and at least one access is a write. Consider an example that is similar to the memory consistency example. A student checks the online schedule of courses, observes that the Computer Architecture course is being held in Room 152 (reads the datum), and copies this information into her calendar app in her mobile phone (caches the datum). Subsequently, the university registrar decides to move the class to Room 252, updates the online schedule (writes to the datum) and informs the students via a text message. The student's copy of the datum is now stale, and we have an incoherent situation. If she goes to Room 152, she will fail to find her class. Examples of incoherence from the world of computing, but not including computer architecture, include stale web caches and programmers using un-updated code repositories.

> 
如果不加注意，当多个参与者（例如，多个核心）可以访问同一个数据的多个副本（例如，在多个缓存中）并且至少有一次访问是写操作时，就可能出现一致性问题。来看一个与内存一致性示例类似的例子。一名学生查看在线选课系统，注意到计算机体系结构课程在152教室上课（读取了数据），并将这一信息复制到她手机中的日历应用里（缓存了该数据）。随后，大学教务员决定将该课程调至252教室，更新了在线选课系统（写入了数据），并通过短信通知了学生们。此时，这名学生所持有的数据副本已过时，我们就遇到了不一致的情形。如果她前往152教室，将找不到她的课程。计算领域中（但不包括计算机体系结构在内）的不一致例子还有过时的网络缓存，以及程序员使用未更新的代码仓库。




Access to stale data (incoherence) is prevented using a coherence protocol, which is a set of rules implemented by the distributed set of actors within a system. Coherence protocols come in many variants but follow a few themes, as developed in Chapters 6-9. Essentially, all of the variants make one processor's write visible to the other processors by propagating the write to all caches, i.e., keeping the calendar in sync with the online schedule. But protocols differ in when and how the syncing happens. There are two major classes of coherence protocols. In the first approach, the coherence protocol ensures that writes are propagated to the caches synchronously. When the online schedule is updated, the coherence protocol ensures that the student's calendar is updated as well. In the second approach, the coherence protocol propagates writes to the caches asynchronously, while still honoring the consistency model. The coherence protocol does not guarantee that when the online schedule is updated, the new value will have propagated to the student's calendar as well; however, the protocol does ensure that the new value is propagated before the text message reaches her mobile phone. This primer focuses on the first class of coherence protocols (Chapters 6-9) while Chapter 10 discusses the emerging second class.

> 
对陈旧数据（不一致性）的访问是通过一致性协议来防止的，该协议是系统内一组分布式参与者所实现的一套规则。一致性协议有多种变体，但都遵循几个主题，如第6-9章所述。从本质上讲，所有变体都是通过将写入传播到所有缓存来使一个处理器的写入对其他处理器可见，即保持日历与在线日程同步。但协议在同步发生的时间和方式上有所不同。一致性协议有两大类。在第一类方法中，一致性协议确保写入被同步传播到缓存。当在线日程更新时，一致性协议确保学生的日历也随之更新。在第二类方法中，一致性协议异步地将写入传播到缓存，同时仍遵守一致性模型。一致性协议不保证当在线日程更新时，新值也会传播到学生的日历；然而，该协议确实确保新值在短信到达她的手机之前被传播。本入门主要关注第一类一致性协议（第6-9章），而第10章则讨论新兴的第二类。




Chapter 6 presents the big picture of cache coherence protocols and sets the stage for the subsequent chapters on specific coherence protocols. This chapter covers issues shared by most coherence protocols, including the distributed operations of cache controllers and memory controllers and the common MOESI coherence states: modified (M), owned (O), exclusive (E), shared (S), and invalid (I). Importantly, this chapter also presents our table-driven methodology for presenting protocols with both stable (e.g., MOESI) and transient coherence states. Transient states are required in real implementations because modern systems rarely permit atomic transitions from one stable state to another (e.g., a read miss in state Invalid will spend some

> 
第6章从全局视角介绍缓存一致性协议，并为后续章节中讨论特定的一致性协议奠定基础。本章涵盖了大多数一致性协议所涉及的共性问题，包括缓存控制器与内存控制器的分布式操作，以及常见的MOESI一致性状态：已修改（M）、已持有（O）、独占（E）、共享（S）和无效（I）。重要的是，本章还介绍了一种表格驱动的方法论，用于描述同时包含稳定状态（如MOESI）和瞬态一致性状态的协议。在实际实现中瞬态状态是必需的，因为现代系统很少允许从一个稳定状态到另一个稳定状态的原子转换（例如，处于无效状态的读缺失会消耗一些……




1.2. COHERENCE (A.K.A., CACHE COHERENCE) 5 time waiting for a data response before it can enter state Shared). Much of the real complexity in coherence protocols hides in the transient states, similar to how much of processor core complexity hides in micro-architectural states.

> 
1.2. 一致性（又称缓存一致性） 在进入共享状态之前等待数据响应的时间。一致性协议中大部分真正的复杂性隐藏在瞬态中，类似于处理器核心的大部分复杂性隐藏在微架构状态中。




Chapter 7 covers snooping cache coherence protocols, which initially dominated the commercial market. At the hand-wave level, snooping protocols are simple. When a cache miss occurs, a core's cache controller arbitrates for a shared bus and broadcasts its request. The shared bus ensures that all controllers observe all requests in the same order and thus all controllers can coordinate their individual, distributed actions to ensure that they maintain a globally consistent state. Snooping gets complicated, however, because systems may use multiple buses and modern buses do not atomically handle requests. Modern buses have queues for arbitration and can send responses that are unicast, delayed by pipelining, or out-of-order. All of these features lead to more transient coherence states. Chapter 7 concludes with case studies of the Sun UltraEnterprise E10000 and the IBM Power5.

> 
第7章介绍监听式缓存一致性协议，这类协议最初在商用市场占据主导地位。从宏观层面看，监听协议较为简单。当发生缓存缺失时，某核心的缓存控制器会为共享总线进行仲裁，并广播其请求。共享总线确保所有控制器按相同顺序观测到所有请求，从而使所有控制器能够协调各自独立的分布式操作，以维护全局一致的状态。然而，监听机制因系统可能使用多条总线而变得复杂，且现代总线并非原子性地处理请求。现代总线设有用于仲裁的队列，并可发送单播响应，这些响应可能因流水线而延迟，或出现乱序。所有这些特性都导致了更多的暂态一致性状态。第7章最后以Sun UltraEnterprise E10000和IBM Power5的案例研究作结。




Chapter 8 delves into directory cache coherence protocols that offer the promise of scaling to more processor cores and other actors than snooping protocols that rely on broadcast. There is a joke that all problems in computer science can be solved with a level of indirection. Directory protocols support this joke: A cache miss requests a memory location from the next level cache (or memory) controller, which maintains a directory that tracks which caches hold which locations. Based on the directory entry for the requested memory location, the controller sends a response message to the requestor or forwards the request message to one or more actors currently caching the memory location. Each message typically has one destination (i.e., no broadcast or multicast), but transient coherence states abound as transitions from one stable coherence state to another stable one can generate a number of messages proportional to the number of actors in the system. This chapter starts with a basic MSI directory protocol and then refines it to handle the MOESI states E and O, distributed directories, less stalling of requests, approximate directory entry representations, and more. The chapter also explores the design of the directory itself, including directory caching techniques. The chapter concludes with case studies of the old SGI Origin 2000 and the newer AMD HyperTransport, HyperTransport Assist, and Intel QuickPath Interconnect (QPI).

> 
第8章深入探讨目录缓存一致性协议，相较于依赖广播的监听协议，此类协议有望扩展到更多的处理器核心及其他参与者。计算机科学界有一个笑话：所有问题都可以通过增加一个间接层来解决。目录协议恰好印证了这一笑话：当发生缓存缺失时，会向下一级缓存（或内存）控制器请求一个内存位置，该控制器维护一个目录，用于跟踪哪些缓存持有哪些位置。根据所请求内存位置的目录条目，控制器向请求者发送响应消息，或者将请求消息转发给当前缓存该内存位置的一个或多个参与者。每条消息通常只有一个目的地（即不进行广播或多播），但瞬态一致性状态大量存在，因为从一个稳定一致性状态转换到另一个稳定状态时，可能产生与系统中参与者数量成正比的诸多消息。本章首先介绍一个基本的MSI目录协议，然后逐步细化，以处理MOESI状态中的E和O状态、分布式目录、减少请求停顿、近似目录项表示等内容。本章还将探讨目录本身的设计，包括目录缓存技术。最后，通过案例研究回顾昔日的SGI Origin 2000，以及较新的AMD HyperTransport、HyperTransport Assist和英特尔QuickPath互连（QPI）。




Chapter 9 deals with some, but not all, of the advanced topics in coherence. For ease of explanation, the prior chapters on coherence intentionally restrict themselves to the simplest system models needed to explain the fundamental issues. Chapter 9 delves into more complicated system models and optimizations, with a focus on issues that are common to both snooping and directory protocols. Initial topics include dealing with instruction caches, multilevel caches, write-through caches, translation lookaside buffers (TLBs), coherent direct memory access (DMA), virtual caches, and hierarchical coherence protocols. Finally, the chapter delves into performance optimizations (e.g., targeting migratory sharing and false sharing) and a new protocol family called Token Coherence that subsumes directory and snooping coherence.

> 
第9章处理了一致性中的部分而非全部高级主题。为便于解释，前几章关于一致性的内容有意局限于解释基本问题所需的最简单系统模型。第9章深入探讨更复杂的系统模型和优化，重点关注对窥探和目录协议都通用的议题。初始主题包括处理指令缓存、多级缓存、写直达缓存、转换后备缓冲器（TLB）、一致直接内存访问（DMA）、虚拟缓存以及分层一致性协议。最后，本章深入性能优化（例如，针对迁移共享和伪共享）以及一种称为令牌一致性的新协议族，该协议族涵盖了目录和窥探一致性。




6 1. INTRODUCTION TO CONSISTENCY AND COHERENCE

> 
6 1. 一致性与连贯性导论




### 1.3 CONSISTENCY AND COHERENCE FOR HETEROGENEOUS SYSTEMS

Modern computer systems are predominantly heterogeneous. A mobile phone processor today not only contains a multicore CPU, it also has a GPU and other accelerators (e.g., neural network hardware). In the quest for programmability, such heterogeneous systems are starting to support shared memory. Chapter 10 deals with consistency and coherence for such heterogeneous processors.

> 
现代计算机系统主要是异构的。如今的手机处理器不仅包含多核 CPU，还集成 GPU 及其他加速器（例如神经网络硬件）。为了追求可编程性，此类异构系统开始支持共享内存。第 10 章将讨论此类异构处理器的一致性与连贯性。




The chapter starts by focusing on GPUs, arguably the most popular accelerators today. The chapter observes that GPUs originally chose not to support hardware cache coherence, since GPUs are designed for embarrassingly parallel graphics workloads that do not synchronize or share data all that much. However, the absence of hardware cache coherence leads to programmability and/or performance challenges when GPUs are used for general-purpose workloads with fine-grained synchronization and data sharing. The chapter discusses in detail some of the promising coherence alternatives that overcome these limitations—in particular, explaining why the candidate protocols enforce the consistency model directly rather than implementing coherence in a consistency-agnostic manner. The chapter concludes with a brief discussion on consistency and coherence across CPUs and the accelerators.

> 
本章首先聚焦于当前最为主流的加速器——GPU。文中指出，GPU最初选择不提供硬件缓存一致性支持，因为其设计初衷是面向高度并行的图形处理任务，这类任务几乎不需要同步或数据共享。然而，当GPU被用于具有细粒度同步和数据共享要求的通用计算负载时，缺乏硬件缓存一致性会带来可编程性和/或性能上的挑战。本章详细讨论了一些有望克服这些局限的缓存一致性替代方案——并特别解释了为什么这些候选协议会直接执行一致性模型，而非以与一致性无关的方式来实现缓存一致性。本章最后简要讨论了跨CPU与加速器的一致性模型与缓存一致性问题。




### 1.4 SPECIFYING AND VALIDATING MEMORY CONSISTENCY MODELS AND CACHE COHERENCE

Consistency models and coherence protocols are complex and subtle. Yet, this complexity must be managed to ensure that multicores are programmable and that their designs can be validated. To achieve these goals, it is critical that consistency models are specified formally. A formal specification would enable programmers to clearly and exhaustively (with tool support) understand what behaviors are permitted by the memory model and what behaviors are not. Second, a precise formal specification is mandatory for validating implementations.

> 
内存一致性模型和缓存一致性协议复杂而微妙。然而，这种复杂性必须受到管理，以确保多核处理器可被编程，且其设计能够得到验证。为实现这些目标，对一致性模型进行形式化规范至关重要。形式化规范将使程序员能够清晰且详尽地（借助工具支持）理解内存模型允许何种行为，不允许何种行为。其次，精确的形式化规范对于验证实现是必需的。




Chapter 11 starts by discussing two methods for specifying systems—axiomatic and operational-focusing on how these methods can be applied for consistency models and coherence protocols. Then the chapter goes over techniques for validating implementations-including processor pipeline and coherence protocol implementations-against their specification. The chapter discusses both formal methods and informal testing.

> 
第11章首先讨论了两种用于规范系统的方法——公理方法和操作方法——重点在于如何将这些方法应用于一致性模型和一致性协议。然后，本章介绍了根据其规范验证实现（包括处理器流水线和一致性协议实现）的技术。本章既讨论了形式化方法，也讨论了非形式化测试。




### 1.5 A CONSISTENCY AND COHERENCE QUIZ

It can be easy to convince oneself that one's knowledge of consistency and coherence is sufficient and that reading this primer is not necessary. To test whether this is the case, we offer this pop quiz.

> 
人们很容易认为自己对一致性和连贯性已有足够了解，无需阅读这份入门指南。为了检验是否真的如此，我们准备了下面的快问快答。




### 1.6. WHAT THIS PRIMER DOES NOT DO 7

Question 1: In a system that maintains sequential consistency, a core must issue coherence requests in program order. True or false? (Answer is in Section 3.8)

> 
问题1：在维护顺序一致性的系统中，核心必须按程序顺序发出一致性请求。正确还是错误？（答案在第3.8节）




Question 2: The memory consistency model specifies the legal orderings of coherence transactions. True or false? (Section 3.8)

> 
问题 2：内存一致性模型指定了一致性事务的合法顺序。对还是错？（第 3.8 节）




Question 3: To perform an atomic read-modify-write instruction (e.g., test-and-set), a core must always communicate with the other cores. True or false? (Section 3.9)

> 
问题3：要执行一个原子读-改-写指令（例如 test-and-set），核心必须始终与其他核心通信。正确还是错误？（第 3.9 节）




Question 4: In a TSO system with multithreaded cores, threads may bypass values out of the write buffer, regardless of which thread wrote the value. True or false? (Section 4.4)

> 
问题 4：在具有多线程核心的 TSO 系统中，无论值由哪个线程写入，线程都可以从写缓冲区中绕过该值。对还是错？（第 4.4 节）




Question 5: A programmer who writes properly synchronized code relative to the high-level language's consistency model (e.g., Java) does not need to consider the architecture's memory consistency model. True or false? (Section 5.9)

> 
问题 5：编写了与高级语言一致性模型（如 Java）正确同步的代码的程序员，无需考虑体系结构的内存一致性模型。对还是错？（第 5.9 节）




Question 6: In an MSI snooping protocol, a cache block may only be in one of three coherence states. True or false? (Section 7.2)

> 
问题 6：在 MSI 监听协议中，缓存块只能处于三种一致性状态之一。正确还是错误？（第 7.2 节）




Question 7: A snooping cache coherence protocol requires the cores to communicate on a bus. True or false? (Section 7.6)

> 
问题7：监听式缓存一致性协议要求各核心在一条总线上进行通信。正确还是错误？（第7.6节）




Question 8: GPUs do not support hardware cache coherence. Therefore, they are unable to enforce a memory consistency model. True or False? (Section 10.1).

> 
问题 8：GPU 不支持硬件缓存一致性。因此，它们无法强制执行内存一致性模型。正确还是错误？（第 10.1 节）




Even though the answers are provided later in this primer, we encourage readers to try to answer the questions before looking ahead at the answers.

> 
尽管答案会在本入门读物的后文中给出，但我们仍鼓励读者在查看答案之前先尝试回答这些问题。




### 1.6 WHAT THIS PRIMER DOES NOT DO

This lecture is intended to be a primer on coherence and consistency. We expect this material could be covered in a graduate class in about ten 75-minute classes (e.g., one lecture per Chapter 2 to Chapter 11).

> 
本讲座旨在作为一致性与连贯性的入门导论。我们预计这些内容可在研究生课程中大约十节75分钟的课时内完成讲授（例如，从第2章到第11章每章一节讲座）。




For this purpose, there are many things the primer does not cover. Some of these include the following.

> 
出于这一目的，本入门读物未涉及诸多内容。其中一些包括以下方面。




- Synchronization. Coherence makes caches invisible. Consistency can make shared memory look like a single memory module. Nevertheless, programmers will probably need locks, barriers, and other synchronization techniques to make their programs useful. Readers are referred to the Synthesis Lecture on Shared-Memory synchronization [2].

> 
- 同步。一致性使缓存不可见。一致性模型能使共享内存看起来像单个内存模块。然而，程序员可能仍需要锁、屏障和其他同步技术来使程序变得有用。读者可参考关于共享内存同步的Synthesis Lecture [2]。




- Commercial Relaxed Consistency Models. This primer does not cover the subtleties of the ARM, PowerPC, and RISC-V memory models, but does describe which mechanisms they provide to enforce order.

> 
- 商用宽松一致性模型。本入门读物未涵盖 ARM、PowerPC 和 RISC-V 内存模型的微妙之处，但确实描述了它们提供了哪些机制来强制顺序。




## 8 1. INTRODUCTION TO CONSISTENCY AND COHERENCE

- Parallel programming. This primer does not discuss parallel programming models, methodologies, or tools.

> 
- 并行编程。本入门指南不讨论并行编程模型、方法或工具。




- Consistency in distributed systems. This primer restricts itself to consistency within a shared memory multicore, and does not cover consistency models and their enforcement for a general distributed system. Readers are referred to the Synthesis Lectures on Database Replication [1] and Quorum Systems [3].

> 
- 分布式系统中的一致性。本入门读物仅聚焦于共享内存多核处理器内部的一致性，不涉及通用分布式系统的一致性模型及其保障机制。读者可参考《Synthesis Lectures on Database Replication》[1] 与《Quorum Systems》[3]。




### 1.7 REFERENCES

[1] B. Kemme, R. Jiménez-Peris, and M. Patiño-Martínez. Database Replication. Synthesis Lectures on Data Management. Morgan & Claypool Publishers, 2010. DOI: 10.1007/978-1-4614-8265-9_110.8

> 
[1] B. Kemme、R. Jiménez-Peris 和 M. Patiño-Martínez. 《数据库复制》. 数据管理综合讲座系列. Morgan & Claypool 出版社，2010年. DOI: 10.1007/978-1-4614-8265-9_110.8




[2] M. L. Scott. Shared-Memory Synchronization. Synthesis Lectures on Computer Architecture. Morgan & Claypool Publishers, 2013. DOI: 10.2200/s00499ed1v01y201304cac023. 7

> 
[2] M. L. Scott. 《共享内存同步》. 计算机体系结构综合讲座. Morgan & Claypool Publishers, 2013. DOI: 10.2200/s00499ed1v01y201304cac023. 7




[3] M. Vukolic. Quorum Systems: With Applications to Storage and Consensus. Synthesis Lectures on Distributed Computing Theory Morgan & Claypool Publishers, 2012. DOI: 10.2200/s00402ed1v01y201202dct009. 8

> 
[3] M. Vukolic.《Quorum Systems: With Applications to Storage and Consensus》。Synthesis Lectures on Distributed Computing Theory, Morgan & Claypool Publishers, 2012。DOI: 10.2200/s00402ed1v01y201202dct009。8
