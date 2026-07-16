---
date: 2026-06-22
categories:
  - gem5
tags:
  - Writing
---

## 目录

1. [术语解释](#terms)
2. [背景](#background)
3. [原理](#principles)
4. [build & run](#build-run)
   1. [build](#build)
   2. [run](#run)
5. [演进、开发历程、替换步骤](#evolution)
6. [具体](#details)
   1. [bridge warpper](#bridge-wrapper)
      1. [背景](#bridge-wrapper-background)
      2. [核心功能](#bridge-wrapper-core)
   2. [FIFO建模](#fifo-modeling)
      1. [背景](#fifo-background)
   3. [decoder设计](#decoder-design)
   4. [Sc main的骨架，接入sc sim top](#sc-main)
   5. [host修改/ddr接入](#host-ddr)
      1. [背景](#host-ddr-background)
      2. [核心实现](#host-ddr-implementation)
   6. [多CCU](#multi-ccu)
7. [不足之处](#limitations)

<a id="terms"></a>
术语解释：

- sc：systemc
- emu/calcore：都指 nbu model

<a id="background"></a>
## 背景
补充架构图

**为什么要用920替换906？**

对于整体的 emu 架构来说，可分为 scalar core 和 npu 模块。scalar core 主要做的事情是进行标量指令的计算，以及分发 nscl（non scalar：非标量指令）给 npu。

原来的负载主要是 CNN，即使有大模型（Transformer），参数量也比较小，几百兆几B。后来随着超大大模型的出现，原本的906 core计算能力不够了，跟不上npu的能力，出现npu等待906这一状况。

两种负载的大致区别：

- cnn：输入一张图 -> 输出分类 / 检测结果
  - 不管输入是什么，都会经过相同的处理层（固定 shape），其计算量可以提前算出来，可以在运行前大致知道 latency。
- transformer：输入 prompt -> prefill -> 逐 token decode -> 直到生成结束
  - 虽然不同输入也会经过同样的架构，但是计算量不一样，会随着输入 token 长度变化而变化，并且输出 token 也不固定。

一个关键区别是：CNN对于不同shape，最终到硬件上都会变成一个统一的shape（不够的地方padding），这给了很多在运行之前就能确定的信息，比如把地址提前在编译阶段就算好，这样scalar core 就不用再算地址了，计算压力也比较小。但是Transformer，shape不固定，这意味着你的地址需要scalar core在运行时动态计算，模型大的时候 scalar core 的计算压力比以前cnn/小模型时期重很多。需要处理变长输入、KV cache、token-by-token decode、动态 batch、mask、采样等更多控制逻辑。这是906需要用920替换的一个主要原因。

经过内部 prof 发现，906 作为标量核时，会出现 NPU 算完了，但下一条 NSCL 指令还没发出来，906 一直转，npu 却经常歇着。这说明 906 的计算、控制能力已经不够了，需要一颗更加强大的标量核心，所以需要替换为 920。

C906 是顺序单发射小核，控制能力够，但遇到复杂 runtime、频繁分支、cache miss、地址计算、队列管理时，容易成为瓶颈。C920 是乱序多发射大核，能更好隐藏访存延迟、提高 IPC，更适合跑复杂软件栈。

**ks2 的 nscl 指令形式变了**

ks2 的指令不再是直接从 ddr 里面获取，而是需要用 4 个标准riscv sd（store double word）来拼接为 256 bits

ks1 的指令集其实是混合的，不仅有标准riscv指令，还有nscl指令集（通过riscv自定义扩展指令实现），虽然nscl指令不会在scalar core里面执行，但是scalar core认识它

程序指令平铺 vs 正常有if else的程序：
* 指令平铺是指把所有if else都展开，整个程序只需要顺着pc一路+1，+1，执行到底。这样的好处是程序少了分支指令，不用判断了，执行的总指令条数相比有分支指令的会少一些，但是程序很大（pc一直递增，不会重用），局部性较差，cache基本不命中
* 正常有 if else的程序本身比较小，很多重复的地方都用跳转指令来重用，局部性比平铺的好。但是常常需要实时计算，判断是否跳转等。计算量比较大，但是内存压力小，cache命中多。

ks2 指令集就只有标准的riscv指令集+玄铁扩展指令集，所有的nscl指令不再显式出现，而是通过store来做。scalar core计算更加密集


<a id="principles"></a>
## 原理
gem5 会对访存指令计算出的 addr 进行判断。如果某个 addr 在 bridge 指定的范围内，则会往外部发。在当前的实现中，把这个范围设置为 ALL，也就是所有的访存请求都会往外发，gem5 只负责计算、执行标量指令。

`gem5 -> bridge -> bridge warpper -> nbu model 大 bus`。这个 `bridge warpper` 模块的作用其实类似一个小 bus，但是由于功能很简单，没有做成 bus：

- 一个 slave 接收 bridge，两个 master（fifo 和 non-fifo）接到 ddr/sram 等大 bus，由大 bus 路由。
- 由于大 bus 原本这个地方的 socket 被 scalar core 占用，需要断开。

**简单介绍gem5接入**
**原理概述**

gem5 能和 SystemC 一起仿真，本质原因是：**两者都是离散事件驱动模拟器**。

- gem5 用 `EventQueue` 管理事件，时间单位是 `Tick`
- SystemC 用内核调度 `SC_THREAD`、`SC_METHOD` 和 `sc_event`，时间单位是 `sc_time`

它们都遵循同一个基本思想：
**在某个仿真时间点处理事件，再推进到下一个事件时间点。**

所以，gem5 与 SystemC 协同仿真的核心，不是“让两边都运行”，而是解决下面三件事：

1. **谁来做总调度器**
2. **两边的时间如何保持一致**
3. **gem5 的请求/响应如何转换成 SystemC/TLM 事务**（这个简单，查看源码 gem5totlmbridge 例子即可）

gem5 提供了两种路径，[在此插入两种路径具体介绍与讨论]()。
我们选用将 gem5 作为一个 lib 供 sc 使用，其基本原理是：

- sc 作为总调度器，gem5 作为一个 systemc 程序嵌入，gem5 的 eventq 属于 sc kernal 管辖，由 gem5 自己调度、push 和 pop，只是这个 eventq 数据结构属于 sc。

**ks2与ks1仿真器的区别：**

- 整个 scalar core 弃用，换为 gem5，原有的与其他模块的连接断开。
- 增加了 inst fifo：【todo文档】
- decode 模式变了：【todo文档】
- host 没有 csr 启动，调用 gem5 启动函数传入 pc 来手动启动。

**架构图**
todo

<a id="build-run"></a>
## build & run
<a id="build"></a>
### build
查看build_new.md

<a id="run"></a>
### run
首先需要 gem5 跑一遍生成 `config.ini`，然后使用 calcore 接收该 `config.ini`。

```
cd gem5/util/tlm
../../build/RISCV/gem5.opt ../../configs/c920/run.py [options] --binary /path/to/elf

cd calemu/nbu_model
./bin/calcore [calcore options] /path/to/gem5_config_ini [gem5 options]
```

此处涉及到参数解析。calcore 有一套参数解析，gem5 也有一套参数解析，以 `config.ini` 作为分界线，分别传入不同的 parser。

<a id="evolution"></a>
## 演进、开发历程、替换步骤
1. 建模 sc fifo，然后测试，此时还是在 gem5 环境。
2. 将 gem5、sc fifo 与 calcore 一起编译，连接 bridge 到 nbu core，需要修改 cmake。
3. 修改 decode，从 64 bits 改到 256 bits。
4. 进行初步集成并测试，ddr/sram 采用 gem5 自带的，只有写到 fifo 的访存请求会通过 bridge 路由出去。并且此时的 nbu host 也没有启动（注释掉了 host 的 sc thread），load elf、启动 cpu 都是 gem5 自己干的。
5. 将 gem5 的 load elf、启动 cpu 转移到 nbu host，同时 gem5 把所有的访存请求都路由到 bridge，再由 bridge 发往下一 systemc 模块。

<a id="details"></a>
## 具体
以下说明一些具体模块的实现、连接关系，以及开发遇到的一些 bug。

<a id="bridge-wrapper"></a>
### bridge warpper
<a id="bridge-wrapper-background"></a>
#### 背景
在最开始的步骤中，gem5 是这样集成的：ddr 使用 gem5，只把写到 fifo 的内存请求通过 bridge 路由到 sc，此时 bridge 的出口只有一条，就是连接到 inst fifo 上面，不用做区分。

到了后续，需要将 ddr 也从 gem5 转向 emu 内部的 ddr，也就是接入 sc ddr。此时对于 gem5 的内存请求来说，就有区别了。按照地址范围，访存可能分为 sram/ddr/fifo，其中 sram/ddr 的区分在内部 mem bus，fifo 还是依旧。

此时 bridge 出来，其实应该有两条路径。如果发现地址位于 fifo 的范围，则依旧传给 fifo；除此之外的所有 addr 都应该交给内部的 mem bus 处理。那么此处其实需要一个小 bus，或者说小模块，来区分 addr 位于 fifo 还是 nonFifo，进行不同的连接与 tlm 调用。这个模块就是 **bridge warpper**。

在实际实现时还会有一个问题：bridge 发过来的包都是 tlm2.0 协议，fifo 接收也是 tlm2.0 协议，但是 mem bus 却是 tlm1.0（mem bus 向外通过 Initiator 连接到 Initiator `nbu_core::DataIsocket`，此处又是 tlm2.0），nonFifo 如何连接出去呢？在此处讨论过两种方案：

1. 方案一：bridge warpper 的 nonFifo 出口直接连接到 nbu core 的 `DataIsocket`，都是 tlm2.0，也就不用转换了。但是要注意这个 socket 得是新增的一个，同时也得在外部大 bus 增加一个 master。因为如果使用原本的 `dataIsocket`，就得断开 mem bus 的连接，而 dm0 和 dm2 又是通过这个 mem bus 连接到外部大 bus 的，因此断开会造成问题。
2. 方案二：nonFifo 出口连接到 mem bus 上面，在 mem bus 上增加一个 master 口和一个用于 tlm1.0 通信的 sc fifo。此改动兼容性较好，但是略微复杂。因为 mem bus 的接口是 tlm1.0，在此需要先把 tlm2.0 转为 1.0，把包发出去。等到包回来时，如果是 read，把读到的数据写回到 2.0 的 payload 中，再 return；如果是 write，则设置 response 状态再 return。

todo 画个图

<a id="bridge-wrapper-core"></a>
#### 核心功能
所处位置：`bridge -> bridge warpper -> fifo/mem bus`

入口：接收来自 gem5 bridge 的 tlm2.0 blocking transport 请求。当前只实现了 blocking。

地址区分：check payload 的 addr，如果是 fifo，直接调用 fifo 的 tlm2.0 transport 接口。如果是 nonFifo，则调用 `putTrans`。该函数首先把 b transport 的 tlm2.0 packet 转换为 1.0，然后通过 tlm1.0 port put 出去。在此函数中，read 和 write 的处理都是一样的，都是把 payload 的 data copy 到 transnoc 中。接着在此处调用 get，阻塞住等待回复。

bridge warpper put 后，mem bus 中的 port 接收，然后往外发请求。等到 response 回来时，mem bus 将 response put，bridge warpper 解除阻塞。根据 read 和 write 的不同处理，完善 tlm2.0 所需信息，然后 return。

这样一次请求就完成了，路径是 `bridgewarpper -> mem bus -> external -> mem bus -> bridge warpper -> gem5 bridge...`

<a id="fifo-modeling"></a>
### FIFO建模

<a id="fifo-background"></a>
#### 背景
ks2 中，nscl 指令不再是从 ddr 中取出，elf 会变为全是 riscv+玄铁指令，不再有 nscl 指令。所有的 ks2 指令会有 4 个 sd 来拼接，每个 sd 写 64 bits，一共 `64*4 bits`，写到 inst fifo 中，该模块设计为 4 个 fifo，每个 fifo 深度是 4，entry 大小为 64 bits。一旦拼接好 256 bits 之后，就进行 decode，然后发给 inst queue。

【插入文档】

FIFO 出口的 256 bits 兼容已有的高级数据结构，发给 inst_queue。
fifo 实例化放在 ccu 内部，每个 ccu 都得有一个 fifo，而不是在 sim top。
放在哪里实例化呢？它们需要解析 argc 和 argv。

作为参数传递？

- fifo 依赖 transactor，transactor 在哪里创建都没关系。
- simcontro 需要解析 argc 和 argv。
- 上述两个 obj 的唯一性？

<a id="decoder-design"></a>
### decoder设计

- Decoder bin 的输入从 32/64 统一改为 256 拼接的。
- **256decode，提取寄存器值，没有寄存器index信息，不提取寄存器index，直接填写0？**

输入的 256 需要新建一个 256 bits 的数据结构。

输出还是 instrptr，但是需要 makeinstr 时再加一个 val，这个 val 从 256 提取。

其他的字段也需要重新解析。

scalar 的 256 decode 不需要做，gem5 自己完成即可。

<a id="sc-main"></a>
### Sc main的骨架，接入sc sim top

```C++
#include <systemc>
#include <tlm>

#include "cli_parser.hh"
#include "report_handler.hh"
#include "sim_control.hh"
#include "stats.hh"

int
sc_main(int argc, char **argv)
{
    CliParser parser;
    parser.parse(argc, argv);

    sc_core::sc_report_handler::set_handler(reportHandler);

    Gem5SystemC::Gem5SimControl sim_control("gem5",
                                           parser.getConfigFile(),
                                           parser.getSimulationEnd(),
                                           parser.getDebugFlags());


    Gem5SystemC::Gem5SlaveTransactor transactor("transactor", "transactor");
   
    instantiate..
    socket bind..
    transactor.sim_control.bind(sim_control);

    SC_REPORT_INFO("sc_main", "Start of Simulation");

    sc_core::sc_start();

    SC_REPORT_INFO("sc_main", "End of Simulation");

    CxxConfig::statsDump();

    return EXIT_SUCCESS;
}

```

简单说明：
1. 这个parser是解析gem5参数的，比如-d Exec等，但是由于运行calcore时，前面是calcore参数解析，以config.ini为分界线，之后是gem5参数，需要对argc，argv进行额外处理
2. simcontrol是一个gem5的顶层控制模块，他会解析config，进行所有gem5 obj的初始化等
3. 端口bind：每个transactor作为slave需要bind到sim control上面，然后transactor作为master连接到nbu这端的inst fifo‘
4. statsdump是在gem5仿真结束后，dump出统计信息，生成stats.txt

<a id="host-ddr"></a>
### host修改/ddr接入

<a id="host-ddr-background"></a>
#### 背景
在最初的集成中，先是 gem5 来 load elf，自己启动 cpu 相关线程，然后取指执行，发给 fifo，驱动 nscl core，此时 host 完全没有启动。

完全集成好的话，需要遵循 host load elf。其实也讨论过 gem5 继续 load elf，只是写到 sc 的 ddr 内，但是考虑到会有一些比较特殊的操作，比如不根据 elf 的 entry point 加载，而是加载到其他地方，这是 gem5 做不到的。为了满足这种要求，还是从 host 来 load elf。原本的 host elf 是 32 bits，而 920 的 elf 是 64 bits，需要修改 elf load 与解析的数据结构。

除此之外还需要支持 host 启动 gem5。原本的 scalar core 启动是由 csr ctrl 来控制，host 会写入 start bit，scalar core 识别到该 start bit 之后启动相关 sc thread。而 gem5 是由一个 `resetThread` 和 `activate` 来控制 cpu ready 启动的。

- `resetThread` 会设置线程的初始化状态、机器态寄存器等，最重要的是根据 gem5 配置的 workload 来设置 pc。
- `activate` 则是把 thread 状态设置为 ready，不然 `sc_start` 之后不会动。

<a id="host-ddr-implementation"></a>
#### 核心实现

1. load elf修改：
  - 将 nbu model 的 load bin 从 32 位解析修改为 64 位解析。
  - 后门加载：需要在 `sc_start` 之前，programinit，所以需要后门加载，因为此时 ddr/sram 还没有创建。
  - 为 gem5 增加 skip load elf 选项，需要在配置文件中修改，默认是跳过。该选项打开时不会触发把 elf 写入 ddr。

2. host start：
  - gem5 增加了一个 host start 选项，开启之后不会主动 activate thread。然后暴露出一个接口供 calcore host 调用。

**需要确认gem5 config host start和skip** **elf****都是true**
问题：如果 `tc->activate` 放在 `sc_start` 之后，gem5 其实根本没动；如果 `tc->activate` 放在 `sc_start` 之前，会发现 gem5 发过来了 fetch。

为什么？
start from host 是谁做的？每个 ccu 一个？每个system各需要一个start from host

ddr：
需要 host 加一个后门加载，在 nbu mem access backdoor 以不消耗时间的动作加载 elf，替换掉原有 programinit 里面的 memblockrwddr。

保证 fetch 过来之前，这个 memory 已经写入 elf。

需要 64 位的 load bin，以及 program init 里相关 32 bit 的数据结构和掩码都改为 64 位 1。

**确认load bin正确，**

<a id="multi-ccu"></a>
### 多CCU
由于每个chip里面实际上有两个ccu，这两个ccu的地址空间是独立的。那么在gem5中也需要类似的配置，如果是两个ccu，则需要两颗独立的gem5 CPU，在后期他们可能运行不同的elf。在gem5具体实现中，需要多个board

从上到下结构是
root
	-board
		-processor
		- cache-hierarchy
		- memory

因此，我们需要在root下面创建两个board，每个board地址空间独立，都有一个transactor向外转发访存请求

* host start：每个ccu都需要来启动自己的gem5，可以利用gem5接口来找到每个ccu各自的board，然后调用start host函数
* elf指定：elf在board.workload中指定，如果后期需要多个不同elf，需要在gem5中配置
* transactor bind：

<a id="limitations"></a>
## 不足之处

- delay 没有建模。
- wmb【TODO】
- 原有的仿真器独有指令 gem5 暂未支持，后期可以考虑在 gem5 实现识别这些指令。

Gem5:

- cacheable 和 uncacheable
- Gem5 是没有 store buffer 的，store queue 充当了这一角色。
- uncacheable 会走 cache 层次，但是只会加上 tag lookup + read/write resp 延迟。
