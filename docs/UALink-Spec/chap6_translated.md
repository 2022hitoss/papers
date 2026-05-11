## 10 UALink Switch Requirements

> 
## 10 UALink 交换机要求




### 10.1 Overview

UALink Switches serve to relay UALink Requests and Responses between Accelerators.

> 
UALink 交换机用于在 UALink 互连结构中转发加速器之间的请求与响应。本文档主要规定这些交换机的架构与行为规范，以确保实现高效、无损、低延迟的通信。

主要贡献包括：仅支持单播转发，不存在交换机到交换机的链路；利用基于目标加速器 ID 的路由表，对从 TL 微片解包出的各个独立请求和响应进行独立路由；完整实现 UALink 数据链路层和事务处理层，并提供与 UPLI 兼容的接口；支持将每个站点的链路分叉为多个端口。交换机必须通过链路级重传和流量控制提供无损交付，采用无阻塞架构以防止被阻塞的请求阻塞响应，并在所有资源上实现无饥饿仲裁。排序遵循 UPLI 规则，强制要求严格排序模式，并可选择支持非严格排序。路由表基于站点（由分叉端口共享），支持拒绝控制以实现虚拟交换机分区，并且必须允许全互联可达性，包括 U 型转向路径。

规范最后总结了必备的配置能力（加速器 ID、路由表、分叉模式、排序、丢弃模式控制），并提供了关于调试注入、延迟目标（例如，128 通道交换机 <200 ns）以及性能目标的建议，即在典型流量模式下保持 200 Gbps 的 FEC 后线路速率。




All switch ports shall connect to Accelerators and Switch-to-switch links shall not be supported. Each Request or Response shall be produced by a Source Accelerator, shall enter the Switch via an ingress Link, shall be relayed by the Switch to a single egress Link, where it shall consumed by a Destination Accelerator. All Response and Request relaying shall be unicast.

> 
所有交换机端口均应连接至加速器，且不应支持交换机间链路。每个请求或响应均应由源加速器产生，经由入口链路进入交换机，由交换机中继至单个出口链路，并在那里由目的加速器消费。所有响应和请求的中继均应为单播。




A UALink Switch is shall only responsible for the delivery of Requests and Responses between accelerators. A UALink Switch shall not be required to decode the contents of these Requests or Responses beyond that necessary to deliver the Requests and Reponses, norshall the Switch track any Request/Response state.

> 
UALink 交换机仅应负责在加速器之间传递请求和响应。UALink 交换机不得解码这些请求或响应的内容，除非解码是传递请求和响应所必需的，交换机也不得跟踪任何请求/响应状态。




Inbound DL flits arriving from the Source Accelerator shall be unpacked into TL flits, which shall then be further unpacked into individual Requests and Responses for forwarding between Ports. The Destination Accelerator ID field within each Request or Response shall index into a routing table to determine the egress Station and Port to which it must be forwarded. Once forwarded between ports, outbound Requests and Responses shall be packed into TL flits, which in turn shall be packed into DL flits and sent to the Destination Accelerator.

> 
从源加速器入站的 DL flit 须解包为 TL flit，再进一步将 TL flit 解包为独立的请求和响应，以便在端口之间进行转发。每个请求或响应中的“目标加速器 ID”字段应作为路由表的索引，以确定其必须转发到的出站站点和端口。完成跨端口转发后，出站请求和响应须打包为 TL flit，继而打包为 DL flit 并发送至目标加速器。




Each Request or Response shall be routed independently. Multiple Requests or Responses unpacked from the same inbound TL flit do not necessarily route to the same egress Station and Port and multiple Requests and Responses packed into the same outbound TL may have arrived from multiple ingress stations and ports.

> 
每个请求或响应须独立路由。从同一入站TL微片解包出的多个请求或响应不必路由至同一出站站点和端口，且打包至同一出站TL的多个请求和响应可能来自多个入站站点和端口。




The UALink Switch shall contain a complete implementation of the UALink DL and TL, similar to that found in an Accelerator. The nominal interface between TL and switch core shall be the UPLI Originator/Completer interface, although a Switch design may substitute any suitable vendor-defined interface. Any vendor-defined interface replacing UPLI shall mimic any UPLI mandated Originator or Completer behaviors in this Specification, including but not limited to UPLI Drop modes (see 3.1.3) and Port ID field manipulation (See 2.4).

