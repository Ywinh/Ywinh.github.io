---
date: 2026-04-08
categories:
  - CPU
tags:
  - Writing
---

# gem5 处理 riscv 中断

一般的 RISCV 中断+任务切换

告诉硬件“当前 hart 可以暂停，等到中断可能需要服务时再继续”的指令。  
但它不保证一定休眠，甚至实现成 `NOP` 也合法；因此软件在 `WFI` 后必须自己检查是否真的有需要处理的事件

一般的中断来源，中断发起，中断处理

1 中断发起，在 hart 这一侧，硬件会体现为 mip（machine interrup pending） 里的某些 pending 位有效，比如：
  - mip.MSIP
  - mip.MTIP
  - mip.MEIP
  - 这里的关键是：中断请求先到达hart并变成pending，不等于立刻跳去中断处理函数

2 核心在指令边界检查能否响应该中断：
  处理器通常在指令提交边界检查：

  - 全局中断使能是否开了：mstatus.MIE
  - 该类中断是否单独使能：mie 对应位
  - 当前特权级是否允许被这个级别的中断打断
  - 是否被委托到更低级模式：mideleg / sideleg（裸机通常不配，默认都进 M 模式）
  - 如果同时有多个 pending，中断控制器/架构优先级规则决定先接哪个

中断是异步的，但通常是在指令之间被接收

中断响应，hart允许接收中断之后，会做如下
* 设置中断返回地址，mepc = pc+4
* 把中断原因写入mcause
* 更新mstatus
	* MPIE = MIE
	* MIE = 0 （关中断）
	* MPP = 之前的特权级
* pc = mtvec（中断设置的trap地址）
* mtvec 有两种常见模式：

- Direct：所有 trap 都跳到 BASE
- Vectored：中断跳到 BASE + 4 * cause，异常还是跳 BASE

硬件不会帮你做的事情：
* 不会自动保存通用寄存器x1～x31
* 不会自动切换栈
* 不会恢复寄存器
* 中断处理完成之后不会执行mret

mret时硬件的动作：
- pc <= mepc
- MIE <= MPIE
- 当前特权级恢复为 MPP
- MPIE/MPP 按架构规则复位到返回后的状态
