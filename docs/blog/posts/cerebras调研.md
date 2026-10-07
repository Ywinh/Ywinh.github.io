资料：
https://www.bilibili.com/video/BV1UZu46UErq/?spm_id_from=333.1007.top_right_bar_window_history.content.click&vd_source=7a39dbfc457222c1894595f42f7958fd


良率，如果有缺陷如何绕开，如何做到损失最小化
* 核心尽量小，如果缺陷在核心上，损失的面积比较少
* 有一个可配置片上网络，可以绕开这些缺陷

WSE的优势在推理，不在训练。训练可以并行，可以用GPU并行地计算。但是推理比如decode阶段需要依赖前一个token的结果，很大的瓶颈都在内存带宽上。
而WSE的内存和计算单元都在同一片晶圆上，内存带宽很高

SRAM替代DRAM

to address emerging extreme-scale ML models, the Cerebras architecture uses an **off-chip fabric** to extend to a cluster of CS-2 systems.

memory带宽和算力带宽

每个小core都有一个对应的、私有的SRAM，大小是48KB

hotchips26
> Cerebras的报告也非常有意思，因为它显示一家原本强调一片晶圆取代很多GPU的公司，也开始谈机架级架构。
> 
> CS-4是新的Nexus机架构平台的第一代产品，一个机架放三颗WSE，每颗WSE都在一个独立计算模块中，供电、液冷、I/O都模块化。Cerebras甚至把电源变换器放到距离晶圆约0.5毫米的位置，而传统GPU系统 往往需要经过更长的PCB电源路径。
> 
> 但真正看点是CS-6路线图。Cerebras一直靠巨量SRAM打存储墙，现在却宣布未来在晶圆级SRAM +计算上方堆叠3D DRAM。
> 
> 原因很简单：二维晶圆已经没有更大空间了，想继续提高存储容量必须往垂直方向走。这实际上与Day1 d-Matrix 3D DRAM的逻辑高度一致，下一轮存储扩展很可能不仅是HBM更快，而是存储与计算的垂直集成。

介绍下memory hierarchy
|层次|容量/结构|典型内容|访问方式|
|---|--:|---|---|
|GPR|每核 16 个|标量、地址、状态|指令直接访问|
|软件管理 cache|每核 256 B|MatMul accumulator 等高频状态|软件显式安排|
|本地 SRAM|每核 48 KB，8 个 bank|激活、代码、FIFO、循环缓冲区|仅本核直接 load/store|
|其他 PE 的 SRAM|全晶圆聚合约 40 GB|分布式激活和中间结果|必须通过 fabric 显式发送|
|MemoryX DRAM/Flash|晶圆外|权重、梯度、optimizer state|权重流入、梯度流出|
|Dataset server|系统外|输入训练数据|经系统 I/O 输入|


编程单位不是thread，而是tensor
每个core有44个DSR（Data Structure Register），张量描述符，gpu也有类似的tma描述符
* - 起始地址；
- 长度；
- shape；
- element size；
- 数据来自本地 SRAM 还是 fabric；
- FIFO 或 circular buffer 状态

稀疏计算下更有优势，少数据复用，因此数据搬运需要快
发送端可以过滤零值：

- 非零权重才生成 packet；
- packet 到达才触发 FMAC；
- 零值没有 packet，因此没有相应计算。

所以这里不是“执行乘零但优化了一点功耗”，而是真正的：

```
没有数据到达 → 没有任务触发 → 没有这次乘加
```

这支持非结构化稀疏，而不一定要求固定块或 2:4 模式。

但要注意：稀疏加速还依赖工作是否均衡。如果非零元素集中在少数 PE 或少数网络链路，即使总非零数很少，也可能出现热点。论文强调 core 级动态调度，但没有详细讨论全晶圆的稀疏负载均衡问题。


dataflow不仅仅是数据，还有控制信号，会选择要执行什么任务

每个core最多支持8 个同时存在的 tensor operation，论文称为 microthreads。8 个独立的张量执行上下文。调度器每个cycle选择一个可用的上下文进行执行。

microthreads之间共享本core的寄存器和SRAM

**通信**：
二维mesh，每个core都有一个五端口router

## 静态路由和 color

路由不是每个 packet 动态查路由表，而是在执行前静态配置。每个 core 有 24 个逻辑 route，称为 colors。

软件需要理解两个事实：

1. color 是逻辑通信通道。
    
    不同 color 可以对应权重、激活、控制命令、partial sum 等不同流。
    
2. 24 个 color 不等于 24 倍物理带宽。
    
    它们最终会在相同物理链路上时分复用。如果多个 color 同时大量经过同一链路，仍会发生拥塞和 backpressure。
    

因此，底层程序/编译器要完成的工作很像片上分布式系统：

- place：任务放在哪些 PE；
- route：数据经过哪些 PE；
- buffer：每条流需要多大队列；
- schedule：什么时候广播、什么时候归约；
- flow control：生产速度与消费速度是否匹配。

广播和multicast