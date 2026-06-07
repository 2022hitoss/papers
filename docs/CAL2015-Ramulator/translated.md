# Ramulator: A Fast and Extensible DRAM Simulator

${\text{ Yoongu Kim }}^{1}$ Weikun Yang ${}^{1,2}$ Onur Mutlu ${}^{1}$

> 
${\text{ Yoongu Kim }}^{1}$ Weikun Yang ${}^{1,2}$ Onur Mutlu ${}^{1}$




${}^{1}$ Carnegie Mellon University ${}^{2}$ Peking University

> 
${}^{1}$ 卡内基梅隆大学（Carnegie Mellon University） ${}^{2}$ 北京大学（Peking University）




Abstract-Recently, both industry and academia have proposed many different roadmaps for the future of DRAM. Consequently, there is a growing need for an extensible DRAM simulator, which can be easily modified to judge the merits of today's DRAM standards as well as those of tomorrow. In this paper, we present Ramulator, a fast and cycle-accurate DRAM simulator that is built from the ground up for extensibility. Unlike existing simulators, Ramulator is based on a generalized template for modeling a DRAM system, which is only later infused with the specific details of a DRAM standard. Thanks to such a decoupled and modular design, Ramulator is able to provide out-of-the-box support for a wide array of DRAM standards: DDR3/4, LPDDR3/4, GDDR5, WIO1/2, HBM, as well as some academic proposals (SALP, AL-DRAM, TL-DRAM, RowClone, and SARP). Importantly, Ramulator does not sacrifice simulation speed to gain extensibility: according to our evaluations, Ramulator is ${2.5} \times$ faster than the next fastest simulator. Ramulator is released under the permissive BSD license.

> 
摘要——最近，工业界和学术界都为动态随机存取存储器（DRAM）的未来提出了许多不同的发展路线图。因此，人们日益需要一个可扩展的DRAM模拟器，它可以方便地修改，以评判当今DRAM标准与未来DRAM标准之优劣。在本文中，我们提出了Ramulator，一个快速且周期精确（cycle-accurate）的DRAM模拟器，其从头构建即着眼于可扩展性。与现有模拟器不同，Ramulator基于一个对DRAM系统建模的通用模板，该模板随后才被注入DRAM标准的具体细节。得益于这种解耦（decoupled）与模块化（modular）设计，Ramulator能够开箱即用地支持众多DRAM标准：DDR3/4、LPDDR3/4、GDDR5、WIO1/2、HBM，以及一些学术提案（SALP、AL-DRAM、TL-DRAM、RowClone和SARP）。重要的是，Ramulator并未为了获得可扩展性而牺牲模拟速度：根据我们的评估，Ramulator比次快的模拟器快${2.5} \times$。Ramulator以宽松的BSD许可证发布。




## 1 INTRODUCTION

In recent years, we have witnessed a flurry of new proposals for DRAM interfaces and organizations. As listed in Table 1, some were evolutionary upgrades to existing standards (e.g., DDR4, LPDDR4), while some were pioneering implementations of die-stacking (e.g., WIO, HMC, HBM), and still others were academic research projects in experimental stages (e.g., Udipi et al. [38], Kim et al. [24]).

> 
近年来，我们目睹了大量关于动态随机存取存储器（DRAM）接口与组织的新提案涌现。如表1所列，其中一些是对现有标准的演进性升级（例如，第四代双倍数据率（DDR4）、低功耗第四代双倍数据率（LPDDR4）），另一些则是芯片堆叠（die-stacking）的先驱性实现（例如，宽输入输出（WIO）、混合内存立方体（HMC）、高带宽内存（HBM）），还有一些则是处于实验阶段的学术研究项目（例如，Udipi 等人 [38]、Kim 等人 [24]）。




<table><tr><td>Segment</td><td>DRAM Standards & Architectures</td></tr><tr><td>Commodity</td><td>DDR3 (2007) [14]; DDR4 (2012) [18]</td></tr><tr><td>Low-Power</td><td>LPDDR3 (2012) [17]; LPDDR4 (2014) [20]</td></tr><tr><td>Graphics</td><td>GDDR5 (2009) [15]</td></tr><tr><td>Performance</td><td>eDRAM [28], [32]; RLDRAM3 (2011) [29]</td></tr><tr><td>3D-Stacked</td><td>WIO (2011) [16]; WIO2 (2014) [21]; MCDRAM (2015) [13]; HBM (2013) [19]; HMC1.0 (2013) [10]; HMC1.1 (2014) [11]</td></tr><tr><td>Academic</td><td>SBA/SSA (2010) [38]; Staged Reads (2012) [8]; RAIDR (2012) [27]; SALP (2012) [24]; TL-DRAM (2013) [26]; RowClone (2013) [37]; Half-DRAM (2014) [39]; Row-Buffer Decoupling (2014) [33]; SARP (2014) [6]; AL-DRAM (2015) [25]</td></tr></table>

Table 1. Landscape of DRAM-based memory

> 
表 1. 基于 DRAM 的存储器 (DRAM-based memory) 格局




At the forefront of such innovations should be DRAM simulators, the software tool with which to evaluate the strengths and weaknesses of each new proposal. However, DRAM simulators have been lagging behind the rapid-fire changes to DRAM. For example, two of the most popular simulators (DRAMSim2 [36] and USIMM [7]) provide support for only one or two DRAM standards (DDR2 and/or DDR3), as listed in Table 2. Although these simulators are well suited for their intended standard(s), they were not explicitly designed to support a wide variety of standards with different organization and behavior. Instead, the simulators are implemented in a way that the specific details of one standard are integrated tightly into their codebase. As a result, researchers - especially those who are not intimately familiar with the details of an existing simulator - may find it cumbersome to implement and evaluate new standards on such simulators.

> 
在这些创新中，最前沿的应该是DRAM模拟器（DRAM simulators），这种软件工具用于评估每个新方案的优缺点。然而，DRAM模拟器一直落后于DRAM的快速变化。例如，如表2所列，两个最流行的模拟器（DRAMSim2 [36] 和 USIMM [7]）仅支持一种或两种DRAM标准 (DDR2和/或DDR3)。尽管这些模拟器非常适用于它们所针对的标准，但它们并非被明确设计为支持具有不同组织和行为的多种标准。相反，这些模拟器的实现方式是将某一标准的具体细节紧密集成到其代码库中。因此，研究人员——尤其是那些不熟悉现有模拟器细节的研究人员——可能会发现在这类模拟器上实现和评估新标准十分繁琐。




<table><tr><td>Type</td><td>Simulator</td><td>DRAM Standards</td></tr><tr><td></td><td>DRAMSim2 (2011) [36]</td><td>DDR2, DDR3</td></tr><tr><td>Standalone</td><td>USIMM (2012) [7]</td><td>DDR3</td></tr><tr><td></td><td>DrSim (2012) [22]</td><td>DDR2, DDR3, LPDDR2</td></tr><tr><td></td><td>NVMain (2012) [34]</td><td>DDR3, LPDDR3, LPDDR4</td></tr><tr><td rowspan="3">Integrated</td><td>GPGPU-Sim (2009) [3]</td><td>GDDR3, GDDR5</td></tr><tr><td>McSimA+ (2013) [2]</td><td>DDR3</td></tr><tr><td>gem5 (2014) [9]</td><td>DDR3, LPDDR3, ${\mathrm{{WIO}}}^{ * }$</td></tr></table>

*Not cycle-accurate [9].

> 
*非周期精确的[9]。*




Table 2. Survey of popular DRAM simulators

> 
表2. 流行DRAM模拟器概览




The lack of an easy-to-extend DRAM simulator is an impediment to both industrial evaluation and academic research. Ultimately, it hinders the speed at which different points in the DRAM design space can be explored and studied. As a solution, we propose Ramulator, a fast and versatile DRAM simulator that treats extensibility as a first-class citizen. Ramulator is based on the important observation that DRAM can be abstracted as a hierarchy of state-machines, where the behavior of each state-machine - as well as the aforementioned hierarchy itself - is dictated by the DRAM standard in question. From any given DRAM standard, Ramulator extracts the full specification for the hierarchy and behavior, which is then entirely consolidated into just a single class (e.g., DDR3.h/cpp). On the other hand, Ramulator also provides a standard-agnostic state-machine (i.e., DRAM.h), which is capable of being paired with any standard (e.g., DDR3.h/cpp or DDR4.h/cpp) to take on its particular hierarchy and behavior. In essence, Ramulator enables the flexibility to reconfigure DRAM for different standards at compile-time, instead of laboriously hardcoding different configurations of DRAM for different standards.