> 
UALink 交换机应包含 UALink 数据链路层（DL）和事务层（TL）的完整实现，类似于加速器中的实现。TL 与交换核心之间的名义接口应为 UPLI 发起方/完成方接口，但交换机设计可以用任何合适的供应商定义接口替代。任何替代 UPLI 的供应商定义接口均应模仿本规范中 UPLI 所规定的发起方或完成方行为，包括但不限于 UPLI 丢弃模式（见 3.1.3）和端口 ID 字段操作（见 2.4）。




Flow control across an individual link between accelerator and switch, including the required independence of Requests vs Responses, shall be handled by the TL layer logic on each end of the link. Requests or Responses stalled by flow control shall wait within the outbound TL until they are eligible, at which time they may be included in a TL flit being packed and sent. Flow control for Requests or Responses being forwarded between Source and Destination Stations also respects the independence of Requests versus Responses, as does the UPLI interface between TL and switch core.

> 
加速器与交换机之间单条链路上的流控制，包括请求与响应所要求的独立性，应由链路两端的 TL 层逻辑处理。因流控制而阻塞的请求或响应应在出站 TL 内等待，直至符合条件，届时可将其纳入正在打包发送的 TL 微片中。在源站与目的站之间转发的请求或响应的流控制同样遵循请求与响应的独立性，TL 与交换机核心之间的 UPLI 接口亦如此。




### 10.2 Bifurcation support

Link bifurcation feature allows a station to split into multiple independent ports. Each ULS station has four lanes. Each station shall support independent configured as a single four-lane port, two two-lane ports, or four single-lane ports.

> 
链路分叉功能允许一个站点拆分成多个独立的端口。每个 ULS 站点拥有四条通道。每个站点应支持独立配置为单个四通道端口、两个双通道端口或四个单通道端口。




### 10.3 Lossless Request and Response delivery

Except in error cases which explicitly require Requests or Responses to be dropped (see Section 3.1.3), all Requests or Responses shall be relayed through the switch in a lossless manner. Link level Retry (LLR) shall be used to ensure delivery from source Accelerator to the switch ingress port, and from switch egress port to destination Accelerator. Flow control shall ensure that Requests and Responses are never dropped due to receive-buffer space limitations.

> 
除明确要求丢弃请求或响应的错误情况（参见第 3.1.3 节）外，所有请求或响应均应通过交换机以无损方式中继。应使用链路级重传（LLR），以确保从源加速器到交换机入口端口以及从交换机出口端口到目的加速器的交付。流控制应确保请求和响应绝不会因接收缓冲区空间限制而被丢弃。




### 10.4 Non-blocking architecture

Traffic between any one pair of Ports shall operate independently of traffic between any other pair of Ports. However, switch core architecture may share resources across ports within a station or even amongst multiple stations.

> 
任意一对端口之间的流量应独立于任何其他端口对之间的流量运行。然而，交换机核心架构可能会在一个站点内的端口间、甚至多个站点之间共享资源。




Ingress and egress traffic of a port shall operate independently of each other. Ingress Requests shall operate independently of ingress Responses, and egress Requests shall operate independently of egress Responses.

> 
一个端口的入口流量和出口流量应相互独立运行。入口请求应独立于入口响应运行，出口请求应独立于出口响应运行。




Stalled egress Requests or Responses to one port of a station may block same station egress Requests or Responses headed towards another port of the same station. Similarly, stalled ingress Requests or Responses from one port of a station may block like ingress Requests or Responses from another port of the same station. Under no circumstances shall stalled Requests be permitted to block Responses.

> 
发往同一站点某一端口的滞留出口请求或响应，可能会阻塞前往该站点另一端口的同类出口请求或响应。同样，来自同一站点某一端口的滞留入口请求或响应，也可能会阻塞来自该站点另一端口的同类入口请求或响应。在任何情况下，滞留的请求都不得被允许阻塞响应。




While such blocking is permitted, it should be minimized by switch designs to the extent reasonably possible. Non-blocking architecture is a crucial property of network switch design. Switches should incorporate sufficient buffering at the ingress and/or egress ports to prevent congestion and blocking.

> 
虽然这种阻塞是允许的，但交换机设计应在合理范围内尽量予以减少。非阻塞架构是网络交换机设计的关键特性。交换机应在入口和/或出口端口处配备充足的缓冲，以防止拥塞和阻塞。




### 10.5 Forward progress guarantee

