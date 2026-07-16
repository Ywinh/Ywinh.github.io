---
date: 2026-04-07
categories:
  - CPU
tags:
  - Writing
---

# gem5访存梳理

https://github.com/orgs/gem5/discussions/1687

以O3CPU+classic cache为例子，梳理cacheable和non-cacheable的访存顺序

需要关注的是 CPU -> LSQ -> Cache(mshr、L1、L2) -> DDR
cacheable和non-cacheable路径的区别
乱序访存，乱的是什么部分，non-cacheable可以乱吗？

大概主线：
  IEW dispatch/execute
  -> O3 LSQ(loadQueue/storeQueue + LSQRequest)
  -> LSQ DcachePort
  -> L1 Cache(cpu_side_port)
  -> L1 MSHR / writeBuffer
  -> coherent xbar
  -> L2 Cache(cpu_side_port)
  -> L2 MSHR / writeBuffer
  -> coherent xbar / memory
  <- response 按原路返回
  <- L2 先处理自己的 MSHR
  <- L1 再处理自己的 MSHR
  <- LSQRequest 完成，指令 writeback/commit

## CPU-> LSQ

通过dispatch阶段指令分发，（此时已经是乱序的了？）识别到 load 或者 store指令，就把对应的指令放到 load 或者 store queue里面等待发射

此处的store处理有些特殊，只是把数据放到store queue，等到该条指令commit时，才能发到cache，这是因为：
  1. load 要拿回数据，结果会立刻影响后续依赖指令，所以它必须尽早发出。
  2. store 不需要从内存拿回普通结果，它真正的架构副作用是“修改内存”，这件事不能在提交前对外发生。
  3. O3 需要精确异常和可回滚性，所以 store 先留在 storeQueue，等 commit 之后再慢慢往 cache/memory 排出；load 则可以投机发出，错了再 squash 掉响应。
这个store queue有点类似store buffer，他会做store-to-load forwarding
区分write_buffer，wb位于cache，不在cpu，是用来排队向下层发writeback/clean，evict/uncacheable write这类包

等待一会儿资源就绪或者执行单元有空了，则会执行load/store
ldstQueue.executeLoad(inst) -> LSQUnit::executeLoad()

CPU会通过buildPackets创建一个包然后发出去

## LSQ->Cache

lsq::read 会进行 store-load forwarding

uncacheable read：LSQ ->loadqueue -> L1 MSHR -> L2 MSHR -> response
1. L1收到lsq的请求之后，在Cache::access中处理，如果是uncacheable，会把本级可能已有的同地址line flush
2. 无视已有mshr，强制分配一个新的mshr，强制miss，挂在mshr queue，等待发出，不创建cacheline
3. L2收到之后进行同样的流程：
  * Cache::access() 看到 UC，invalidate 本级 line，强制 miss
  - handleTimingReqMiss() 为 UC read 新分配 L2 自己的 MSHR
  - 以后再由 L2 的 sendMSHRQueuePacket() 往更下层发
1. 收到responce之后一路返回，由于这是 isForward 的 uncacheable read，is_fill 为假，所以 L1和L2不会 fill，只会 serviceMSHRTargets() 把数据交给它保存的 target（也就是来自 L1 的那个请求包）

uncacheable write：LSQ -> storequeue -> L1 writeBuffer -> L2 writeBuffer -> response
write需要等到commit时才会发出请求
* L1收到后，不走mshr而是走write_buffer，区别在于writeBuffer entry 在成功发出后就释放了，它不是等 ack 的地方。

write_buffer和mshr同时都ready，派谁出去；sendDeferredPacket -> getNextQueueEntry，后者会进行仲裁，每次只选一个queue entry
 - 如果 wq_entry ready，并且 writeBuffer.isFull()，或者当前没有 ready 的 miss_mshr，就优先尝试发 writeBuffer。
  - 但发 writeBuffer 之前，会先查 mshrQueue.findPending(wq_entry)；如果有同地址冲突、而且那个 miss 更早（order < wq_entry->order），就先发那个 MSHR。
  - 否则，如果有 miss_mshr ready，默认发 MSHR。
  - 但发 MSHR 前，还会查 writeBuffer.findPending(miss_mshr)；如果有冲突的 pending write，就先发这个 writeBuffer entry，代码注释说是为了先保住 dirty data。

gem5如何区分cacheable和uncacheable
* riscv通过PMAchecker

MMIO和uncacheable的区别？
MMIO一般需要是强序的，不能推测执行，退休时才能执行
MMIO属于uncacheable，他更多代表一种设备类型


load/store
* uncacheable
* strict order
MMIO一般是uncacheable && stirct order

 uncacheable load 如果不是 strictly ordered，执行时就能发；如果是 strictly ordered，则要等它到 ROB 头、由 commit 触发 non-spec replay 后才发。它不是“退休完成以后”才发，而是“到 commit 点、在真正退休前发”。

解释清楚，为什么cacheable和uncacheable的store路径不同