> 
缺乏一款易于扩展的DRAM仿真器对工业评估与学术研究均构成阻碍。最终，这限制了在DRAM设计空间中探索和研究不同设计点的速度。作为解决方案，我们提出Ramulator，一款快速且通用的DRAM仿真器，将可扩展性视为首要特性（first-class citizen）。Ramulator基于一个重要观察：DRAM可被抽象为状态机（state-machine）的层次结构，其中每个状态机的行为——以及上述层次结构本身——均由所讨论的DRAM标准决定。对于任意给定的DRAM标准，Ramulator提取其层次结构与行为的完整规范，随后将其完全整合到单个类（例如 DDR3.h/cpp）中。另一方面，Ramulator还提供了一种标准无关的状态机（即 DRAM.h），能够与任何标准（例如 DDR3.h/cpp 或 DDR4.h/cpp）配对，以采用其特定的层次结构与行为。本质上，Ramulator能够在编译时为不同标准灵活重构DRAM配置，而无需为不同标准繁琐地硬编码不同的DRAM配置。




The distinguishing feature of Ramulator lies in its modular design. More specifically, Ramulator decouples the logic for querying/updating the state-machines from the implementation specifics of any particular DRAM standard. As far as we know, such decoupling has not been achieved in previous DRAM simulators. Internally, Ramu-lator is structured around a collection of lookup-tables (Section 2.3), which are computationally inexpensive to query and update. This allows Ramulator to have the shortest runtime, outperforming other standalone simulators, shown in Table 2, by ${2.5} \times$ (Section 4.2). Below, we summarize the key features of Ramulator, as well as its major contributions.

> 
Ramulator 的显著特点在于其模块化设计 (modular design)。更具体地说，Ramulator 将查询/更新状态机 (state-machines) 的逻辑与任何特定 DRAM 标准 (DRAM standard) 的实现细节解耦。据我们所知，以前的 DRAM 模拟器 (DRAM simulators) 尚未实现这种解耦。内部，Ramulator 的结构围绕着一组查找表 (lookup-tables)（第2.3节），这些表的查询和更新计算成本较低。这使得 Ramulator 具有最短的运行时间 (runtime)，其性能超过其他独立模拟器 (standalone simulators)，如表2所示，速度提升 ${2.5} \times$（第4.2节）。下面，我们总结 Ramulator 的主要特点及其主要贡献。




- Ramulator is an extensible DRAM simulator providing cycle-accurate performance models for a wide variety of standards: DDR3/4, LPDDR3/4, GDDR5, WIO1/2, HBM, SALP, AL-DRAM, TL-DRAM, RowClone, and SARP. Ramulator's modular design naturally lends itself to being augmented with additional standards. For some of the standards, Ramulator is capable of reporting power consumption by relying on DRAMPower [5] as the backend.

> 
Ramulator 是一个可扩展的 DRAM 模拟器（extensible DRAM simulator），为多种标准提供周期精确的性能模型：DDR3/4、LPDDR3/4、GDDR5、WIO1/2、HBM、SALP、AL-DRAM、TL-DRAM、RowClone 和 SARP。Ramulator 的模块化设计使其自然易于扩展以支持更多标准。对于某些标准，Ramulator 能够通过依赖 DRAMPower [5] 作为后端来报告功耗。




- Ramulator is portable and easy to use. It is equipped with a simple memory controller which exposes an external API for sending and receiving memory requests. Ramulator is available in two different formats: one for standalone usage and the other for integrated usage with gem5 [4]. Ramulator is written in C++11 and is released under the permissive BSD-license [1].

> 
- Ramulator 具有可移植性且易于使用。它配备了一个简单的内存控制器（memory controller），该控制器暴露了一个外部 API，用于发送和接收内存请求。Ramulator 提供两种不同的形式：一种用于独立使用（standalone usage），另一种用于与 gem5 [4] 集成使用（integrated usage）。Ramulator 采用 C++11 编写，并以宽松的 BSD 许可证（BSD-license）发布 [1]。




## 2 RAMULATOR: HIGH-LEVEL DESIGN

Without loss of generality, we describe the high-level design of Ramulator through a case-study of modeling the widespread DDR3 standard. Throughout this section, we assume a working knowledge of DDR3, otherwise referring the reader to literature [14]. In Section 2.1, we explain how Ramulator employs a reconfigurable tree for modeling the hierarchy of DDR3. In Section 2.2, we describe the tree's nodes, which are reconfigurable state-machines for modeling the behavior of DDR3. Finally, Section 2.3 provides a closer look at the state-machines, revealing some of their implementation details.

> 
不失一般性，我们通过一个对广泛使用的DDR3标准进行建模的案例研究，描述Ramulator的高层设计。在本节中，我们假设读者具备DDR3的工作知识，否则请参考文献[14]。在第2.1节中，我们解释Ramulator如何采用可重构树 (reconfigurable tree) 来建模DDR3的层次结构 (hierarchy)。在第2.2节中，我们描述树的节点，这些节点是可重构状态机 (reconfigurable state-machines)，用于建模DDR3的行为。最后，第2.3节深入探讨状态机，揭示其部分实现细节。




### 2.1 Hierarchy of State-Machines

In Code 1 (left), we present the DRAM class, which is Ramulator's generalized template for building a hierarchy (i.e., tree) of state-machines (i.e., nodes). An instance of the DRAM class is a node in a tree of many other nodes, as is evident from its pointers to its parent node and children nodes in Code 1 (left, lines 4-6). Importantly, for the sake of modeling DDR3, we specialize the DRAM class for the DDR3 class, which is shown in Code 1 (right). An instance of the resulting specialized class (DRAM<DDR3>) is then able to assume one of the five levels that are defined by the DDR3 class.

> 
在代码1（左）中，我们展示了DRAM类，这是Ramulator用于构建状态机（即节点）层次结构（即树）的通用模板。DRAM类的实例是树中众多其他节点中的一个节点，这一点从代码1（左，第4-6行）中指向其父节点和子节点的指针即可看出。重要的是，为了对DDR3进行建模，我们将DRAM类特化为DDR3类，如代码1（右）所示。最终得到的特化类（DRAM<DDR3>）实例随后可以承担由DDR3类定义的五个层级之一。




// DRAM.h // DDR3.h/cpp

> 
// DRAM.h（动态随机存取存储器头文件）
// DDR3.h/cpp（第三代双倍数据率同步动态随机存取存储器头文件/实现文件）




template <typename T> class DDR3 \{

> 
template <typename T> class DDR3 \{




class DRAM \{ enum class Level \{

> 
```cpp
class DRAM {
    enum class Level {
```




DRAM<T>* parent; Channel, Rank, Bank,

> 
DRAM<T>* parent; 通道 (Channel), 秩 (Rank), 存储体 (Bank),




vector<DRAM<T>*> Row, Column, MAX

> 
vector<DRAM<T>*> Row, Column, MAX




children; \};

> 
children; \};




T::Level level;

> 
T::Level level;




int index;

> 
int index;




// more code... // more code...

> 
// more code...
// more code...




\}; \};

> 
\}; \};




Code 1. Ramulator's generalized template and its specialization

> 
代码1. Ramulator的泛化模板 (generalized template) 及其特化 (specialization).




In Figure 1, we visualize a fully instantiated tree, consisting of nodes at the channel, rank, and bank levels. ${}^{1}$ Instead of having a separate class for each level (DDR3_Channel, DDR3_Rank, DDR3_Bank), Ramulator simply treats a level as just another property of a node - a property that can be easily reassigned to accommodate different hierarchies with different levels. Ramulator also provides a memory controller (not shown in the figure) that interacts with the tree through only the root node (i.e., channel). Whenever the memory controller initiates a query or an operation, it results in a traversal down the tree, touching only the relevant nodes during the process. This, and more, will be explained next.

