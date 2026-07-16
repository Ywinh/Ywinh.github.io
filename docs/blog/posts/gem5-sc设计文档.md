---
date: 2026-04-06
categories:
  - LLM
tags:
  - Writing
---

# gem5建模c920

## config对齐rtl与920手册


## 性能测试
测试方式主要是和硬件跑相同的elf，硬件需要配置玄铁相关csr来enable 920的特性，保证基本特征如prefetch等对齐
然后硬件需要配置，以dump性能计数器。与gem5 dump出的stats.txt进行对比

参数太多了，主要对比什么参数？



# 集成的两条路
## ext systemc


## gem5 systemc



# 集成编译
在nbu_model/cmake/下增加了gem5.cmake

需要enable cxx_std_17，因为gem5编译时依赖这个选项，不然运行时会报错

有几个target
1. gem5_tlm


由于原本nbu model build时依赖的lib fmt、yaml-cpp、systmec在旧的g++工具链编译的，而worker22的工具链比较新，导致link时出现abi不兼容，gem5还只能在这个worker编译



因此从外网github导入了对应版本的包，在worker22重新编译，放到了/home/common_share/SMG/gem5_newlib下面

需要更改path.cmake改成对应的path才能编译成功

# blocking inst fifo设计
**背景**：ks2硬件提出的新方案，所有的nscl指令会通过四个sd来拼接，64bits\*4 = 256，写到inst fifo结构，该fifo负责暂存各个64bits数据以及整合发送256bits拼接指令

**术语解释：**
* payload：指tlm2.0的标准包数据结构，tlm_generic_payload


**对外接口**
1. 一个slave tlm2.0接口，由gem5 bridge连接
2. 一个master tlm1.0接口，连接到instr queue

**数据结构设计**
* 内部核心结构是四个fifo，分别是fifo 0～3，每个fifo深度为4，每个entry是64bits
* 其余比较重要的是一个sc event，当4个fifo都有效时触发pop 256bits

fifo构造时需要一个base addr和size，目前和硬件对齐在构造时写死，base addr是0x0a082000，size是16\*8 = 128bytes
当前设计四个fifo，每个fifo深度为4，每个entry是64bits


**fifo什么时候加入entry**，通过
gem5的内存请求  -> bridge -> 转换为tlm2.0 nb请求 -> blocking请求 -> instrfifo::b_transport_bus
* gem5有三种仿真mode（timing、atomic、blocking）由配置文件指定，一般都用timing模式建模比较精确，但是timing模式对应的其实是tlm2.0的nb transport，我只实现了b transport，是如何正确调用的？
	* systemc帮忙做的，如果没有实现nb，会把nb的语义转为blocking，从而正确调用

当请求到来时行为
1. assert是write请求，fifo只能被gem5写
2. 从payload提取addr，check addr在规定范围之内，以及data length为8 bytes，若失败assert 0
3. 从payload copy data
4. 根据addr来确定往哪个fifo push（地址映射，规则和robin的图一致）
5. push时采用sc fifo的阻塞式write，如果当前fifo满了，是会在这里一直卡住，等到有空闲的entry
6. 成功push之后，设置response ok
7. 如果当前push到fifo之后，4个fifo能够拼接一个256bits，则直接notify pop事件，pop 256


**fifo什么时候出entry**
* pop thread一直在wait一个事件，该事件由push fifo在4个fifo都有有效entry时触发
* 采用fifo.read，阻塞式读出消耗entry
* 定义一个数据结构接收：内部是一个uint64数组，长度为4，接受4个fifo读出的值，然后调用decode模块生成fullinstptr，通过tlm1.0 port发送给instr queue


TODO：延时设计

讨论过两种入fifo的设计
Sc fifo是否是有四个就触发？还是说要第四个进来时才从reg写到fifo
即0 1 2 3可触发，0 2 3触发时发现不对不够，后来了个1，如何处理？
后续看rtl check

目前的设计是第一种有4个就出发，提前写入fifo，没有经过reg

# 256bits decode

背景：之前nbu一直都是64bits decode，即直接接收指令本身，然后需要解码出一个高级数据结构fullinstr供instr queue驱动。ks2不再接收指令本身，而是利用编译器，将每条原本64bits的nscl转为4条sd，4\*64=256bits，因此decoder需要解析256bits指令

区别：
* 64 bits指令本身，其中寄存器field存的是寄存器index，而不是具体值，因此一般的decode流程是

根据pc从mem中读出64bits指令，然后decode生成fullinstr，只不过由于指令中只有寄存器index，没有具体值，此时各个reg的val还是0
需要额外通过一个fill register步骤从mem中读出值，填写到生成的fullinstr数据结构中，此时才算构造好可以传给下一模块处理

现在256bits，其实自带了寄存器值，不用一个额外的步骤从mem中读取寄存器值，而是直接从256里面读，然后填充fullinstr，传给下一模块

decode256的实现
该函数的输入为拼接好的256bits，uint64_t array\[4\]
* 借鉴硬件的实现，有一个helper函数，提取256bits中每个寄存器的start，end bits
* 然后有一个extract 256函数，该函数接收start，end，然后从256数据结构中提取对应值。需要解决跨64bits的获取值问题，因为虽然是256，但是是以64为单位，从数组中索引的，如果是跨64，就需要索引两个entry（我们把数组看作是4个entry）
* 读取出值之后，在makeinstr时，把寄存器值也填进去，index现在默认0，因为256bits中是没有存储index信息的，但是可以知道name，如果一定要index，可以通过某种方式索引name得到index，比如一个预先填好的固定的map

简单来说，256和64的不同之处，就在于没有index了，需要通过顺序读取某条指令的所有寄存器，记录start和end bit

最前面提取低12位，低12位都是一样的，然后逐个提取reg 位域

问题：格式可能不太优雅，和之前的没有对齐

遇到一个bug
rtl和页面上decode的起始bit不一样，导致decode识别错误，修改start bit后解决

# transactor接入，单ccu/双ccu


# ddr和fifo都接入
此时其实有两种设计，一种是gem5只走一个transactor，每个transactor其实是要对应到一个addr列表，这样gem5才能够知道如果访问到了这里的地址时，会发送到transactor而不是内部的某个mem，如果只采用一个transactor，那么也就只有一个addr列表，那么这里需要两串地址，\[\[fifo\]，\[ddr\]。gem5做的只是识别到对指定addr的load/store时转发到transactor，然后transactor通过一个socket连接到sc bus，由sc bus来进行具体地址的区分和转发，什么地方转发到fifo，什么地址转发到ddr？

第二种设计是采用两个transactor，分别对应fifo和ddr，这样的好处是不用建一级bus，集成比较快速和方便


load elf方案：
方案1. gem5来load elf到ddr
可能需要tlm2.0的direct mem ptr来初始化ddr，需要转到transactor上

方案2.sc来load elf到ddr
方案2改动会少一点，sc暴露一个接口让gem5拿到pc就好，对以后的host也方便，兼容现有的一些别的功能

可以不load elf，但是可以解析elf，这样不需要sc通过某种方式传pc
需要确认gem5的load elf和nbu 的load elf功能是否有区别

# debug手段



# 遗留的问题

1. gem5的uncacheable目前会通过cache层次，会加上tag lookup+read/write resp延迟，可以考虑更改，如果影响大的话
