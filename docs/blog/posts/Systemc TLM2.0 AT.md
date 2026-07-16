---
date: 2026-04-15
categories:
  - HW
tags:
  - Writing
---

参考：
1. https://www.cnblogs.com/sasasatori/p/19077607
2. https://ywinh.github.io/show-gem5-sc/21

tlm_generic_payload
```c++
private:

/* --------------------------------------------------------------------- */

/* Generic Payload attributes: */

/* --------------------------------------------------------------------- */

/* - m_command : Type of transaction. Three values supported: */

/* - TLM_WRITE_COMMAND */

/* - TLM_READ_COMMAND */

/* - TLM_IGNORE_COMMAND */

/* - m_address : Transaction base address (byte-addressing). */

/* - m_data : When m_command = TLM_WRITE_COMMAND contains a */

/* pointer to the data to be written in the target.*/

/* When m_command = TLM_READ_COMMAND contains a */

/* pointer where to copy the data read from the */

/* target. */

/* - m_length : Total number of bytes of the transaction. */

/* - m_response_status : This attribute indicates whether an error has */

/* occurred during the transaction. */

/* Values supported are: */

/* - TLM_OK_RESP */

/* - TLM_INCOMPLETE_RESP */

/* - TLM_GENERIC_ERROR_RESP */

/* - TLM_ADDRESS_ERROR_RESP */

/* - TLM_COMMAND_ERROR_RESP */

/* - TLM_BURST_ERROR_RESP */

/* - TLM_BYTE_ENABLE_ERROR_RESP */

/* */

/* - m_byte_enable : It can be used to create burst transfers where */

/* the address increment between each beat is greater */

/* than the word length of each beat, or to place */

/* words in selected byte lanes of a bus. */

/* - m_byte_enable_length : For a read or a write command, the target */

/* interpret the byte enable length attribute as the */

/* number of elements in the bytes enable array. */

/* - m_streaming_width : */

/* --------------------------------------------------------------------- */

  

sc_dt::uint64 m_address;

tlm_command m_command;

unsigned char *m_data;

unsigned int m_length;

tlm_response_status m_response_status;

bool m_dmi;

unsigned char *m_byte_enable;

unsigned int m_byte_enable_length;

unsigned int m_streaming_width;

tlm_gp_option m_gp_option;
```

阻塞：需要b_transport
* 注册
* bind：initiator绑定target


非阻塞：需要nb_transport
* 注册 通过 register_nb_transport_\[fw|bw\] 在对应的port
	* initiator注册bw
	* target注册fw
* bind：
	```c++
	m_init.m_initiator_port.bind(m_target.m_target_port);
	```
四个phase：

fw和bw都是处理收到的phase
fw：
* BeginREQ
* END_RESP

bw函数内：每个case会调用一个回调函数
* END_REQ
* BEGIN_RESP

begin放入，即notify
而end消费，即get

peq_with_get：特殊的工具，用于管理信息传输过程中的时序（使用了tlm提供的peq_with_get类）

![](./pic/Systemc%20TLM2.0%20AT-1776242649100.webp)

## gem5 load elf
gem5的memory还有一类是system memory，用来load elf等
根因不是汇编指令本身，而是你的自定义 SystemC memory 路径没有给 SE mode 提供一个可用的 AbstractMemory/physmem。

关键点有两个：

1. System.memories 只会收集 AbstractMemory 子对象，src/sim/System.py (line 64) 明确写的是 VectorParam.AbstractMemory(Self.all, ...)。  
    但你这条链路里的两个对象：
    
    - C920TlmPrintMem.py (line 7)
    - TlmBridge.py (line 35) 里的 Gem5ToTlmBridgeBase
    
    都是 SystemC_ScModule，不是 AbstractMemory。所以它们不会进入 System.memories。
    
2. System 在构造时会用 p.memories 去建 physmem，src/sim/system.cc (line 167)。然后 SEWorkload::setSystem() 会从 sys->getPhysMem().getConfAddrRanges() 取地址范围，填充 memPools，src/sim/se_workload.cc (line 43)。  
    如果 System.memories 是空的，那 getConfAddrRanges() 就是空，memPools.populate() 后 pools 仍然是空的。
    

后面一到装载 ELF 的阶段就会炸：

- Process::initState() 会先用 SETranslatingPortProxy(Always) 把 ELF 段写进目标内存，src/sim/process.cc (line 300)。
- MemoryImage::write() 会对每个 segment 做 writeBlob/memsetBlob，src/base/loader/memory_image.cc (line 38)。
- 一旦目标页还没映射，SETranslatingPortProxy::fixupRange() 就会调用 process->allocateMem()，src/mem/se_translating_port_proxy.cc (line 55)。
- allocateMem() 再去调用 seWorkload->allocPhysPages()，src/sim/process.cc (line 317)。
- 最后 MemPools::allocPhysPages() 直接做 pools[pool_id].allocate(npages)，src/sim/mem_pool.cc (line 162)，但这时 pools 是空的，所以是越界访问，结果就是你看到的 segfault。