> 
在图1中，我们展示了一个完全实例化的树，它由通道 (channel)、列 (rank) 和存储体 (bank) 级别的节点组成。${}^{1}$ Ramulator 没有为每个级别提供单独的类（如 DDR3_Channel、DDR3_Rank、DDR3_Bank），而是简单地将级别视为节点的另一个属性——这个属性可以轻松重新分配，以适应具有不同级别的不同层次结构。Ramulator 还提供了一个内存控制器（图中未显示），它仅通过根节点（即通道）与树进行交互。每当内存控制器发起查询或操作时，都会导致向下遍历树，在此过程中仅触及相关节点。这一点，以及更多内容，将在接下来进行解释。




![Figure 1. Tree of DDR3 state-machines](images/fig01.jpg)

Figure 1. Tree of DDR3 state-machines

> 
图 1. DDR3 状态机树




### 2.2 Behavior of State-Machines

States. Generally speaking, a state-machine maintains a set of states, whose transitions are triggered by an external input. In Ramulator, each state-machine (i.e., node) maintains two types of states as shown in Code 2 (top, lines 5-6): status and horizon. First, status is the node's state proper, which can assume one of the statuses defined by the DDR3 class in Code 2 (bottom). The node may transition into another status when it receives one of the commands defined by the DDR3 class. Second, horizon is a lookup-table for the earliest time when each command can be received by the node. Its purpose is to prevent a node from making premature transitions between statuses, thereby honoring DDR3 timing parameters (to be explained later). We purposely neglected to mention a third state called leaf_status, because it is merely an optimization artifact - leaf_status is a sparsely populated hash-table used by a bank to track the status of its rows (i.e., leaf nodes) instead of instantiating them.

> 
状态。一般而言，状态机（state-machine）维护一组状态，其转换由外部输入触发。在 Ramulator 中，每个状态机（即节点）维护两种类型的状态，如代码 2（顶部，第 5–6 行）所示：状态（status）和时间范围（horizon）。第一，status 是节点的固有状态，可假定为代码 2（底部）中 DDR3 类定义的任一状态。当节点收到 DDR3 类定义的命令之一时，它可能转换到另一状态。第二，horizon 是一个查找表，记录节点可以接收每个命令的最早时间。其目的是防止节点在各状态间过早转换，从而遵守 DDR3 时序参数（稍后解释）。我们特意未提及第三种状态，称为叶子状态（leaf_status），因为它仅是一个优化产物——leaf_status 是一个稀疏填充的哈希表，由存储体（bank）用来跟踪其行（即叶子节点）的状态，而非实例化它们。




Functions. Code 2 (top, lines 9-11) also shows three functions that are exposed at each node: decode, check, and update. These functions are recursively defined, meaning that an invocation at the root node (by the memory controller) causes these functions to walk down the tree. In the following, we explain how the memory controller relies on these three functions to serve a memory request - in this particular example, a read request.

> 
函数。代码 2（顶部，第 9-11 行）也展示了每个节点上暴露的三个函数：decode、check 和 update。这些函数是递归定义的，意味着在根节点处（由内存控制器 (memory controller)）的调用会使这些函数沿树向下遍历。接下来，我们解释内存控制器 (memory controller) 如何依赖这三个函数来服务一个内存请求——在这个具体例子中，是一个读请求 (read request)。




1. decode(): The ultimate goal of a read request is to read from DRAM, which is accomplished by a read command. Depending on the status of the tree, however, it may not be possible to issue the read command: e.g., the rank is powered-down or the bank is closed. For a given command to a given address, ${}^{2}$ the decode function returns a "prerequisite" command that must be issued before it, if any exists: e.g., power-up or activate command.

> 
1. decode()：读请求的最终目标是从 DRAM 读取数据，这通过读命令实现。然而，取决于树的状态，可能无法发出该读命令：例如，该排 (rank) 已断电，或存储体 (bank) 已关闭。对于给定地址的给定命令，${}^{2}$ decode 函数会返回一个“先决条件”命令，该命令必须先于它被发出（如果存在的话）：例如，上电 (power-up) 或激活 (activate) 命令。




2. check(): Even if there are no prerequisites, it doesn't mean that the read command can be issued right away: e.g., the bank may not be ready if it was activated just recently. For a given command to a given address, the check function returns whether or not the command can be issued right now (i.e., current cycle).

> 
2. check()：即使没有前提条件 (prerequisites)，也并不意味着可以立即发出读取命令 (read command)：例如，存储体 (bank) 如果最近刚被激活，可能尚未准备好。对于给定地址 (address) 的给定命令 (command)，检查函数 (check function) 返回该命令是否可以立即在当前周期 (current cycle) 发出。




3. update(): If the check is passed, there is nothing preventing the memory controller from issuing the read command. For a given command to a given address, the update function triggers the necessary modifications to the status/horizon (of the affected nodes) to signify the command's issuance at the current cycle. In Ramulator, invoking the update function is issuing a command.

> 
3. update(): 如果检查通过，则没有任何阻碍内存控制器发出读取指令的因素。对于给定地址的给定指令，更新函数会触发状态/时间窗口（受影响节点的）的必要修改，以表示指令在当前周期已被发出。在 Ramulator 中，调用更新函数即意味着发出一条指令。




---

// DRAM.h

> 
// DRAM.h




template <typename T>

> 
template <typename T>




class DRAM \{

> 
class DRAM \{




// states (queried/updated by functions below)

> 
// 状态 (states)（由以下函数 (functions) 查询/更新 (queried/updated)）




T::Status status;

> 
T::Status status;




long horizon[T::Command::MAX];

> 
long horizon[T::Command::MAX];




map<int, T::Status> leaf_status; // for bank only

> 
map<int, T::Status> leaf_status; // 仅供银行 (bank) 使用




// functions (recursively traverses down tree)

> 
// 函数 (functions) (递归向下遍历树)




T::Command decode(T::Command cmd, int addr[]);

> 
T::Command decode(T::Command cmd, int addr[]);




bool check(T::Command cmd, int addr[], long now);

> 
bool check(T::Command cmd, int addr[], long now);




void update(T::Command cmd, int addr[], long now);

> 
void update(T::Command cmd, int addr[], long now);




\};

> 
\};




// DDR3.h/cpp

> 
// DDR3.h/cpp




class DDR3 \{

> 
class DDR3 {




enum class Status \{Open, Closed, ..., MAX\};

> 
enum class Status {Open, Closed, ..., MAX};




enum class Command \{ACT, PRE, RD, WR, ..., MAX\};

> 
```cpp
enum class Command {ACT, PRE, RD, WR, ..., MAX};
```




\};

> 
\};




---

Code 2. Specifying the DDR3 state-machines: states and functions

> 
代码2. 指定DDR3状态机：状态和函数




### 2.3 A Closer Look at a State-Machine

So far, we have described the role of the three functions without describing how they exactly perform their role. To preserve the standard-agnostic nature of the DRAM class, the three functions defer most of their work to the DDR3 class, which supplies them with all of the standard-dependent information in the form of three lookup-tables: (i) prerequisite, (ii) timing, and (iii) transition. Within these tables are encoded the DDR3 standard, providing answers to the following three questions: (i) which commands must be preceded by which other commands at which levels/statuses? (ii) which timing parameters at which levels apply between which commands? (iii) which commands to which levels trigger which status transitions?

> 
到目前为止，我们描述了这三个函数的作用，但未描述它们究竟如何发挥其作用。为保持 DRAM 类（DRAM class）的标准无关性（standard-agnostic nature），这三个函数将大部分工作委托给 DDR3 类（DDR3 class），后者以三张查找表的形式向它们提供所有标准依赖信息（standard-dependent information）：(i) 先决条件（prerequisite），(ii) 时序（timing），(iii) 转换（transition）。这些表中编码了 DDR3 标准（DDR3 standard），为以下三个问题提供了答案：(i) 哪些命令必须在哪些层级/状态下由哪些其他命令先行？(ii) 哪些时序参数在哪些层级下应用于哪些命令之间？(iii) 哪些命令在哪些层级触发哪些状态转换？




