---
date: 2026-06-28
categories:
  - gem5
tags:
  - RISC-V
  - Benchmark
---

# gem5与硬件仿真通用测试程序

**gem5对于通用测试程序的要求**：最终是一个elf形式，需要是裸机可运行程序，如果有编译工具链独特的csr，需要提前编译到gem5，不然gem5不认识会报错



## Get started

**获取源码：**

```C++
git clone https://github.com/Ywinh/gem5-resources.git
git checkout develop
cd src/simple
```

**编译工具链安装：**

需要先安装riscv gnu toolchain或者xuantie toolchain

1. riscv gnu toolchain：https://github\.com/riscv\-collab/riscv\-gnu\-toolchain

    - 需要指定安装32位且有\-march=rv32imf \-mabi=ilp32f的编译选项

2. xunatie：官网下载编译好的二进制，开箱即用

    - https://www\.xrvm\.cn/community/download?id=4460156621967921152

![Image](https://internal-api-drive-stream.feishu.cn/space/api/box/stream/download/authcode/?code=Yzc0NmZlYTlkNzQzMWZjNzY3OGNkN2U5YTRiOTMyYmFfNzRlNjMxOTM5ZDNkYTYyYjAyZjhhYWZhNWJmYzg1YzdfSUQ6NzYwMzI0ODEyOTQ0OTgzOTU3Nl8xNzgyNjQxMDMzOjE3ODI3Mjc0MzNfVjM)

**编译自己的程序**

1. 在 src/simple 下编写好 \.c 文件

2. 编译

```C++
make ISA=riscv BARE_INS=my_coremark bare CCFLAGS_ISA="-march=rv32imf -mabi=ilp32f"
```

- 编译**32位**程序：make ISA=riscv BARE\_INS=my\_coremark bare CCFLAGS\_ISA="\-march=rv32imf \-mabi=ilp32f"

- 编译**64位**程序：make ISA=riscv BARE\_INS=920\_test\_compact bare CCFLAGS\_ISA="\-march=rv64imf \-mabi=lp64f"

> \-march 和 \-mabi 可自己修改
> 
> 



需要关注

`src/simple/Makefile`

```C++
128行指定所用工具链前缀
*# PREFIX = riscv32-unknown-linux-gnu-*
PREFIX = /home/yinjianhui/Xuantie-900-gcc-elf-newlib-x86_64-V3.2.0/bin/riscv64-unknown-elf-
```

`src/simple/bootloader/riscv.S`：如下黄色部分是开启906的某些feature，需要使用xuantie工具链；去掉黄色部分则可以使用官方工具链编译

```YAML
.global _start
.section .text

_start:
    *# enable write allocate*
    *# li x3, 0x4*
    *# csrs 0x7c1,x3*
    li x3, 0x11ff
    csrs mhcr,x3

    *# enable lbuf,way_pred,data_cache_prefetch, amr*
    *# li x3, 0x7e30c*
    *# csrs 0x7c5,x3*
    li x3, 0x6e30c
    csrs mhint,x3

    la sp, stack_top

    li a0, 0

    call main

exit:
    fence                                                       
    li a7, 93
    li a0, 0        
    ecall

```





## 920 riscv\-coremark

from：https://github\.com/riscv\-boom/riscv\-coremark



Usage

- 可以打开浮点即DHAS\_FLOAT=1，不然计时不准确，从而导致跑分不准确

- 在build\-coremark\.sh修改BARE\_ITERATIONS，实测至少500轮，才能跑出结果

```Plain Text
git clone git@github.com:Ywinh/riscv-coremark.git
enable_920={0/1} BAREMETAL_ENABLE_PRINTF={0/1} ./build-coremark.sh
```

> - baremetal 版本再参数化，兼顾 gem5 和玄铁 920 硬件。
> 
> - 新增了 enable\_920 开关；打开时，启动阶段会写一组 920 专有 CSR，默认关闭，所以 gem5 默认不会碰这些 CSR，riscv64\-baremetal/crt\.S:43 build\-coremark\.sh:11。
> 
> - 新增了 BAREMETAL\_ENABLE\_PRINTF，把 stdio/printf/stats 做成可选，适合“硬件上没法方便打印”的场景，
> 
> 

为了能够在gem5运行，做的修改：

- 把 baremetal 端口从原来偏 spike/pk 的运行方式，改成更适合 gem5 直接加载的裸机程序。

- 启动代码大幅简化成 \_start \-\> main \-\> ecall exit，去掉了原先复杂的 trap/TLS/tohost/fromhost 逻辑，riscv64\-baremetal/crt\.S:1。

- 把 syscall 实现改成直接走 RISC\-V Linux ABI 风格的 ecall write/exit，而不是 proxy\-kernel 的 magic memory/tohost 机制，riscv64\-baremetal/syscalls\.c:15。

- 重写了链接脚本，固定从 0x80000000 开始布局，与gem5\-resources差不多，显式给出 \.text/\.rodata/\.data/\.bss/heap/stack，这是裸机镜像在 gem5 里运行需要的内存模型，riscv64\-baremetal/link\.ld:1。

- 把 baremetal 编译切到玄铁的 riscv64\-unknown\-elf 工具链，配上 \-march=rv64imf \-mabi=lp64f \-DHAS\_FLOAT=0，并用 \-nostdlib \-nostartfiles \-T link\.ld \-lgcc 做纯裸机链接，build\-coremark\.sh:7 riscv64\-baremetal/core\_portme\.mak:42。

- 把 Linux 版 coremark\.riscv 的构建补齐了 ISA/ABI 参数，并加了 toolchain\-compat/bin 这套 wrapper 来兼容工具链子程序查找，build\-coremark\.sh:17



## 踩过的坑（均已解决）

1. **尝试自编译测试程序的错误总结**

    1. C程序：https://github\.com/gem5/gem5\-resources/tree/develop/src/simple

        1. 在simple目录下写一个C程序，\.s程序以及\.ld脚本，复用该目录下的makefile，需要更改下riscv工具链的prefix

            - 编译`make ISA=riscv BARE_INS=xxx bare`，成功编译

            - 运行时出错，gem5中会报一个pagefault的错误，报错信息与trace如下

            ```YAML
            88000: board.processor.cores.core: T0 : 0x80000000 @_start    : auipc sp, 8192             : IntAlu :  D=0x0000000082000000
              89000: board.processor.cores.core: T0 : 0x80000004 @_start+4    : addi sp, sp, 26            : IntAlu :  D=0x000000008200001a
              90000: board.processor.cores.core: T0 : 0x80000008 @_start+8    : c_li a0, 0                 : IntAlu :  D=0x0000000000000000
              91000: board.processor.cores.core: T0 : 0x8000000a @_start+10    : jal ra, 6                  : IntAlu :  D=0x000000008000000e
             154000: board.processor.cores.core: T0 : 0x80000010 @main    : c_addi sp, -16             : IntAlu :  D=0x000000008200000a
            src/sim/faults.cc:102: panic: panic condition !handled && !tc->getSystemPtr()->trapToGdb(GDBSignal::SEGV, tc->contextId()) occurred: Page table fault when accessing virtual address 0x82000016
            Memory Usage: 1190184 KBytes
            ```

            - 定位到是sw指令访问栈上的某个地址时，访问到了一个没有映射的虚拟地址。这个比较奇怪，起初以为是栈地址太大超出了内存范围，尝试减小栈起始地址还是不能解决。

            ```C++
            // fib.c
            int main(void) {
                volatile int a = 0;
                return a;
            }
            
            // riscv.S
            .global _start
            .section .text
            
            _start:
                la sp, stack_top
            
                li a0, 0
            
                call main
            
            loop:
                j loop
                
            // ld脚本
            ENTRY(_start)
            SECTIONS
            {
              .text : {
                */bootloader.o(.text)
                *(.text)
                *(.rodata)
                *(.data)
                *(COMMON)
              }
              .bss : { *(.bss) }
              heap_low = .;
              . = . + 0x1000000;
              heap_top = .;
              . = . + 0x1000000;
              stack_top = .;
            }
            ```

        - 最后编译出来长这样

        ```YAML
        out/riscv/bare/fib.out:     file format elf64-littleriscv
        
        
        Disassembly of section .text:
        
        0000000080000000 <_start>:
            80000000:   02000117                auipc   sp,0x2000
            80000004:   01a10113                addi    sp,sp,26 # 8200001a <stack_top>
            80000008:   4501                    li      a0,0
            8000000a:   006000ef                jal     80000010 <main>
        
        000000008000000e <loop>:
            8000000e:   a001                    j       8000000e <loop>
        
        Disassembly of section .text.startup:
        
        0000000080000010 <main>:
            80000010:   1141                    addi    sp,sp,-16
            80000012:   c602                    sw      zero,12(sp) # pagefault在这里出现
            80000014:   4532                    lw      a0,12(sp)
            80000016:   0141                    addi    sp,sp,16
            80000018:   8082                    ret
        ```

    2. 汇编：https://github\.com/gem5/gem5\-resources/tree/develop/src/asmtest

        1. 首先像benchmark这些用不了，他们的系统调用很多，不能简单地处理成一个nop，我猜测如果这样处理会造成结果误差很大，甚至都不能正常运行

        2. 尝试自己写一个汇编，遵守其macro格式，但是编译时报错`invalid -march= option: `rv32g'`，查了下是riscv工具链的问题，尝试重新编译，还是有报错，已解决

    3. 目前的方法：在https://resources\.gem5\.org/页面直接下载编译好的指令集test，像是https://resources\.gem5\.org/resources/rv64uf\-p\-move?version=1\.0\.0。它们能够满足测试程序的要求



2. C程序编译bug复现

    1. 编译：

    ```C++
    cd ~/gem5-resources/src/simple
    make clean
    make make ISA=riscv BARE_INS=fib bare
    ```

    2. 运行

    ```C++
    cd ~/gem5
    ./build/RISCV/gem5.opt configs/906/run.py 
    ```

在run\.py更改文件路径可以改变workload

```C++
board.set_se_binary_workload(
    binary=BinaryResource(
        # local_path="/home/yinjianhui/gem5-resources/src/simple/out/riscv/bare/fib.out"
        # local_path="/home/yinjianhui/gem5-resources/src/simple/out/riscv/user/hello.out"
        local_path="/home/yinjianhui/gem5/riscv-test/FloatMM"
    )
)
```





3. **后续可能的解决方案**（TODO）：通过汇编来编译测试程序

    1. 首先解决编译问题

        1. 选项1：解决C程序运行问题，C程序是可以正常运行的，但是会pagefault以及不能正常退出，试着解决这个问题**（源流解决了）**

            - 把entry从0x80000000改到了0x0就解决了，这样使得访存时栈位于0\-1G这个范围。原先的0x80000000这个地址超出了这个范围。但是目前的行为还是有些奇怪，取指令的时候有地址翻译，第一条0x80000000这个指令可以取出来并且运行，但是到了访存时就会pagefault

                - **目前推测是**gem5 load elf时是真正放在cpu上而不是配置的memory上面，这造成了取指令时这个访问mem的行为是可以的，但是访存时访问到了配置的memory，这会造成pagefault，有空时可以详细看看

                - 和ld脚本有很大关系，必须

        2. 选项2：解决汇编编译问题。从已有的汇编分析ld脚本、汇编入口和退出有什么要求，中间的地方就可以自定义了，还要利用已有的makefile脚本**（已解决）**

            - 编译asmtest要求，需要编译multilib

            ```C++
            ./configure --prefix=/opt/riscv \
                        --with-arch=rv64gc_zba_zbb_zbc_zbs_zfh_zicboz \
                        --with-abi=lp64d \
                        --enable-multilib
            make linux install
            ```

            - 程序入口没有要求，退出需要调用exit，即li a7, 93然后再ecall，这样就能从系统退出