所以这个崩溃发生在“程序还没真正开始执行 _start/main 之前”，是装载器在给进程分配物理页时死掉的。  
你那个 systemc_mem.out 的 readelf 也能佐证这点：它有一个 RW LOAD segment，MemSiz 是 0x2002810，装载器会在 tick 0 之前就去为它分配并清零很多页


gem5会调用functional request来load image，路径如下，接到tlm时，最终会调用到transport_dbg
你要的完整路径，在当前 SE + configs/c920 下是这条：

  1. Process::initState() 创建 SETranslatingPortProxy(tc, Always)，然后调用 image.write(*initVirtMem)
     见 src/sim/process.cc:289
  2. MemoryImage::write() 遍历 segment，调用 proxy.writeBlob(seg.base, seg.data, seg.size)
     见 src/base/loader/memory_image.cc:38
  3. proxy 的动态类型是 SETranslatingPortProxy，所以 PortProxy::writeBlob() 内部调用的虚函数 tryWriteBlob()，实际落到 TranslatingPortProxy::tryWriteBlob()
     见 src/mem/port_proxy.hh:192 和 src/mem/translating_port_proxy.cc:101
  4. TranslatingPortProxy::tryWriteBlob() 先做 translateFunctional() 把虚拟地址翻成物理地址，然后调用 PortProxy::writeBlobPhys(range.paddr, ...)
     见 src/mem/translating_port_proxy.cc:101
  5. PortProxy::writeBlobPhys() 构造一个 WriteReq 的 Packet，然后执行 sendFunctional(&pkt)
     见 src/mem/port_proxy.cc:75
  6. 这个 sendFunctional 在 PortProxy(ThreadContext *tc, ...) 构造时，已经绑成了 tc->sendFunctional(pkt) 这个 lambda
     见 src/mem/port_proxy.cc:47
  7. ThreadContext::sendFunctional() 取当前 CPU 的 getDataPort()，然后调用这个 RequestPort 的 sendFunctional(pkt)
     见 src/cpu/thread_context.cc:157
  8. RequestPort::sendFunctional() 调 FunctionalRequestProtocol::send(_responsePort, pkt)
     见 src/mem/port.hh:579
  9. FunctionalRequestProtocol::send() 最终就是直接调用对端 peer->recvFunctional(pkt)
     见 src/mem/protocol/functional.cc:48
  10. 在当前 configs/c920 拓扑里，这个 functional 包会沿着已绑定的端口链往下走：
     CPU data port -> L1D -> L2 bus/L2 -> membus -> memory port
     原因是：

  - CPU dcache 连到 L1D：见 configs/c920/cache_hierarchy.py:96
  - L1D 再连到 L2 bus：见 configs/c920/cache_hierarchy.py:100
  - memory side 最终挂到 board.get_mem_ports() 返回的 port：见 configs/c920/cache_hierarchy.py:75
  - 对于 systemc_memory，这个 port 就是 bridge.gem5：见 configs/c920/systemc_memory.py:45 和 configs/c920/systemc_memory.py:60

  11. 所以最后到达 Gem5ToTlmBridge::recvFunctional()，桥再调用 socket->transport_dbg() 进 SystemC
     见 src/systemc/tlm_bridge/gem5_to_tlm.cc:501

## peq
peq_with_get 和 peq_with_cb_and_phase 区别和联系