Decode. Due to space limitations, we cannot go into detail about all three lookup-tables. However, Code 3 (bottom) does provide a glimpse of only the first lookup-table, called prerequisite, which is consulted inside the decode function as shown in Code 3 (top). In brief, prerequisite is a two-dimensional array of lambdas (a C++11 construct), which is indexed using the (i) level in the hierarchy at which the (ii) command is being decoded. As a concrete example, Code 3 (bottom, lines 7-13) shows how one of its entries is defined, which happens to be for (i) the rank-level and (ii) the refresh command. The entry is a lambda, whose sole argument is a pointer to the rank-level node that is trying to decode the refresh command. If any of the node's children (i.e., banks) are open, the lambda returns the precharge-all command (i.e., PREA, line 11), which would close all the banks and pave the way for a subsequent refresh command. Otherwise, the lambda returns the refresh command itself (i.e., REF, line 12), signaling that no other command need be issued before it. Either way, the command has been successfully decoded at that particular level, and there is no need to recurse further down the tree. However, that may not always be the case. For example, the only reason why the rank-level node was asked to decode the refresh command was because its parent (i.e., channel) did not have enough information to do so, forcing it to invoke the decode function at its child (i.e., rank). When a command cannot be decoded at a level, the lambda returns a sentinel value (i.e., MAX), indicating that the recursion should continue on down the tree, until the command is eventually decoded by a different lambda at a lower level (or until the recursion stops at the lowest-level).

> 
解码。由于篇幅所限，我们无法详细介绍全部三个查找表。不过，代码 3（底部）确实展示了首个查找表——称为前置条件表（prerequisite）的一瞥，该表在代码 3（顶部）所示的 decode 函数内部被查询。简言之，前置条件表是一个二维 lambda 数组（C++11 构造），使用层次结构中的（i）层级和正在解码的（ii）命令作为索引。举个具体例子，代码 3（底部，第 7–13 行）展示了其中一项的定义，该项恰好对应（i）rank 层级和（ii）刷新命令（refresh command）。该条目是一个 lambda，其唯一参数是指向尝试解码刷新命令的 rank 层级节点的指针。如果该节点的任一子节点（即 bank）处于打开状态，该 lambda 返回全预充电命令（即 PREA，第 11 行），该命令将关闭所有 bank，为后续刷新命令铺平道路。否则，该 lambda 返回刷新命令本身（即 REF，第 12 行），表示无需在此之前发出其他命令。无论哪种情况，命令在该特定层级都已成功解码，无需继续向树的下层递归。然而，情况并非总是如此。例如，rank 层级节点之所以被要求解码刷新命令，唯一的原因是它的父节点（即通道，channel）没有足够的信息来解码，迫使它调用其子节点（即 rank）的 decode 函数。当命令在某一层级无法解码时，该 lambda 会返回一个哨兵值（即 MAX），表明递归应继续沿树向下进行，直到命令最终由较低层级的另一个 lambda 完成解码（或直到递归在最低层级停止）。




---

${}^{1}$ Due to their sheer number (tens of thousands), nodes at or below the row level are not instantiated. Instead, their bookkeeping is relegated to their parent - in DDR3's particular case, the bank.

> 
${}^{1}$ 由于节点数量庞大（数以万计），行级(row level)或行级以下的节点都不会被实例化。相反，它们的簿记管理(bookkeeping)被移交给了它们的父节点——在DDR3的特定情况下，即存储体(bank)。




${}^{2}$ An address is an array of node indices specifying a path down the tree.

> 
${}^{2}$ 地址是一个节点索引数组，指定了向下穿过树的路径。




---

---

// DRAM.h

> 
// 动态随机存取存储器 (DRAM).h




template <typename T>

> 
template <typename T>




class DRAM \{

> 
class DRAM \{




T::Command decode(T::Command cmd, int addr[]) \{

> 
T::Command decode(T::Command cmd, int addr[]) \{




if (prereq[level][cmd]) \{

> 
if (prereq[level][cmd]) {




// consult lookup-table to decode command

> 
// 查阅查找表 (lookup-table) 以解码命令




T::Command p = prereq[level][cmd](this);

> 
T::Command p = prereq[level][cmd](this);




if (p != T::Command::MAX)

> 
if (p != T::Command::MAX)




return p; // decoded successfully

> 
return p; // 解码成功




\}

> 
\}




if (children.size() == 0) // lowest-level

> 
if (children.size() == 0) // 最低层级




return cmd; // decoded successfully

> 
return cmd; // 解码成功




// use addr[] to identify target child...

> 
// 使用 addr[] 识别目标子节点...




// invoke decode() at the target child...

> 
// 在目标子节点处调用 decode()...




\}

> 
}




\};

> 
\};




// DDR3.h/cpp

> 
// DDR3.h/cpp




class DDR3 \{

> 
class DDR3 \{




// declare 2D lookup-table of lambdas

> 
// 声明二维的 lambda 查找表 (lookup-table)




function<Command(DRAM<DDR3>*))>

> 
function<Command(DRAM<DDR3>*))>




prereq[Level::MAX][Command::MAX];

> 
prereq[Level::MAX][Command::MAX];




// populate an entry in the table

> 
// 在表中填充一个条目




prereq[Level::Rank][Command::REF] =

> 
prereq[Level::Rank][Command::REF] =




[] (DRAM<DDR3>* node) -> Command \{

> 
[] (DRAM<DDR3>* node) -> Command \{




for (auto bank : node->children)

> 
for (auto bank : node->children)




if (bank->status == Status::Open)

> 
if (bank->status == Status::Open)




return Command::PREA;

> 
return Command::PREA;




return Command::REF;

> 
return Command::REF;




\};

> 
\};




// populate other entries...

> 
// populate other entries...




\};

> 
\};




---

Code 3. The lookup-table for decode(): prereq

> 
代码 3. decode() 的查找表：先决条件




Check & Update. In addition to prerequisite, the DDR3 class also provides two other lookup-tables: transition and timing. As is apparent from their names, they encode the status transitions and the timing parameters, respectively. Similar to prerequisite, these two are also indexed using some combination of levels, commands, and/or statuses. When a command is issued, the update function consults both lookup-tables to modify both the status (via lookups into transition) and the horizon (via lookups into timing) for all of the affected nodes in the tree. In contrast, the check function does not consult any of the lookup-tables in the DDR3 class. Instead, it consults only the horizon, the localized lookup-table that is embedded inside the DRAM class itself. More specifically, the check function simply verifies whether the following condition holds true for every node affected by a command: horizon[cmd] $\leq$ now. This ensures that the time, as of right now, is already past the earliest time at which the command can be issued. The check function relies on the update function for keeping the horizon lookup-table up-to-date. As a result, the check function is able to remain computationally inexpensive - it simply looks up a horizon value and compares it against the current time. For performance reasons, we deliberately optimized the check function to be lightweight, because it could be invoked many times each cycle - the memory controller typically has more than one memory request whose scheduling eligibility must be determined. In contrast, the update function is invoked at most once-per-cycle and can afford to be more expensive. The implementation details of the update function, as well as that of other components, can be found in the source code.

> 
检查与更新（Check & Update）。除了先决条件（prerequisite）之外，DDR3 类还提供了另外两个查找表：转换表（transition）和时序表（timing）。顾名思义，它们分别编码状态转换和时序参数。与先决条件类似，这两个表也是通过层级（level）、命令（command）和/或状态（status）的某种组合来索引的。当一条命令被发出时，更新（update）函数会同时查阅这两个查找表，以修改树中所有受影响节点的状态（通过查找转换表）和时间界限（通过查找时序表）。相比之下，检查（check）函数并不查阅 DDR3 类中的任何查找表，而只查阅时间界限——即嵌入在 DRAM 类自身内部的本地化查找表。更具体地说，检查函数仅验证对于受命令影响的每个节点，以下条件是否成立：horizon[cmd] $\leq$ now。这确保了当前时间已经超过了该命令可以被执行的最早时刻。检查函数依赖于更新函数来保持时间界限查找表的最新状态。因此，检查函数得以保持计算上的轻量——它只需查一个时间界限值并与当前时间进行比较。出于性能考虑，我们特意将检查函数优化得十分轻量，因为它每个周期可能被调用许多次——内存控制器通常有多个内存请求需要判定其调度资格。相反，更新函数最多每周期调用一次，可以承受更高的开销。更新函数以及其他组件的实现细节可在源代码中找到。