All arbitration within the ULS shall be starvation-free, to avoid livelock or starvation cases. This shall include but is not limited to arbitration between ingress ports within a station, arbitration between ingress stations competing for access to an egress station, and arbitration between Requests and Responses for access to available TL flit sectors.

> 
ULS 内的所有仲裁均应为无饥饿的，以避免活锁或饥饿情况的发生。这应包括但不限于站内入口端口间的仲裁、竞争访问出口站的入口站间的仲裁，以及请求与响应间对可用 TL flit 扇区访问的仲裁。




Starvation-free arbitration may include simple round-robin, weighted and/or deficit round-robin, age-based oldest-first, etc.

> 
无饥饿仲裁可能包括简单轮询、加权和/或差额轮询、基于年龄的最老优先等。




### 10.6 Ordering and Virtual Channels

Each port-to-port path through ULS shall follow Request or Response delivery ordering and Virtual Channel handling as defined for UPLI (see 2.7.9). Support for Strict Ordering mode is mandatory, and support for non-Strict Ordering mode is optional. If non-Strict Ordering mode is supported, it shall be selected on a per-port basis, with identical settings being used on ingress and egress ports.

> 
通过 ULS 的每条端口到端口路径应遵循 UPLI 所定义的请求或响应交付排序及虚拟通道处理方式（参见 2.7.9）。必须支持严格排序（Strict Ordering）模式，非严格排序（non‑Strict Ordering）模式为可选支持。如果支持非严格排序模式，则应基于每个端口进行选择，并在入站和出站端口上使用相同的设置。




## 10.7Routing Table Structure

Routing decisions shall be made by looking up the 10-bit Destination Accelerator ID of inbound Requests and Responses in a route table. The number of entries in the table shall equal or exceed the maximum number of supported ports in the Switch with single-lane bifurcation. Each table entry shall contain the following information:

> 
根据 10 位目标加速器 ID 在路由表中查找入站请求和响应，以此做出路由决策。表中的条目数量应等于或超过交换机在单通道拆分下支持的最大端口数。每个表项应包含以下信息：




- For each port of a station, an indication of whether to deny routing matching Requests or Responses received by the port, versus allowing the matching Requests or Responses to be routed shall be provided. The Deny setting may be used to prevent routing Switches in a partitioned Physical Switch, and between Virtual Pods. Upon Switch reset, the Deny setting shall apply to all table entries.

> 
- 对于站点的每个端口，应提供一种指示，说明是拒绝为该端口接收到的匹配请求或响应进行路由，还是允许这些匹配的请求或响应进行路由。拒绝设置可用于阻止在已分区的物理交换机内以及虚拟Pod之间进行路由交换。交换机复位时，拒绝设置应应用于所有表条目。




- If not denied, an indication of the egress station and port to which the Request or Response shall be routed shall be provided. For example, in a switch with 64 stations, each

> 
- 若未被拒绝，应提供请求或响应将路由到的出口站及端口的指示。例如，在一个具有 64 个站的交换机中，每个




Ultra Accelerator Link Consortium Inc. (UALink) - UALink_200 Rev 1.0 Specification bifurcatable into at most four ports, each route table entry shall supply a 6-bit station number and a 2-bit port number.

> 
Ultra Accelerator Link Consortium Inc. (UALink) — UALink_200 Rev 1.0 规范，可分叉为最多四个端口，每个路由表条目应提供一个6位站号和一个2位端口号。




Where the table contains fewer than 1,024 entries, Requests or Responses whose Destination Accelerator ID value exceeds the capacity of the table shall be denied routing. Requests or Responses which are denied routing shall be silently dropped and shall be logged by the management controller.

> 
若路由表条目少于 1,024 项，则目标加速器 ID 值超出该表容量范围的请求或响应应被拒绝路由。遭到拒绝路由的请求或响应应被静默丢弃，并由管理控制器进行记录。




#### 10.7.1 Routing Table Instances

The switch shall contain a separate, independently programmable instance of the Routing Table for each station. Where a station contains multiple ports due to bifurcation, all ports within the station shall share the same routing table.

> 
交换机应为每个站配备一个独立、可分别编程的路由表实例。若某站因分叉而包含多个端口，则该站内的所有端口应共享同一路由表。




The independent programmability of different routing table instances, and of the independent deny controls for different ports within each bi-furcated station, allow a switch to be subdivided into multiple independent Virtual Switches , serving multiple Virtual Pods.

