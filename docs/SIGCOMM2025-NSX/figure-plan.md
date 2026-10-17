# NSX figure plan

- Figure 1 → images/fig01.jpg：GPU架构与执行模型；放问题定义，解释SIMT与访存约束，明确不是动机性能实测。
- Figure 2 → 原文解析为伪代码：放方案MET小节，用代码块按原逻辑转写，并解释安全边界与传播承诺。
- Figure 3 → images/fig02.jpg：多GPU扩展性；放实验，解释红叉为OOM、4卡已能运行524k、8卡小于2秒。
- Figure 4 → images/fig03.jpg：CPU对照；放实验，解释对数纵轴、红叉为超时、基本转发公平性范围。
- Figure 5 → images/fig04.jpg：CUDA graphs与flex queues消融；放实验，标记读图近似值、解释相反趋势。
- Table 1 → Markdown表格完整转写：放真实集群验证，明确KB单位，并计算Leaf Down中位数相对误差。

已逐张视觉检查全部四张图；本地资产编号与原文图号不同，以原文图号作图注。