## 3 EXTENSIBILITY OF RAMULATOR

Ramulator's extensibility is a natural result of its fully-decoupled design: Ramulator provides a generalized skeleton of DRAM (i.e., DRAM.h) that is capable of being infused with the specifics of an arbitrary DRAM standard (e.g., DDR3.h/cpp). To demonstrate the extensibility of Ramulator, we describe how easy it was to add support for DDR4: (i) copy DDR3.h/cpp to DDR4.h/cpp, (ii) add BankGroup as an item in DDR4: :Level, and (iii) add or edit 20 entries in the lookup-tables -1 in prerequisite, 2 in transition, and 17 in timing. Although there were some other changes that were also required (e.g., speed-bins), only tens of lines of code were modified in total - giving a general idea about the ease at which Ramulator is extended. As far as Ramulator is concerned, the difference between any two DRAM standards is simply a matter of the difference in their lookup-tables, whose entries are populated in a disciplined and localized manner. This is in contrast to existing simulators, which require the programmer to chase down each of the hardcoded for-loops and if-conditions that are likely scattered across the codebase.

> 
Ramulator 的可扩展性是其完全解耦设计的自然结果：Ramulator 提供了一个 DRAM 的通用框架（即 DRAM.h），它能够注入任意 DRAM 标准的具体规范（例如 DDR3.h/cpp）。为了展示 Ramulator 的可扩展性，我们描述了添加 DDR4 支持是多么容易：(i) 将 DDR3.h/cpp 复制为 DDR4.h/cpp，(ii) 在 DDR4::Level 中添加 BankGroup 作为一个选项，以及 (iii) 在查找表 (lookup-table) 中添加或修改 20 个条目——1 个在前提条件、2 个在状态转换、17 个在时序方面。尽管还需要一些其他更改（例如速度等级 (speed-bins)），但总体上只修改了几十行代码——这大致体现了 Ramulator 扩展的便捷程度。就 Ramulator 而言，任意两种 DRAM 标准之间的差异仅在于其查找表的差异，而这些表的条目是以规范且本地化的方式填充的。这与现有的模拟器形成对比，后者需要程序员逐一追踪可能散布在整个代码库中的硬编码 for 循环和 if 条件。




In addition, Ramulator also provides a single, unified memory controller that is compatible with all of the standards that are supported by Ramulator (Table 2). Internally, the memory controller maintains three queues of memory requests: read, write, and maintenance. Whereas the read/write queues are populated by demand memory requests (read, write) generated by an external source of memory traffic, the maintenance queue is populated by other types of memory requests (refresh, powerdown, selfrefresh) generated internally by the memory controller as they are needed. To serve a memory request in any of the queues, the memory controller interacts with the tree of DRAM state-machines using the three functions described in Section 2.2 (i.e., decode, check, and update). The memory controller also supports several different scheduling policies that determine the priority between requests from different queues, as well as those from the same queue.

> 
此外，Ramulator 还提供了一个单一、统一的内存控制器 (memory controller)，该控制器与 Ramulator 支持的所有标准兼容（表 2）。内部地，内存控制器维护着三个内存请求队列：读、写和维护。读/写队列由外部内存流量源产生的按需内存请求 (demand memory requests)（读、写）填充，而维护队列则由内存控制器根据需要内部生成的其他类型的内存请求（刷新 (refresh)、断电 (powerdown)、自刷新 (selfrefresh)）填充。为了服务任意队列中的内存请求，内存控制器通过第 2.2 节中描述的三个函数（即解码 (decode)、检查 (check) 和更新 (update)）与 DRAM 状态机 (DRAM state-machines) 树进行交互。该内存控制器还支持多种不同的调度策略 (scheduling policies)，这些策略决定了来自不同队列的请求之间以及同一队列内请求之间的优先级。




## 4 VALIDATION & EVALUATION

As a simulator for the memory controller and the DRAM system, Ramulator must be supplied with a stream of memory requests from an external source of memory traffic. For this purpose, Ramulator exposes a simple software interface that consists of two functions: one for receiving a request into the controller, and the other for returning a request after it has been served. To be precise, the second function is a callback that is bundled inside the request. Using this interface, Ramulator provides two different modes of operation: (i) standalone mode where it is fed a memory trace or an instruction trace, and (ii) integrated mode where it is fed memory requests from an execution-driven engine (e.g., gem5 [4]). In this section, we present the results from operating Ramulator in standalone-mode, where we validate its correctness (Section 4.1), compare its performance with other DRAM simulators (Section 4.2), and conduct a cross-sectional study of contemporary DRAM standards (Section 4.3). Directions for conducting the experiments are included the source code release [1].

> 
作为内存控制器 (memory controller) 和动态随机存取存储器 (DRAM) 系统的模拟器，Ramulator 必须从外部内存流量源接收内存请求流 (memory requests)。为此，Ramulator 暴露了一个简单的软件接口，它由两个函数组成：一个用于将请求接收到控制器中，另一个用于在请求得到服务后将其返回。准确地说，第二个函数是一个捆绑在请求内部的回调 (callback)。利用此接口，Ramulator 提供两种不同的工作模式：(i) 独立模式 (standalone mode)，它接收内存访问踪迹 (memory trace) 或指令踪迹 (instruction trace)，以及 (ii) 集成模式 (integrated mode)，它接收来自执行驱动引擎 (execution-driven engine) (例如 gem5 [4]) 的内存请求。在本节中，我们展示了 Ramulator 在独立模式下运行的结果，其中我们验证其正确性（第4.1节），将其性能与其他 DRAM 模拟器 (DRAM simulators) 进行比较（第4.2节），并对当代 DRAM 标准 (contemporary DRAM standards) 进行横截面研究 (cross-sectional study)（第4.3节）。进行实验的说明包含于源代码发布 [1] 中。




### 4.1 Validating the Correctness of Ramulator