> 
不同路由表实例的独立可编程性，以及每个分叉站内不同端口的独立拒绝控制，允许将一个交换机细分为多个独立的虚拟交换机（Virtual Switch），以服务多个虚拟 Pod。




#### 10.7.2 Egress port reachability

All egress ports of all stations shall be reachable from any ingress port, with no arbitrary restrictions. This shall include allowing route-to-self (a.k.a. U-turn) cases, where the Requests and Responses ingress and egress via the same port.

> 
所有站点的所有出口端口均应可从任何入口端口到达，无任意限制。这应包括允许自路由（亦称 U 型转弯）的情况，即请求和响应通过同一端口进出。




## 10.8Configuration

Configurability options are left up to the implementation. The list below is an incomplete but a useful list:

> 
配置选项留待实现决定。以下列表虽不完整，但实用：




- Accelerator ID value associated with each port.

> 
- 与每个端口关联的加速器 ID 值。




- Routing table associated with each port, as described in Section (see 10.7)Bifurcation mode for each station.

> 
- 与每个端口关联的路由表，如第（见10.7）节所述每个站点的分叉模式。




- For each port, the Strict versus non-Strict ordering mode (see 10.6).

> 
- 对于每个端口，严格排序模式与非严格排序模式（参见10.6）。




- For each port, whether or not Authentication is enabled, which affects TL flit packing and unpacking.

> 
- 对于每个端口，是否启用身份验证，这会影响 TL flit 的打包和解包。




- Ability to determine which stations are in drop mode, and in the case of port-granular modes, ability to determine which ports are in drop mode.

> 
- 能够确定哪些站点处于丢弃模式，并且在端口粒度模式下，能够确定哪些端口处于丢弃模式。




- Ability to reset drop modes to resume normal operation, with station granular or port granular control, as appropriate.

> 
- 支持重置丢弃模式以恢复正常运行，并根据需要提供站粒度或端口粒度的控制。




### 10.9 UALink Switch Recommendations and Goals

The following subsections are recommendations and goals and are not meant to form UALink switch requirements but are meant to help guide implementors in areas of importance.

> 
以下小节为建议和目标，并非旨在构成 UALink 交换机要求，而是为了在重要领域为实施者提供指导。




#### 10.9.1 Debug

It is recommended that UALink switches support both transaction injection with and without errors towards switch core and towards the UALink. Other vendor defined debug support is allowed, but none is required under this section. UALink switches are recommended to support injection of key errors (i.e. link errors, link down, at least one silicon ECC error) to allow for platform/system level testing of RAS. The Injection, if supported, shall be disabled by default during mission mode and shall have a mechanism to allow enabling injection through a secure FW switch.

> 
建议 UALink 交换机支持向交换核心及 UALink 方向进行带错误和无错误的交易注入。允许其他由供应商定义的调试支持，但本节不作强制要求。建议 UALink 交换机支持关键错误（即链路错误、链路断开、至少一种芯片 ECC 错误）的注入，以便进行平台/系统级的 RAS 测试。若支持注入功能，则应在任务模式下默认禁用，并具备通过安全固件交换机制启用注入的机制。




## 10.9.2Latency goals

UALink Switches are recommended to target the following idle and unloaded pin-to-pin latency for a 64 byte Write Request on a 4-lane non bifurcated port with FEC enabled based on size of the switch:

> 
建议 UALink 交换机根据自身规模，在启用前向纠错（FEC）的 4 通道非分叉端口上，针对 64 字节写请求，达到以下空闲及空载条件下的引脚到引脚延迟目标：




- 128 lane switch: <200ns

> 
- 128 通道交换机：<200ns




- 256 lane switch: <250ns

> 
- 256 通道交换机：<250ns




- 512 lane switch: <300ns

> 
- 512 通道交换机：<300ns




#### 10.9.3 Performance goals

It is recommended that UALink Switch core should maintain a post-FEC line rate of 200Gbps. Switches should be capable of maintaining line rate for pairwise communication as well as concurrent communication across a small group (8-to-32) of accelerators. DL and TL protocol overhead will lower the effective line rate based on the traffic pattern and security protocols.

> 
建议 UALink 交换机核心应维持 200Gbps 的 FEC 后线速。交换机应能在成对通信以及跨小规模（8 至 32 个）加速器的并发通信中保持线速。DL 和 TL 协议开销将根据流量模式和安全协议降低有效线速。