## 对比simple mem和sc mem
  - Bus/XBar 给每个包打上 headerDelay=2ns、payloadDelay=3ns，见 src/mem/xbar.cc:118 和 src/mem/xbar.cc:134
  - 请求大小都是 64B
  - bandwidth 对应一次请求占用 5ns
  - latency=30ns
  - A/B/C 是 3 个不同 cache line 的 LLC miss
  - A@0ns，B@1ns，C@3ns

  1. LLC -> Bus -> SimpleMemory

  代码落点在 src/mem/simple_mem.cc:144、src/mem/simple_mem.cc:154、src/mem/simple_mem.cc:174。

  Time    LLC               Bus                    SimpleMemory
  ----    ----------------  ---------------------  -----------------------------
  0ns     miss A ---------> packet(A,h=2,p=3) --> recvTimingReq(A)
                                                  accept A immediately
                                                  busy = [0, 5]
                                                  resp_ready(A) = 0+2+3+30 = 35

  1ns     miss B ---------> packet(B,h=2,p=3) --> recvTimingReq(B)
                                                  busy, return false
                                                  B stays upstream
                                                  (still in LLC/bus side)

  3ns     miss C ---------> packet(C,h=2,p=3) --> recvTimingReq(C)
                                                  busy, return false
                                                  C also stays upstream

  5ns                                             release()
                                                  sendRetryReq()

  5ns     retry B --------> packet(B,h=2,p=3) --> recvTimingReq(B)
                                                  accept B
                                                  busy = [5, 10]
                                                  resp_ready(B) = 5+2+3+30 = 40

  10ns                                            release()
                                                  sendRetryReq()

  10ns    retry C --------> packet(C,h=2,p=3) --> recvTimingReq(C)
                                                  accept C
                                                  busy = [10, 15]
                                                  resp_ready(C) = 10+2+3+30 = 45

  这个路径里，拥塞点就在 SimpleMemory 入口。
  A 忙的时候，B/C 都卡在 memory 上游，没法下沉。

  2. LLC -> Bus -> Transactor -> SC SimpleMemory

  代码落点在：

  - bridge 用 headerDelay 作为 BEGIN_REQ 的注入延迟，并清掉 payloadDelay，见 src/systemc/tlm_bridge/gem5_to_tlm.cc:421
  - bridge 在收到 END_REQ 时才对 gem5 侧 sendRetryReq()，见 src/systemc/tlm_bridge/gem5_to_tlm.cc:223
  - target 用 busyUntil 和 payloadDuration() 推迟 END_REQ，见 src/systemc/tlm_bridge/c920_tlm_simple_mem.cc:83、src/systemc/tlm_bridge/c920_tlm_simple_mem.cc:87
  - target 在 END_REQ 之后再等 responseLatency() 发 BEGIN_RESP，见 src/systemc/tlm_bridge/c920_tlm_simple_mem.cc:93

  Time    LLC          Bus             Bridge/Transactor              SC SimpleMem
  ----    -----------  --------------  -----------------------------  -------------------------
  0ns     miss A --->  pkt(A,h=2,p=3)  recvTimingReq(A)
                                       blockingRequest = A
                                       BEGIN_REQ(A) scheduled at 2ns  -

  1ns     miss B --->  pkt(B,h=2,p=3)  bridge still blocked by A
                                       return false for B             -

  2ns                                 BEGIN_REQ(A) -----------------> target sees A
                                                                        acceptDelay = 0
                                                                        busyUntil = 2+5 = 7
                                                                        END_REQ(A) at 2ns
  2ns                                 <---------------- END_REQ(A)
                                       unblock request channel
                                       sendRetryReq() for B
                                                                        BEGIN_RESP(A) at 32ns

  2ns     retry B -->  pkt(B,h=2,p=3)  recvTimingReq(B)
                                       blockingRequest = B
                                       BEGIN_REQ(B) scheduled at 4ns  -

  3ns     miss C --->  pkt(C,h=2,p=3)  bridge blocked by B
                                       return false for C             -

  4ns                                 BEGIN_REQ(B) -----------------> target sees B
                                                                        busyUntil=7 > 4
                                                                        acceptDelay = 3
                                                                        END_REQ(B) at 7ns
                                                                        busyUntil = 12
  7ns                                 <---------------- END_REQ(B)
  9ns                                 BEGIN_REQ(C) -----------------> target sees C
                                                                        busyUntil=12 > 9
                                                                        acceptDelay = 3
                                                                        END_REQ(C) at 12ns
                                                                        BEGIN_RESP(C) at 42ns

  图里最关键的不同

  可以把两条路径压缩成一句话：

  - SimpleMemory：A 在服务时，B/C 都卡在 memory 上游
  - Bridge + SC：A 在服务时，B 已经可以更早地下沉到 bridge/target 之间，只剩 C 卡在上游

  也就是：

  SimpleMemory 拥塞时：
  [LLC/MSHR/bus侧]  B, C   |   [memory内] A

  Bridge+SC 拥塞时：
  [LLC/MSHR/bus侧]  C      |   [bridge/target等待槽] B   |   [SC memory内] A

  这相当于 Bridge + SC 比 SimpleMemory 多了一个“请求暂存槽”。

  为什么这会导致 miss / MSHR / prefetch 统计不同

  - 对不同 cache line 的 miss：
    B/C 更早地下沉到 LLC 以下，所以更早占用下游独立资源。结果通常是：
    mshrHits 变少，no_mshrs 变多。
  - 对同一 cache line，或者 prefetch 和 demand：
    由于桥这边 payloadDelay 没像 SimpleMemory 那样直接算进 resp_ready，响应可能更早回来。上面例子里：
    A 在 SC 路径是 32ns，在 SimpleMemory 是 35ns。
    这 3ns 足够让“prefetch 先填进去”还是“demand 先到 miss”发生翻转，所以：
    pfLate、pfHitInMSHR、甚至 demandMisses 都会变。

simple mem是按照duration为节点发下一个req，req节点为\[0，duration1，duration1+duration2...]
sc mem则是按照END_REQ为节点发下一个req \[0，head1,..  公式比较复杂