Ramulator must simulate any given stream of memory requests using a legal sequence of DRAM commands, honoring the status transitions and the timing parameters of a standard (e.g, DDR3). To validate this behavior, we created a synthetic memory trace that would stress-test Ramulator under a wide variety of command interleavings. More specifically, the trace contains 10M memory requests, the majority of which are reads and writes (9:1 ratio) to a mixture of random and sequential addresses (10:1 ratio), and the minority of which are refreshes, power-downs, and self-refreshes. ${}^{3}$ While this trace was fed into Ramulator as fast as possible (without overflowing the controller's request buffer), we collected a timestamped log of every command that was issued by Ramulator. We then used this trace as part of an RTL simulation by feeding it into Micron's DDR3 Verilog model [30] - a reference implementation of DDR3. Throughout the entire duration of the RTL simulation ( $\sim  {10}$ hours), no violations were ever reported, indicating that Ramulator's DDR3 command sequence is indeed legal. ${}^{4}$ Due to the lack of corresponding Verilog models, however, we could not employ the same methodology to validate other standards. Nevertheless, we are reasonably confident in their correctness, because we implemented them by making careful modifications to Ramulator's DDR3 model, modifications that were expressed succinctly in just a few lines of code - minimizing the risk of human error, as well as making it easy to double-check. In fact, the ease of validation is another advantage of Ramulator, arising from its clean and modular design.

> 
Ramulator 必须使用合法的 DRAM 命令序列来模拟任意给定的内存请求流 (stream of memory requests)，并遵循某一标准（例如 DDR3）的状态转换和时序参数。为验证这一行为，我们创建了一个合成的内存访问踪迹 (memory trace)，可在各种命令交织 (command interleavings) 下对 Ramulator 进行压力测试。具体来说，该踪迹包含 10M 条内存请求，其中大多数是读和写（比例为 9:1），针对随机地址和顺序地址的混合（比例为 10:1），少数是刷新、掉电 (power-downs) 和自刷新 (self-refreshes)。${}^{3}$ 在尽可能快地将此踪迹送入 Ramulator 的同时（不使控制器的请求缓冲区 (request buffer) 溢出），我们收集了 Ramulator 发出的每条命令的带时间戳日志。然后，我们将此踪迹作为 RTL 仿真 (RTL simulation) 的一部分，送入 Micron 的 DDR3 Verilog 模型 [30]——一个 DDR3 的参考实现 (reference implementation)。在整个 RTL 仿真期间（ $\sim {10}$ hours），从未报告任何违规，表明 Ramulator 的 DDR3 命令序列确实合法。${}^{4}$ 然而，由于缺乏相应的 Verilog 模型，我们无法采用同样的方法验证其他标准。尽管如此，我们对其正确性有相当的信心，因为我们在实现它们时对 Ramulator 的 DDR3 模型进行了仔细修改，这些修改仅需寥寥数行代码就能简洁地表达——最大限度地降低了人为错误的风险，并且易于复核。事实上，易于验证是 Ramulator 的另一个优势，这源于其简洁而模块化的设计。




---

${}^{3}$ We exclude maintenance-related requests which are not supported by Ramulator or other simulators: e.g., ZQ calibration and mode-register set.

> 
${}^{3}$ 我们排除了维护相关请求（maintenance-related requests），这些请求不受 Ramulator 或其他模拟器支持：例如，ZQ 校准 (ZQ calibration) 和模式寄存器设置 (mode-register set)。




${}^{4}$ This verifies that Ramulator does not issue commands too early. However, the Verilog model does not allow us to verify whether Ramulator issues commands too late.

> 
${}^{4}$ 这验证了 Ramulator 不会过早发出命令。然而，Verilog 模型不允许我们验证 Ramulator 是否会过晚发出命令。




---

### 4.2 Measuring the Performance of Ramulator

In Table 3, we quantitatively compare Ramulator with four other standalone simulators using the same experimental setup. All five were configured to simulate DDR3-16005 for two different memory traces, Random and Stream, comprising 100M memory requests (read:write=9:1) to random and sequential addresses, respectively. For each simulator, Table 3 presents four metrics: (i) simulated clock cycles, (ii) simulation runtime, (iii) simulated request throughput, and (iv) maximum memory consumption. From the table, we make three observations. First, all five simulators yield roughly the same number of simulated clock cycles, where the slight discrepancies are caused by the differences in how their memory controllers make scheduling decisions (e.g., when to issue reads vs. writes). Second, Ramulator has the shortest simulation runtime (i.e., the highest simulated request throughput), taking only 752/249 seconds to simulate the two traces - a ${2.5} \times  /{3.0} \times$ speedup compared to the next fastest simulator. Third, Ramulator consumes only a small amount of memory while it executes (2.1MB). We conclude that Ramulator provides superior performance and efficiency, as well as the greatest extensibility.

> 
在表3中，我们使用相同的实验设置定量比较了Ramulator与其他四个独立的模拟器。所有五个模拟器均被配置为模拟DDR3-16005，针对两种不同的内存访问轨迹 (memory traces)：随机 (Random) 和顺序流 (Stream)，分别包含1亿个指向随机地址和顺序地址的内存请求（读取:写入=9:1）。对于每个模拟器，表3列出了四个指标：(i) 模拟时钟周期 (simulated clock cycles)，(ii) 模拟运行时间 (simulation runtime)，(iii) 模拟请求吞吐量 (simulated request throughput)，以及 (iv) 最大内存消耗 (maximum memory consumption)。从表中，我们得出三点观察。第一，所有五个模拟器产生了大致相同的模拟时钟周期数，细微的差异源于各自内存控制器做出调度决策（例如，何时发出读取 vs. 写入）的方式不同。第二，Ramulator具有最短的模拟运行时间（即最高的模拟请求吞吐量），模拟两个内存访问轨迹仅耗时752/249秒——相比第二快的模拟器获得了${2.5} \times  /{3.0} \times$ 的加速比 (speedup)。第三，Ramulator在执行时仅消耗少量内存（2.1MB）。我们得出结论，Ramulator提供了卓越的性能和效率，以及最大的可扩展性 (extensibility)。




<table><tr><td rowspan="2">Simulator (clang -O3)</td><td colspan="2">Cycles $\left( {10}^{6}\right)$</td><td colspan="2">Runtime (sec.)</td><td colspan="2">Req/sec (103)</td><td rowspan="2">Memory (MB)</td></tr><tr><td>Random</td><td>Stream</td><td>Random</td><td>Stream</td><td>Random</td><td>Stream</td></tr><tr><td>Ramulator</td><td>652</td><td>411</td><td>752</td><td>249</td><td>133</td><td>402</td><td>2.1</td></tr><tr><td>DRAMSim2</td><td>645</td><td>413</td><td>2,030</td><td>876</td><td>49</td><td>114</td><td>1.2</td></tr><tr><td>USIMM</td><td>661</td><td>409</td><td>1,880</td><td>750</td><td>53</td><td>133</td><td>4.5</td></tr><tr><td>DrSim</td><td>647</td><td>406</td><td>18,109</td><td>12,984</td><td>6</td><td>8</td><td>1.6</td></tr><tr><td>NVMain</td><td>666</td><td>413</td><td>6,881</td><td>5,023</td><td>15</td><td>20</td><td>4,230.0</td></tr></table>

Table 3. Comparison of five simulators using two traces

> 
表 3. 使用两个轨迹对五个模拟器的比较




### 4.3 Cross-Sectional Study of DRAM Standards

With its integrated support for many different DRAM standards - some of which (e.g., LPDDR4, WIO2) have never been modeled before in academia - Ramulator unlocks the ability to perform a comparative study across them. In particular, we examine nine different standards (Table 4), whose configurations (e.g., timing) were set to reasonable values. Instead of memory traces, we collected instruction traces from 22 SPEC2006 benchmarks, 6 which were fed into a simplistic "CPU" model that comes with Ramulator. ${}^{7}$

> 
凭借对多种不同动态随机存取存储器（DRAM）标准的集成支持——其中一些标准（例如，低功耗DDR4（LPDDR4）、宽I/O 2（WIO2））此前从未在学术界被建模过——Ramulator使得能够对它们进行比较研究。我们具体考察了九种不同的标准（表4），其配置（如时序）被设置为合理的值。我们没有使用内存跟踪，而是从22个SPEC2006基准测试中收集了指令跟踪，其中6个被送入Ramulator自带的简化“CPU”模型中。${}^{7}$




<table><tr><td>Standard</td><td>Rate (MT/s)</td><td>Timing (CL-RCD-RP)</td><td>Data-Bus (Width×Chan.)</td><td>Rank-per-Chan</td><td>${BW}$ (GB/s)</td></tr><tr><td>DDR3</td><td>1,600</td><td>11-11-11</td><td>64-bit $\times  1$</td><td>1</td><td>11.9</td></tr><tr><td>DDR4</td><td>2,400</td><td>16-16-16</td><td>64-bit $\times  1$</td><td>1</td><td>17.9</td></tr><tr><td>SALP†</td><td>1,600</td><td>11-11-11</td><td>64-bit $\times  1$</td><td>1</td><td>11.9</td></tr><tr><td>LPDDR3</td><td>1,600</td><td>12-15-15</td><td>64-bit × 1</td><td>1</td><td>11.9</td></tr><tr><td>LPDDR4</td><td>2,400</td><td>22-22-22</td><td>32-bit $\times  {2}^{ * }$</td><td>1</td><td>17.9</td></tr><tr><td>GDDR5 [12]</td><td>6,000</td><td>18-18-18</td><td>64-bit $\times  1$</td><td>1</td><td>44.7</td></tr><tr><td>HBM</td><td>1,000</td><td>7-7-7</td><td>128-bit $\times  {8}^{ * }$</td><td>1</td><td>119.2</td></tr><tr><td>WIO</td><td>266</td><td>7-7-7</td><td>128-bit $\times  {4}^{ * }$</td><td>1</td><td>15.9</td></tr><tr><td>WIO2</td><td>1,066</td><td>9-10-10</td><td>128-bit $\times  {8}^{ * }$</td><td>1</td><td>127.2</td></tr></table>

${}^{ \dagger  }$ MASA [24] on top of DDR3 with 8 subarrays-per-bank.

> 
${}^{ \dagger  }$ 基于 DDR3 且每存储体配备 8 个子阵列 (8 subarrays-per-bank) 的 MASA [24]。




* More than one channel is built into these particular standards.

> 
* 这些特定标准中内置了多个信道 (channel)。




Table 4. Configuration of nine DRAM standards used in study

> 
表4. 研究中使用的九种DRAM标准配置




Figure 2 contains the violin plots and geometric means of the normalized IPC compared to the DDR3 baseline. We make several broad observations. First, newly upgraded standards (e.g., DDR4) perform better than their older counterparts (e.g., DDR3). Second, standards for embedded systems (i.e., LPDDRx, WIOx) have lower performance because they are optimized to consume less power. Third, standards for graphics systems (i.e., GDDR5, HBM) provide a large amount of bandwidth, leading to higher average performance than DDR3 even for our non-graphics benchmarks. Fourth, a recent academic proposal, SALP, provides significant performance improvement (e.g., higher than that of WIO2) by reducing the serialization effects of bank conflicts without increasing peak bandwidth. These observations are only a small sampling of the analyses that are enabled by Ramulator.

> 
图 2 展示了与 DDR3 基线相比，标准化 IPC 的小提琴图和几何平均值。我们得出几项广泛观察。首先，新升级的标准（如 DDR4）性能优于其前代标准（如 DDR3）。其次，面向嵌入式系统的标准（即 LPDDRx、WIOx）因优化功耗而性能较低。第三，图形系统标准（即 GDDR5、HBM）提供大量带宽，即使针对非图形基准测试，也带来了高于 DDR3 的平均性能。第四，近期的一项学术提案 SALP，通过降低存储体冲突的串行化效应而不增加峰值带宽，实现了显著的性能提升（例如高于 WIO2）。这些观察仅是 Ramulator 所支持分析的一小部分示例。




![Figure 2. Performance comparison of DRAM standards](images/fig02.jpg)

Figure 2. Performance comparison of DRAM standards

> 
图2. 动态随机存取存储器 (DRAM) 标准的性能比较




## 5 CONCLUSION

In this paper, we introduced Ramulator, a fast and cycle-accurate simulation tool for current and future DRAM systems. We demonstrated Ramulator's advantage in efficiency and extensibility, as well as its comprehensive support for DRAM standards. We hope that Ramulator would facilitate DRAM research in an era when main memory is undergoing rapid changes [23], [31].

> 
在本文中，我们介绍了Ramulator，一个用于当前和未来动态随机存取存储器(DRAM)系统的快速且周期精确的仿真工具。我们展示了Ramulator在效率和可扩展性方面的优势，以及其对于DRAM标准的全面支持。我们希望Ramulator能够在主内存经历快速变化的时代促进DRAM研究[23], [31]。




## REFERENCES

[1] Ramulator source code. https://github.com/CMU-SAFARI/ramulator.

> 
[1] Ramulator 源代码 (source code). https://github.com/CMU-SAFARI/ramulator.




[2] J. H. Ahn et al. McSimA+: A Manycore Simulator with Application-Level+ Simulation and Detailed Microarchitecture Modeling. In ISPASS, 2013.

> 
[2] J. H. Ahn 等. McSimA+：一款具有应用级+模拟与详细微架构建模的众核模拟器 (McSimA+: A Manycore Simulator with Application-Level+ Simulation and Detailed Microarchitecture Modeling). 收录于 ISPASS, 2013.




[3] A. Bakhoda et al. Analyzing CUDA Workloads Using a Detailed GPU Simulator. In ISPASS, 2009.

> 
[3] A. Bakhoda 等人. 使用详细的图形处理器（GPU）模拟器分析统一计算设备架构（CUDA）工作负载. 见 ISPASS, 2009.




[4] N. Binkert et al. The Gem5 Simulator. SIGARCH Comput. Archit. News, May 2011.

> 
[4] N. Binkert 等人. Gem5 模拟器 (The Gem5 Simulator). SIGARCH 计算机架构新闻 (SIGARCH Comput. Archit. News), 2011年5月.




[5] K. Chandrasekar et al. DRAMPower: Open-Source DRAM Power & Energy Estimation Tool. http://www.drampower.info, 2012.

> 
[5] K. Chandrasekar et al. DRAMPower：开源 (Open-Source) DRAM 功耗与能耗估算工具 (Power & Energy Estimation Tool)。http://www.drampower.info，2012。




[6] K. Chang et al. Improving DRAM Performance by Parallelizing Refreshes with Accesses. In HPCA, 2014.

> 
[6] K. Chang 等人. 通过并行化刷新与访问提升DRAM性能 (Improving DRAM Performance by Parallelizing Refreshes with Accesses). 发表于高性能计算机体系结构会议 (HPCA), 2014.




[7] N. Chatterjee et al. USIMM: the Utah SImulated Memory Module. UUCS-12-002, University of Utah, Feb. 2012.

> 
[7] N. Chatterjee 等. USIMM：犹他模拟内存模块 (SImulated Memory Module). UUCS-12-002，犹他大学，2012年2月.




[8] N. Chatterjee et al. Staged Reads: Mitigating the Impact of DRAM Writes on DRAM Reads. In HPCA, 2012.

> 
[8] N. Chatterjee 等人. 分阶段读取 (Staged Reads)：减轻DRAM写入对DRAM读取的影响. 收录于 HPCA, 2012.




[9] A. Hansson et al. Simulating DRAM Controllers for Future System Architecture Exploration. In ISPASS, 2014.

> 
[9] A. Hansson 等人。用于未来系统架构探索的 DRAM 控制器模拟 (Simulating DRAM Controllers for Future System Architecture Exploration)。见 IEEE 国际系统性能分析研讨会 (ISPASS)，2014 年。




[10] Hybrid Memory Cube Consortium. HMC Specification 1.0, Jan. 2013.

> 
[10] 混合内存立方体联盟 (Hybrid Memory Cube Consortium)。HMC 规范 1.0 (HMC Specification 1.0)，2013年1月。




[11] Hybrid Memory Cube Consortium. HMC Specification 1.1, Feb. 2014.

> 
[11] 混合存储立方体 (Hybrid Memory Cube) 联盟。HMC 规范 (HMC Specification) 1.1，2014年2月。




[12] Hynix. GDDR5 SGRAM H5GQ1H24AFR, Nov. 2009.

> 
[12] 海力士 (Hynix)。GDDR5 SGRAM H5GQ1H24AFR，2009年11月。




[13] James Reinders. Knights Corner: Your Path to Knights Landing, Sept. 17, 2014.

> 
[13] James Reinders. Knights Corner：通往 Knights Landing 之路，2014年9月17日。




[15] JEDEC. JESD212 GDDR5 SGRAM, Dec. 2009.

> 
[15] 联合电子设备工程委员会 (JEDEC). JESD212 第五代图形双倍数据速率同步图形随机存取存储器 (GDDR5 SGRAM)，2009年12月.




[16] JEDEC. JESD229 Wide I/O Single Data Rate (Wide/IO SDR), Dec. 2011.

> 
[16] JEDEC. JESD229 宽输入/输出单数据速率（Wide/IO SDR），2011年12月。




[17] JEDEC. JESD209-3 Low Power Double Data Rate 3 (LPDDR3), May 2012.

> 
[17] JEDEC. JESD209-3 低功耗双倍数据速率3（LPDDR3），2012年5月。




[18] JEDEC. JESD79-4 DDR4 SDRAM, Sept. 2012.

> 
[18] JEDEC. JESD79-4 第四代双倍数据速率同步动态随机存取存储器 (DDR4 SDRAM)，2012年9月。




[19] JEDEC. JESD235 High Bandwidth Memory (HBM) DRAM, Oct. 2013.

> 
[19] JEDEC. JESD235 高带宽内存 (High Bandwidth Memory, HBM) DRAM, 2013年10月.




[20] JEDEC. JESD209-4 Low Power Double Data Rate 3 (LPDDR4), Aug. 2014.

> 
[20] 固态技术协会 (JEDEC). JESD209-4 低功耗双倍数据速率3 (LPDDR4), 2014年8月.




[21] JEDEC. JESD229-2 Wide I/O 2 (WideIO2), Aug. 2014.

> 
[21] 联合电子器件工程委员会 (JEDEC). JESD229-2 Wide I/O 2 (WideIO2)，2014年8月。




[22] M. K. Jeong et al. DrSim: A Platform for Flexible DRAM System Research. http://lph.ece.utexas.edu/public/DrSim, 2012.

> 
[22] M. K. Jeong 等人。DrSim：一个用于灵活的动态随机存取存储器 (DRAM) 系统研究的平台。http://lph.ece.utexas.edu/public/DrSim, 2012。




[23] U. Kang et al. Co-Architecting Controllers and DRAM to Enhance DRAM Process Scaling. In The Memory Forum (Co-located with ISCA), 2014.

> 
[23] U. Kang 等人. 共同架构控制器与DRAM以增强DRAM工艺微缩 (Co-Architecting Controllers and DRAM to Enhance DRAM Process Scaling). 在内存论坛 (The Memory Forum，与ISCA合办), 2014.




[24] Y. Kim et al. A Case for Exploiting Subarray-Level Parallelism (SALP) in DRAM. In ISCA, 2012.

> 
[24] Y. Kim 等人. 在DRAM中利用子阵列级并行（SALP）的一个案例. 在ISCA, 2012.




25] D. Lee et al. Adaptive-Latency DRAM: Optimizing DRAM Timing for the Common-Case. In HPCA, 2015.

> 
25] D. Lee 等人，《自适应延迟 DRAM（Adaptive-Latency DRAM）：面向常见情况优化 DRAM 时序》，发表于 HPCA，2015 年。




[26] D. Lee et al. Tiered-Latency DRAM: A Low Latency and Low Cost DRAM Architecture. In HPCA, 2013.

> 
[26] D. Lee 等人。分层延迟DRAM (Tiered-Latency DRAM)：一种低延迟、低成本的DRAM架构。在HPCA会议上，2013年。




[27] J. Liu et al. RAIDR: Retention-Aware Intelligent DRAM Refresh. In ISCA, 2012.

> 
[27] J. Liu 等人. RAIDR：保留感知的智能DRAM刷新 (RAIDR: Retention-Aware Intelligent DRAM Refresh). 见国际计算机体系结构研讨会 (ISCA), 2012.




[28] M. Meterelliyoz et al. 2nd Generation Embedded DRAM with 4X Lower Self Refresh Power in 22nm Tri-Gate CMOS Technology. In VLSI Symposium, 2014.

> 
[28] M. Meterelliyoz 等. 采用 22nm 三栅极 (Tri-Gate) CMOS 技术实现自刷新功耗 (Self Refresh Power) 降低 4 倍的第二代嵌入式 DRAM (Embedded DRAM). 载于 VLSI 研讨会 (VLSI Symposium), 2014.




[29] Micron. Micron Announces Sample Availability for Its Third-Generation RLDRAM(R) Memory. http://investors.micron.com/releasedetail.cfm?ReleaseID=581168, May 26, 2011.

> 
[29] 美光 (Micron)。美光宣布其第三代 RLDRAM(R) 存储器提供样片。http://investors.micron.com/releasedetail.cfm?ReleaseID=581168，2011 年 5 月 26 日。




[30] Micron. DDR3 SDRAM Verilog model, 2012.

> 
[30] 美光 (Micron). DDR3 SDRAM Verilog 模型, 2012.




[31] O. Mutlu. Memory Scaling: A Systems Architecture Perspective. MemCon, 2013.

> 
[31] O. Mutlu. 内存扩展：系统架构视角 (Memory Scaling: A Systems Architecture Perspective). MemCon, 2013.




[32] S. Narasimha et al. 22nm High-Performance SOI Technology Featuring Dual-Embedded Stressors, Epi-Plate High-K Deep-Trench Embedded DRAM and Self-Aligned Via 15LM BEOL. In IEDM, 2012.

> 
[32] S. Narasimha 等. 22nm 高性能绝缘层上硅 (SOI) 技术，具有双嵌入式应力源 (Dual-Embedded Stressors)、外延平板高 k 深沟嵌入式 DRAM (Epi-Plate High-K Deep-Trench Embedded DRAM) 和自对准通孔 15 层金属后段制程 (Self-Aligned Via 15LM BEOL). 见 IEDM, 2012.




[33] S. O et al. Row-Buffer Decoupling: A Case for Low-latency DRAM Microarchitecture. In ISCA, 2014.

> 
[33] S. O 等人. 行缓冲解耦：一种面向低延迟 DRAM 微架构的方案 (Row-Buffer Decoupling: A Case for Low-latency DRAM Microarchitecture). 见 国际计算机体系结构研讨会 (ISCA), 2014.




[34] M. Poremba and Y. Xie. NVMain: An Architectural-Level Main Memory Simulator for Emerging Non-volatile Memories. In ISVLSI, 2012.

> 
[34] M. Poremba 和 Y. Xie. NVMain：一种面向新兴非易失性存储器的架构级主存模拟器 (NVMain: An Architectural-Level Main Memory Simulator for Emerging Non-volatile Memories). 载于 ISVLSI, 2012.




[35] S. Rixner et al. Memory Access Scheduling. In ISCA, 2000.

> 
[35] S. Rixner et al. 内存访问调度 (Memory Access Scheduling). In ISCA, 2000.




[36] P. Rosenfeld et al. DRAMSim2: A Cycle Accurate Memory System Simulator. CAL, 2011.

> 
[36] P. Rosenfeld 等人. DRAMSim2：一种周期精确的内存系统模拟器 (Cycle Accurate Memory System Simulator). CAL, 2011.




[37] V. Seshadri et al. RowClone: Fast and Efficient In-DRAM Copy and Initialization of Bulk Data. In MICRO, 2013.

> 
[37] V. Seshadri 等人。RowClone：快速高效的 DRAM 内批量数据复制与初始化。收录于 MICRO，2013。




[38] A. N. Udipi et al. Rethinking DRAM Design and Organization for Energy-Constrained Multi-Cores. In ISCA, 2010.

> 
[38] A. N. Udipi 等人. 重新思考面向能耗受限多核的动态随机存取存储器 (DRAM) 设计与组织. 载于 ISCA, 2010.




[39] T. Zhang et al. Half-DRAM: A High-Bandwidth and Low-Power DRAM Architecture from the Rethinking of Fine-Grained Activation. In ISCA, 2014.

> 
[39] T. Zhang 等. Half-DRAM：一种基于细粒度激活再思考的高带宽、低功耗DRAM架构 (Half-DRAM: A High-Bandwidth and Low-Power DRAM Architecture from the Rethinking of Fine-Grained Activation). 发表于ISCA, 2014.




---

${}^{5}$ Single rank, ${800}\mathrm{{Mhz}},{11} - {11} - {11}$ , row-interleaved, FR-FCFS [35], open-row policy. ${}^{6}$ perlbench, bwaves, gamess, povray, calculix, tonto were unavailable for trace collection.

> 
${}^{5}$ 单列 (Single rank)，${800}\mathrm{Mhz}$，${11} - {11} - {11}$，行交错 (row-interleaved)，先就绪先来先服务 (FR-FCFS) [35]，开放行策略 (open-row policy)。${}^{6}$ perlbench、bwaves、gamess、povray、calculix、tonto 无法用于跟踪收集 (trace collection)。




${}^{7}{3.2}\mathrm{{GHz}},4$ -wide issue,128-entry ROB, no instruction-dependency, one cycle for non-DRAM instructions, instruction trace is pre-filtered through a 512KB cache, memory controller has 32/32 entries in its read/write request buffers.

> 
${}^{7}{3.2}\mathrm{GHz}$，4发射，128项重排序缓冲(ROB)，无指令依赖，非DRAM指令需一个周期，指令跟踪通过512KB缓存预过滤，内存控制器的读/写请求缓冲区各有32个条目。




---
