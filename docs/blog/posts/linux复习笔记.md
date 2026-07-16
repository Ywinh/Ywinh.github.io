---
date: 2026-02-08
categories:
  - Linux
tags:
  - Writing
---

fork时：复制task_struct以及复制页表（copy on write）
进程 0 复制进程 1 时页表的复制只有 160 项，也就是映射 640K，而之后进程的复制，统统都是复制 1024 项，也就是映射 4M 空间

线性地址转为物理地址
![](https://calculet.feishu.cn/space/api/box/stream/download/asynccode/?code=MjMwZTVjNDRmNTg5NzIzNTVjMzBjMzhjNDQzNTJkZDhfVm5KUTBuR050NFRMQXR2Tk0ydWR4Y3cwNUVBM0ppcXNfVG9rZW46UGU0QmJLMjhLb3NvWVJ4dnZGbWN3dE90blBoXzE3ODE0MjM2OTk6MTc4MTQyNzI5OV9WNA&add_watermark=true&scene_type=CCM)

  

注意copy_on_write的时候，如果计数为一，不是什么也不干，而是要恢复其写权限
从0号进程fork出1号进程之后，`init`函数做的事情

setup主要工作：加载根文件系统，以便后续可以顺着根inode找到所有文件
flip是进程的文件描述符数组，大小是20；file_table是系统文件表，大小是64。可以得出一个进程最多打开20个文件，整个系统最多打开64个文件

  

# 思考题1

**1.为什么开始启动计算机的时候，执行的是BIOS代码而不是操作系统自身的代码？**

- 计算机被设计为从内存中运行程序，无法直接从软盘或者硬盘中运行。最开始启动计算机的时候，计算机内存未初始化，没有任何程序。而因为CPU只能读取内存中的程序，所以必须将操作系统先加载进内存当中，需要使用BIOS引导。
- 在加电后， BIOS 需要完成一些硬件检测工作，同时设置实模式下的中断向量表和服务程序，并将操作系统的引导扇区加载至 0x7C00 处，然后将跳转至 0x7C00运行操作系统自身的代码。
- BIOS程序存放在ROM中，ROM断电后也能保持信息，但一被烧就不能改变数据，适合存放BIOS这种不需要修改的例行工作。所以计算机启动最开始运行的是BIOS代码。

**2.为什么BIOS只加载了一个****扇区****，后续扇区却是由bootsect代码加载？为什么BIOS没有直接把所有需要加载的扇区都加载？**

- BIOS和操作系统的开发职责不同，BIOS的设计必须遵循**通用性**原则，而操作系统的具体结构是**特异性**的。
- 按固定的规则约定，可以进行灵活的各自设计相应的部分。BIOS接到启动操作系统命令后，只从启动扇区将代码加载至0x7c00(BOOTSEG)位置，而后续扇区由bootsect代码加载，这些代码由编写系统的用户负责，与之前BIOS无关。这样构建的好处是站在整个体系的高度，统一设计和统一安排，简单而有效。BIOS和操作系统的开发都可以遵循这一约定，灵活地进行各自的设计。
- 例如，BIOS可以不用知道内核镜像的大小以及其在软盘的分布等等信息，减轻了BIOS程序的复杂度，降低了硬件上的开销。而操作系统的开发者也可以按照自己的意愿，内存的规划，等等都更为灵活。另外，如果要使用BIOS进行加载，而且加载完成之后再执行，则需要很长的时间，此外，对于不同的操作系统，其代码长度不一样，可能导致操作系统加载不完全。因此Linux采用的是边执行边加载的方法

**3.为什么BIOS把bootsect加载到0x07c00，而不是0x00000？加载后又马上挪到0x90000处，是何道理？为什么不一次加载到位？**

- 加载0x07c00是BIOS提前约定设置的，BIOS把bootsect加载到0x07c00而不是0x00000，是因为0x00000处存放着BIOS构建的1k大小的中断向量表和256B的BIOS数据区，这些数据还有用处，不能进行覆盖。
    
- 加载后又挪到0x90000是因为，操作系统对内存的规划是在0x90000存放bootsect，然后bootsect执行结束之后，立即将系统机器数据存放在此处，这样就可以及时回收寿命结束的程序占据的内存空间。而且后续会把120K的系统模块存放到0x00000处，这会覆盖0x07c00处的代码和数据。
    
- 不一次加载到位的原因是由于“两头约定”和“定位识别”，所以在开始时bootsect“被迫”加载到0X07c00位置。现在将自身移至0x90000处，说明操作系统开始根据自己的需要安排内存了。
    

![](https://calculet.feishu.cn/space/api/box/stream/download/asynccode/?code=ZjM4NjdlZDRhMThmYzYwZTkyMGUzYWNhNjVmOWM4OWNfS0tXcmhUMkFNNGFLZ3BlemJuSHA4SXFzUEtuQ0JDeVhfVG9rZW46SWdEc2JOMWIxb2ZlNVJ4NzVjRmNuOXJabkFxXzE3ODE0MjM2OTk6MTc4MTQyNzI5OV9WNA&add_watermark=true&scene_type=CCM)

  

**4.bootsect、setup、head程序之间是怎么衔接的？给出代码证据。**

① bootsect跳转到setup程序：jmpi 0,SETUPSEG;

bootsect首先利用int 0x13中断分别加载setup程序及system模块，待bootsect程序的任务完成之后，执行代码jmpi 0,SETUPSEG。由于 bootsect 将 setup 段加载到了 SETUPSEG:0 （0x90200）的地方,在实模式下，CS:IP指向setup程序的第一条指令，此时setup开始执行。

② setup跳转到head程序：jmpi 0,8

执行setup后，内核被移到了0x00000处，CPU变为保护模式，执行jmpi 0,8

并加载了中断描述符表和全局描述符表。该指令执行后跳转到以GDT第2项中的 base_addr 为基地址，以0为偏移量的位置，其中base_addr为0。由于head放置在内核的头部，因此程序跳转到head中执行。

  

  

**5.setup程序的最后是jmpi 0,8 ，为什么这个8不能简单的当作阿拉伯数字8看待，究竟有什么内涵？**

- 此时为32位保护模式，“0”表示段内偏移，“8”表示段选择符。
    
- 8要转化为二进制：1000，最后两位00表示内核特权级（若是11则表示用户），第三位0表示 GDT 表（若是1则表示LDT表），第四位1表示根据GDT中的第2项来确定代码段的段基址和段限长等信息。可以得到代码是从head 的开始位置，段基址 0x00000000、偏移为 0 处开始执行的，即head的开始位置。
    

|   |   |   |   |
|---|---|---|---|
|16位段选择符的结构|15-3|2|1,0|
||描述符表内的索引|0代表GDT，代表LDT|请求的特权级0-3|

  

**6.保护模式在“保护”什么？它的“保护”体现在哪里？特权级的目的和意义是什么？分页有“保护”作用吗？**

（1） 保护模式在“保护”什么？它的“保护”体现在哪里？

- 保护操作系统的安全，不受到恶意攻击。保护进程地址空间。
    
- “保护”体现在：打开保护模式后，CPU 的寻址模式发生了变化，基于 GDT 去获取代码或数据段的基址，相当于增加了一个段位寄存器。防止了对代码或数据段的覆盖以及代码段自身的访问超限，明显增强了保护作用。对描述符所描述的对象进行保护：在 GDT、 LDT 及 IDT 中，均有对应界限、特权级等，这是对描述符所描述的对象的保护；在不同特权级间访问时，系统会对 CPL、 RPL、 DPL、 IOPL 等进行检验，同时限制某些特殊指令如 lgdt, lidt,cli 等的使用；分页机制中 PDE 和 PTE 中的 R/W 和 U/S 等提供了页级保护，分页机制通过将线性地址与物理地址的映射，提供了对物理地址的保护。
    

  

（2）特权级的目的和意义是什么？

- 特权级机制目的是为了进行合理的管理资源，保护高特权级的段。其中操作系统的内核处于最高的特权级。
    
- 意义是进行了对系统的保护，对操作系统的“主奴机制”影响深远。Intel 从硬件上禁止低特权级代码段使用部分关键性指令，通过特权级的设置禁止用户进程使用 cli、 sti 等对掌控局面至关重要的指令。有了这些基础，操作系统可以把内核设计成最高特权级，把用户进程设计成最低特权级。这样，操作系统可以访问 GDT、 LDT、 TR，而 GDT、 LDT 是逻辑地址形成线性地址的关键，因此操作系统可以掌控线性地址。物理地址是由内核将线性地址转换而成的，所以操作系统可以访问任何物理地址。而用户进程只能使用逻辑地址。总之，特权级的引入对操作系统内核进行保护。
    

  

（3）分页有“保护”作用吗？

- 分页机制有保护作用，使得用户进程不能直接访问内核地址，进程间也不能相互访问。用户进程只能使用逻辑地址，而逻辑地址通过内核转化为线性地址，根据内核提供的专门为进程设计的分页方案，由MMU非直接映射转化为实际物理地址形成保护。此外，通过分页机制，每个进程都有自己的专属页表，有利于更安全、高效的使用内存，保护每个进程的地址空间。
    

  

为什么特权级是基于段的？（超纲备用）

- 在操作系统设计中，一个段一般实现的功能相对完整，可以把代码放在一个段，数据放在一个段，并通过段选择符（包括CS、SS、DS、ES、Fs和GS）获取段的基址和特权级等信息。通过段，系统划分了内核代码段、内核数据段、用户代码段和用户数据段等不同的数据段，有些段是系统专享的，有些是和用户程序共享的，因此就有特权级的概念。特权级基于段，这样当段选择子具有不匹配的特权级时，按照特权级规则评判是否可以访问。特权级基于段，是结合了程序的特点和硬件实现的一种考虑。
    

  

**7.在setup程序里曾经设置过gdt，为什么在head程序中将其废弃，又重新设置了一个？为什么设置两次，而不是一次搞好？**

- 原来GDT所在的位置是设计代码时在setup.s里面设置的数据，将来这个setup模块所在的内存位置会在设计缓冲区时被覆盖。如果不改变位置，将来GDT的内容肯定会被缓冲区覆盖掉，从而影响系统的运行。这样一来，将来整个内存中唯一安全的地方就是现在head.s所在的位置了。
    
- 那么有没有可能在执行setup程序时直接把GDT的内容复制到head.s所在的位置呢？肯定不能。
    
    - 如果先复制GDT的内容，后移动system模块，它就会被后者覆盖
        
    - 如果先移动system模块，后复制GDT的内容，它又会把head.s对应的程序覆盖，而这时head.s还没有执行。所以，无论如何，都要重新建立GDT。
        

  

**8.进程0的task_struct在哪？具体内容是什么？**

- 进程0的task_struct位于内核数据区，因为在进程0未激活之前，使用的是boot阶段的user_stack（在运行进程0之前它是内核栈，以后用作进程0和1的用户态栈），因此存储在user_stack中。
    
- 具体内容：包含了进程 0 的进程状态、进程 0 的 LDT、进程 0 的 TSS 等等。其中 ldt 设置了代码段和堆栈段的基址和限长(640KB)，而 TSS 则保存了各种寄存器的值，包括各个段选择符。
    

```C
/*
 *  INIT_TASK is used to set up the first task table, touch at
 * your own risk!. Base=0, limit=0x9ffff (=640kB)
 */
#define INIT_TASK \
/* state etc */ { 0,15,15, \      // state, counter, priority 
/* signals */   0,{{},},0, \      // signal, sigaction[32], blocked 
/* ec,brk... */ 0,0,0,0,0,0, \    // exit_code,start_code,end_code,end_data,brk,start_stack
/* pid etc.. */ 0,-1,0,0,0, \     // pid, father, pgrp, session, leader 
/* uid etc */   0,0,0,0,0,0, \    // uid, euid, suid, gid, egid, sgid 
/* alarm */ 0,0,0,0,0,0, \        // alarm, utime, stime, cutime, cstime, start_time 
/* math */  0, \                  // used_math
/* fs info */   -1,0022,NULL,NULL,NULL,0, \  // tty,umask,pwd,root,executable,close_on_exec 
/* filp */  {NULL,}, \            // flip[20]
    { \                           
        {0,0}, \                  // ldt[3] 
/* ldt */   {0x9f,0xc0fa00}, \    // 代码长640K，基址0x0，G=1，D=1，DPL=3，P=1 TYPE=0x0a
        {0x9f,0xc0f200}, \        // 数据长640K，基址0x0，G=1，D=1，DPL=3，P=1 TYPE=0x02 
    }, \
/*tss*/ {0,PAGE_SIZE+(long)&init_task,0x10,0,0,0,0,(long)&pg_dir,\
     0,0,0,0,0,0,0,0, \
     0,0,0x17,0x17,0x17,0x17,0x17,0x17, \
     _LDT(0),0x80000000, \
        {} \
    }, \
}
```

ldt：

![](https://calculet.feishu.cn/space/api/box/stream/download/asynccode/?code=MjBmNTVlOTI0NGY1ZDE5NTIyODU0MmVmM2UwMTQ3NmFfaWFTbjJrTEZCY0xGVG1MSGt6c0pZUGx4cXptME9aamZfVG9rZW46UlQ0ZWJpVXpmb2dRbkt4TkJDbGNhbmw5bkRiXzE3ODE0MjM2OTk6MTc4MTQyNzI5OV9WNA&add_watermark=true&scene_type=CCM)

当 G=1 时，限长的单位是 **4KB (页)**；当 G=0 时，单位是字节

Size = (段限长+1) * （G==1 ? 4KB : byte）

  

**9.内核的线性地址空间是如何分页的？画出从0x000000开始的7个页（包括页目录表、页表所在页）的挂接关系图，就是页目录表的前四个页目录项、第一个个页表的前7个页表项指向什么位置？给出代码证据。**

- 如何分页：head.s在setup_paging开始创建分页机制。将页目录表和4个页表放到物理内存的起始位置，从内存起始位置开始的5个页空间内容全部清零（每页4KB），然后设置页目录表的前4项，使之分别指向4个页表。然后开始从高地址向低地址方向填写4个页表，依次指向内存从高地址向低地址方向的各个页面。即将第4个页表的最后一项指向寻址范围的最后一个页面。即从0xFFF000开始的4kb 大小的内存空间。将第4个页表的倒数第二个页表项指向倒数第二个页面，即0xFFF000-0x1000000开始的4KB字节的内存空间，依此类推。
    
    ```Bash
    //代码路径：boot\head.s
        /*       
        * 这个子程序通过设置控制寄存器cr0的标志（PG 位31）来启动对内存的分页处理功能，       
        * 并设置各个页表项的内容，以恒等映射前16 MB的物理内存。分页器假定不会产生非法的       
        * 地址映射（也即在只有4Mb的机器上设置出大于4Mb的内存地址）。       
        * 注意！尽管所有的物理地址都应该由这个子程序进行恒等映射，但只有内核页面管理函数能       
        * 直接使用>1Mb的地址。所有“一般”函数仅使用低于1Mb的地址空间，或者是使用局部数据       
        * 空间，地址空间将被映射到其它一些地方去 -- mm(内存管理程序)会管理这些事的。       
        * 对于那些有多于16Mb内存的家伙 – 真是太幸运了，我还没有，为什么你会有☺。代码就在       
        * 这里，对它进行修改吧。（实际上，这并不太困难的。通常只需修改一些常数等。我把它设置       
        * 为16Mb，因为我的机器再怎么扩充甚至不能超过这个界限（当然，我的机器是很便宜的☺）。       
        * 我已经通过设置某类标志来给出需要改动的地方（搜索“16Mb”），但我不能保证作这些       
        * 改动就行了）。       
        */ 
        # 在内存物理地址0x0处开始存放1页页目录表和4页页表。页目录表是系统所有进程公用的，而      
        # 这里的4页页表则是属于内核专用。对于新的进程，系统会在主内存区为其申请页面存放页表。      
        # 1页内存长度是4096字节。 
    .align 2
    setup_paging:  # 首先对5页内存（1页目录 + 4页页表）清零
            movl $1024*5,%ecx                /* 5 pages - pg_dir+4 page tables */
            xorl %eax,%eax
            xorl %edi,%edi                        /* pg_dir is at 0x000 */
            cld;rep;stosl
         # "$pg0+7"表示：0x00001007，是页目录表中的第1项。
            movl $pg0+7,_pg_dir                /* set present bit/user r/w */
            movl $pg1+7,_pg_dir+4                /*  --------- " " --------- */
            movl $pg2+7,_pg_dir+8                /*  --------- " " --------- */
            movl $pg3+7,_pg_dir+12                /*  --------- " " --------- */
            movl $pg3+4092,%edi
            movl $0xfff007,%eax                /*  16Mb - 4096 + 7 (r/w user,p) */
            std
    1:        stosl                        /* fill pages backwards - more efficient :-) */
            subl $0x1000,%eax
            jge 1b
        # 设置页目录基址寄存器cr3的值，指向页目录表。
            xorl %eax,%eax                /* pg_dir is at 0x0000 */
            movl %eax,%cr3                /* cr3 - page directory start */
       # 设置启动使用分页处理（cr0的PG标志，位31）
            movl %cr0,%eax
            orl $0x80000000,%eax
            movl %eax,%cr0                /* set paging (PG) bit */
            ret                        /* this also flushes prefetch-queue */
    ```
    

![](https://calculet.feishu.cn/space/api/box/stream/download/asynccode/?code=ZDNkNTkyYzQwNGJjNzBmNTc4NjMxZGUxZDFjNDI4Y2RfYjNIZmZveW5BT280QkgxMHdOczB1S0pZWVJZcEVGYktfVG9rZW46T1VBTGJsZldzb0EwVDV4cHRxTmNzVUdyblpiXzE3ODE0MjM2OTk6MTc4MTQyNzI5OV9WNA&add_watermark=true&scene_type=CCM)

  

**10.在head程序执行结束的时候，在idt的前面有184个字节的head程序的剩余代码，剩余了什么？为什么要剩余？**

剩余代码：

- 包含代码段如下：after_page_tables(栈中压入了些参数)、 ignore_int(初始化中断时的中断处理函数) 和 setup_paging(初始化分页)。
    

剩余的原因：

- after_page_tables 中压入的参数，为内核进入 main 函数的跳转做准备。设计者在栈中压入了 L6: main，以使得系统出错时，返回到 L6 处执行。需要保留
    
- ignore_int 为中断处理函数，使用 ignore_int 将 idt 全部初始化，如果中断开启后存在使用了未设置的中断向量，那么将默认跳转到 ignore_int 处执行，使得系统不会跳转到随机的地方执行错误的代码。
    
- setup_paging 进行初始化分页，是在初始化页目录表及四个内核页表, 这是进入main函数前最后的操作, 没有办法被覆写, 只能剩余
    

  

**11.为什么不用call，而是用ret“调用”main函数？画出调用路线图，给出代码证据。**

- call指令会将EIP的值自动压栈，保护返回现场，然后执行被调函数的程序，等到执行被调函数的ret 指令时，自动出栈给EIP并还原现场，继续执行call的下一条指令。然而对操作系统的main 函数来说，如果用call 调用main函数，那么ret时返回给谁呢？在由head程序向main函数跳转时，是不需要main函数返回的；同时由于main函数已经是最底层的函数了，没有更底层的支撑函数支持其返回。
    
- 用ret 实现的调用操作当然就不需要返回了，call做的压栈和跳转动作需要手工编写代码，模仿了call的全部动作，实现了调用setup_paging函数。压栈的EIP值不是调用setup_paging函数的下一行指令的地址，而是操作系统的main函数的执行入口地址，这样当setup_paging函数执行到ret时，从栈中将操作系统的main函数的执行入口地址_main自动出栈给EIP，EIP指向main函数的入口地址,实现了用返回指令调用main函数。所以要达到既调用main又不需返回，就不采用call而是选择了ret“调用”了。
    

![](https://calculet.feishu.cn/space/api/box/stream/download/asynccode/?code=M2Q0NzA4ZDk0YTJiMmMwMjAxYjI2MGMxODQyOGEzZGNfall0bEJjbWtuTzMwQnpocW9IZXNTNE53ZGlDZDBXVjdfVG9rZW46VWs4ZmJQV1BzbzZDOGl4MGhxb2N1YWJHbnNnXzE3ODE0MjM2OTk6MTc4MTQyNzI5OV9WNA&add_watermark=true&scene_type=CCM)

**12.用文字和图说明中断描述符表是如何初始化的，可以举例说明（比如：set_trap_gate(0,&divide_error)），并给出代码证据。**

![](https://calculet.feishu.cn/space/api/box/stream/download/asynccode/?code=ZGFlOGMyZGQ0M2M5ZjI0YWM3MDc5ZTVkM2JiNzQxNDFfUlJUM2NjOVRFWlFieWdWSEllZlh6cTJyWGdsb0ZVdnlfVG9rZW46T2Y4YmJlNFFnb1JlcE54cUM3Z2N1eHM1bkNoXzE3ODE0MjM2OTk6MTc4MTQyNzI5OV9WNA&add_watermark=true&scene_type=CCM)

![](https://calculet.feishu.cn/space/api/box/stream/download/asynccode/?code=NWZjZjZiOGQzMTMwYTc5ZTVlZDNhMjc2OTAxZDZmYzZfdGFvM2VRMlE5VTdlVlJndGZOQmVCN0RVbGY0d2pxUndfVG9rZW46S2EzVmJLYmhrb0phOVh4QmxoQWNlbUxIblVmXzE3ODE0MjM2OTk6MTc4MTQyNzI5OV9WNA&add_watermark=true&scene_type=CCM)

举例子：set_trap_gate(0,&divide_error)，等价于set_gate(&idt[0],15,0,addr)，type为15，特权级是0

  

**13.在IA-32中，有大约20多个指令是只能在0特权级下使用，其他的指令，比如****cli****，并没有这个约定。奇怪的是，在Linux0.11中，3特权级的进程代码并不能使用cli指令，这是为什么？请解释并给出代码证据。**

- 根据Intel Manual，cli和sti指令与CPL和EFLAGS[IOPL]有关。通过IOPL来加以保护指令in,ins,out,outs,cli,sti等I/O敏感指令，只有CPL(当前特权级)<=IOPL才能执行，低特权级访问这些指令将会产生一个一般性保护异常。
    
- IOPL位于EFLAGS的12-13位，仅可通过iret来改变，INIT_TASK中IOPL为0，在move_to_user_mode中直接执行“pushfl \n\t”指令，继承了内核的EFLAGS。IOPL的指令仍然为0没有改变，所以用户进程无法调用cli指令。因此，通过设置 IOPL， 3特权级的进程代码不能使用 cli 等I/O敏感指令。
    

具体代码：move_to_user_mode()此处一共两部分代码第一部分 P79

```C
#define move_to_user_mode() \
asm(“movl %%esp, %%eax\n\t” \
 ……
 "pushfl\n\t” \ // ELAGS 进栈
 ……
")
```

  

第二部分代码见P 68 INIT_TASK，在哪呢？

![](https://calculet.feishu.cn/space/api/box/stream/download/asynccode/?code=ODYzY2IzY2VhMWYwZDc4ODA0OGM3NDdhN2JlZWMyMGRfWVhoRlNTN2lZbGVwU09PdVl4Q0ViT1BxRUNGajM3aEhfVG9rZW46S2F4d2J6bldEbzh1ZXp4dTRNZmM2SHBSbktmXzE3ODE0MjM2OTk6MTc4MTQyNzI5OV9WNA&add_watermark=true&scene_type=CCM)

  

**14.进程0的task_struct、内核栈在哪？具体内容是什么？给出代码证据。**

与问题8相同？

  

**15.在system.h里**

```Java
#define _set_gate(gate_addr,type,dpl,addr) \
asm ("movw %%dx,%%ax\n\t" \
    "movw %0,%%dx\n\t" \
    "movl %%eax,%1\n\t" \
    "movl %%edx,%2" \
    : \
    : "i" ((short) (0x8000+(dpl<<13)+(type<<8))), \
    "o" (*((char ) (gate_addr))), \
    "o" ((4+(char *) (gate_addr))), \
    "d" ((char *) (addr)),"a" (0x00080000))

#define set_intr_gate(n,addr) \
    _set_gate(&idt[n],14,0,addr)

#define set_trap_gate(n,addr) \
    _set_gate(&idt[n],15,0,addr)

#define set_system_gate(n,addr) \
    _set_gate(&idt[n],15,3,addr)
```

**读懂代码。这里中断门、陷阱门、系统调用都是通过_set_gate设置的，用的是同一个嵌入汇编代码，比较明显的差别是dpl一个是3，另外两个是0，这是为什么？说明理由。**

set_trap_gate 和set_intr_gate的dpl是3，set_system_gate的dpl是0。dpl为0表示只能在内核态下允许，dpl为3表示系统调用可以由3特权级调用。

当用户程序产生系统调用软中断后， 系统都通过system_call总入口找到具体的系统调用函数。 set_system_gate设置系统调用，须将 DPL设置为 3，允许在用户特权级的进程调用，否则会引发 General Protection 异常。set_trap_gate 及 set_intr_gate 设置陷阱和中断为内核使用，需禁止用户进程调用，所以 DPL为 0。

  

**16.进程0 fork进程1之前，为什么先调用move_to_user_mode()？用的是什么方法？解释其中的道理。**

Linux规定，除了进程0外，所有进程都要由一个已有的进程在3特权级下创建，进程0此时处于0特权级。按照规定，在创建进程1之前要将进程0转变为3特权级。方法是调用move_to_user_mode()函数，模仿中断返回动作，实现进程0的特权级从内核态转化为用户态。又因为在Linux-0.11中，转换特权级时采用中断和中断返回的方式，调用系统中断实现从3到0的特权级转换，中断返回时转换为3特权级。因此，进程0从0特权级到3特权级转换时采用的是模仿中断返回。设计者首先手工写压栈代码模拟int（中断）压栈，当执行iret指令时，CPU自动将这5个寄存器的值（SS，ESP,EFLAGS，CS，EIP）按序恢复给CPU，CPU就会翻转到3特权级去执行代码。

```C
#define move_to_user_mode() \
__asm__ ("movl %%esp,%%eax\n\t" \  // 保存堆栈指针esp到eax寄存器中。
        "pushl $0x17\n\t" \        // 首先将堆栈段选择符(SS)入栈。
        "pushl %%eax\n\t" \        // 然后将保存的堆栈指针值(esp)入栈。
        "pushfl\n\t" \            // 将标志寄存器(eflags)内容入栈。
        "pushl $0x0f\n\t" \        // 将Task0代码段选择符(cs)入栈。
        "pushl $1f\n\t" \        // 将下面标号1的偏移地址(eip)入栈。 
        "iret\n" \                 // 执行中断返回指令，则会跳转到下面标号1处。
        "1:\tmovl $0x17,%%eax\n\t" \    // 此时开始执行任务0， 
        "movw %%ax,%%ds\n\t" \    // 初始化段寄存器指向本局部表的数据段。 
        "movw %%ax,%%es\n\t" \
        "movw %%ax,%%fs\n\t" \
        "movw %%ax,%%gs" \
        :::"ax")
```

  

**17.进程0的用户栈在哪？给出代码证据。**

进程 0 的用户栈位于内核静态定义的 `user_stack` 数组中。通常认为用户栈是在 `execve` 时动态分配在 64MB 地址空间顶端的。但 进程 0 是个特例，它直接复用了系统启动时的临时栈作为自己的用户栈。

user_stack定义时段选择子为0x10，即内核数据段。当 `main.c` 执行到最后，调用 `move_to_user_mode()` 变身为进程 0（进入用户态）时，发生了关键的“复用”。首先将esp复制到eax，然后把eax放在esp的位置压栈，后面iret时弹出esp，即用户0栈指针指向了user_stack

  

**18.static inline _syscall0(int,fork)中，为什么要加inline？去掉会有什么问题？给出详细论证。**

因为_syscall0(int,fork)展开是一个真函数，普通真函数调用时需要将eip入栈，返回时需要将eip出栈。inline是内联函数，它将标明为inline的函数代码放在符号表中，而此处的fork函数需要调用两次，加上inline后先进行词法分析、语法分析正确后就地展开函数，不需要有普通函数的call\ret等指令，也不需要保持栈的eip，效率很高。若不加上inline，第一次调用fork结束时将eip 出栈，第二次调用返回的eip出栈值将是一个错误值。

答案2：inline一般是用于定义内联函数，内联函数结合了函数以及宏的优点，在定义时和函数一样，编译器会对其参数进行检查；在使用时和宏类似，内联函数的代码会被直接嵌入在它被调用的地方，这样省去了函数调用时的一些额外开销，比如保存和恢复函数返回地址等，可以加快速度。

  

**19.copy_process函数的参数最后五项是：long eip,long cs,long eflags,long esp,long ss。查看栈结构确实有这五个参数，奇怪的是其他参数的压栈代码都能找得到，确找不到这五个参数的压栈代码，****反汇编****代码中也查不到，请解释原因。**

- 在 fork()中， 当执行“int $0x80” 时产生一个软中断， 使 CPU 硬件自动将 SS、 ESP、EFLAGS、 CS、 EIP 这 5 个寄存器的数值按这个顺序压入进程 0 的内核栈。 硬件压栈可确保 eip 的值指向正确的指令， 使中断返回后程序能继续执行。因为通过栈进行函数传递参数，所以恰可做为 copy_process 的最后五项参数。
    

  

**20.分析get_free_page()函数的代码，叙述在主****内存****中获取一个空闲页的技术路线。**

通过逆向扫描页表位图 mem_map， 并由第一空页的下标左移 12 位加 LOW_MEM 得到该页的物理地址， 位于 16M 内存末端。P89代码考试不用看

过程：

① 将EAX 设置为0,EDI 设置指向mem_map 的最后一项（mem_map+PAGING_PAGES-1），std设置扫描是从高地址向低地址。从mem_map的最后一项反向扫描，找出引用次数为0(AL)的页，如果没有则退出；如果找到，则将找到的页设引用数为1；

② ECX左移12位得到页的相对地址，加LOW_MEM得到物理地址，将此页最后一个字节的地址赋值给EDI（LOW_MEM+4092）；

③ stosl将EAX的值设置到ES:EDI所指内存，即反向清零1024*32bit，将此页清空；

④ 将页的地址（存放在EAX）返回。

  

# 思考题2

**1.分析copy_page_tables（）函数的代码，叙述父进程如何为****子进程****复制页表。**

- 先为新的页表申请一个空闲页面，并把进程0中第一个页表里的前160个页表项复制到这个页面中（其他进程是1k个页表项？）。这使得进程0和进程1的页表暂时都指向了相同的页面， 此后两个进程将共享内存区，直到有一个进程执行写操作时，才分配新的内存页（写时复制机制）。之后对进程1的页目录表进行设置
    

![](https://calculet.feishu.cn/space/api/box/stream/download/asynccode/?code=YzZmOWY5ZmJiOGUyMjM4ZTE0NmJkNzE0ZWQwZDI1MTlfbFVJamNqZzZBWFZiVGVLQTZramcwbFBESFZxQjhNTzdfVG9rZW46RkdQMmJxRkNmb1JMdkV4VVdUQ2NnRW1vbmxoXzE3ODE0MjM2OTk6MTc4MTQyNzI5OV9WNA&add_watermark=true&scene_type=CCM)

  

**2.进程0创建进程1时，为进程1建立了task_struct及内核栈，第一个页表，分别位于****物理内存****16MB顶端倒数第一页、第二页。请问，这两个页究竟占用的是谁的线性地址空间，内核、进程0、进程1、还是没有占用任何线性地址空间？说明理由（可以图示）并给出代码证据。**

答：均占用内核的线性地址空间， 原因如下：

通过逆向扫描页表位图，并由第一空页的下标左移 12 位加 LOW_MEM 得到该页的物理地址，位于 16M 内存末端。 代码如下

```C++
unsigned long get_free_page(void)
{
register unsigned long __res asm("ax");

__asm__("std ; repne ; scasb\n\t"   // 置方向位，al(0)与对应每个页面的(di)内容比较
    "jne 1f\n\t"                    // 如果没有等于0的字节，则跳转结束(返回0).
    "movb $1,1(%%edi)\n\t"          // 1 => [1+edi],将对应页面内存映像bit位置1.
    "sall $12,%%ecx\n\t"            // 页面数*4k = 相对页面其实地址
    "addl %2,%%ecx\n\t"             // 再加上低端内存地址，得页面实际物理起始地址
    "movl %%ecx,%%edx\n\t"          // 将页面实际其实地址->edx寄存器。
    "movl $1024,%%ecx\n\t"          // 寄存器ecx置计数值1024
    "leal 4092(%%edx),%%edi\n\t"    // 将4092+edx的位置->dei（该页面的末端地址）
    "rep ; stosl\n\t"               // 将edi所指内存清零(反方向，即将该页面清零)
    "movl %%edx,%%eax\n"            // 将页面起始地址->eax（返回值）
    "1:"
    :"=a" (__res)
    :"0" (0),"i" (LOW_MEM),"c" (PAGING_PAGES),
    "D" (mem_map+PAGING_PAGES-1)
    );
return __res;           // 返回空闲物理页面地址(若无空闲页面则返回0).
}
```

进程 0 和进程 1 的 LDT 的 LIMIT 属性将进程 0 和进程 1 的地址空间限定0~640KB， 所以进程 0、 进程 1 均无法访问到这两个页面， 故两页面占用内核的线性地址空间。进程 0 的局部描述符如下

```Bash
//代码路径：boot\head.s
.align 2
setup_paging:
        movl $1024*5,%ecx                /* 5 pages - pg_dir+4 page tables */
        xorl %eax,%eax
        xorl %edi,%edi                        /* pg_dir is at 0x000 */
        cld;rep;stosl
        movl $pg0+7,_pg_dir                /* set present bit/user r/w */
        movl $pg1+7,_pg_dir+4                /*  --------- " " --------- */
        movl $pg2+7,_pg_dir+8                /*  --------- " " --------- */
        movl $pg3+7,_pg_dir+12                /*  --------- " " --------- */
        movl $pg3+4092,%edi
        movl $0xfff007,%eax                /*  16Mb - 4096 + 7 (r/w user,p) */
        std
1:        stosl                        /* fill pages backwards - more efficient :-) */
        subl $0x1000,%eax
        jge 1b
        xorl %eax,%eax                /* pg_dir is at 0x0000 */
        movl %eax,%cr3                /* cr3 - page directory start */
        movl %cr0,%eax
        orl $0x80000000,%eax
        movl %eax,%cr0                /* set paging (PG) bit */
        ret                        /* this also flushes prefetch-queue */
```

上面的代码，指明了内核的线性地址空间为0x000000~Oxffffff(即前16M），且线性地址与物理地址呈现一一对应的关系。为进程1分配的这两个页，在16MB的顶端倒数第一页、第二页，因此占用内核的线性地址空间。

进程0的线性地址空间是内存前640KB，因为进程0的LDT中的limit 属性限制了进程0能够访问的地址空间。进程1拷贝了进程0的页表（160项），而这160个页表项即为内核第一个页表的前160项，指向的是物理内存前640KB=0.64MB，因此无法访问到16MB的顶端倒数的两个页。

进程0创建进程1的时候，先后通过get_free_page函数从物理地址中取出了两个页，但是并没有将这两个页的物理地址填入任何新的页表项中。此时只有内核的页表中包含了与这段物理地址对应的项，也就是说此时只有内核页表中有页表项指向这两个页的首地址，所以这两个页占用了内核线性空间。

  

3.

```Java
#define switch_to(n) {\
struct {long a,b;} __tmp; \
asm("cmpl %%ecx,_current\n\t" \
    "je 1f\n\t" \
    "movw %%dx,%1\n\t" \
    "xchgl %%ecx,_current\n\t" \
    "ljmp %0\n\t" \
    "cmpl %%ecx,_last_task_used_math\n\t" \
    "jne 1f\n\t" \
    "clts\n" \
    "1:" \
    ::"m" (*&__tmp.a),"m" (*&__tmp.b), \
    "d" (_TSS(n)),"c" ((long) task[n])); \
}
```

**代码中的"ljmp %0\n\t" 很奇怪，按理说jmp指令跳转到得位置应该是一条指令的地址，可是这行代码却跳到了"m" (*&__tmp.a)，这明明是一个数据的地址，更奇怪的，这行代码竟然能正确执行。请论述其中的道理。**

答：其中a对应EIP，b对应CS，ljmp此时通过CPU中的电路进行硬件切换，进程由当前进程切换到进程n。CPU将当前寄存器的值保存到当前进程的TSS中，将进程n的TSS数据及LDT的代码段和数据段描述符恢复给CPU的各个寄存器，实现任务切换。

![](https://calculet.feishu.cn/space/api/box/stream/download/asynccode/?code=MWQ4NzcyMTJmOGFjMDExYjYxZDJmOWRjNTNkN2U5OWNfbk1ybzVMdGZBVjB1NG9teXlVZVhxY0xZd3B3NXo5R2RfVG9rZW46UVJxTGJHc3NPb2dyc3p4UmpZRmNlTVNsbkdkXzE3ODE0MjM2OTk6MTc4MTQyNzI5OV9WNA&add_watermark=true&scene_type=CCM)

这行代码之所以能正确执行，其中的“道理”如下：

1. **数据即指针：** `ljmp` 操作的是内存数据，将其视为跳转指针，而非直接跳转到该内存地址。
    
2. **构造指针：** 代码在栈上临时构造了一个 `__tmp` 结构，将**目标进程的 TSS 选择子**填入了高位。
    
3. **触发机制：** 利用 x86 硬件特性，当 `ljmp` 遇到 TSS 选择子时，**不再是普通的跳转，而是变成了上下文切换**。
    
4. **忽略偏移：** 因为是切任务，目标入口点由 TSS 内部保存的 EIP 决定，所以 `__tmp.a`（指令中的偏移部分）虽然存在，但被硬件忽略，这就是为什么它即使没初始化也没关系。
    

  

  

**4.进程0开始创建进程1，调用fork（），跟踪代码时我们发现，fork代码执行了两次，第一次，执行fork代码后，跳过init（）直接执行了for(;;) pause()，第二次执行fork代码后，执行了init（）。奇怪的是，我们在代码中并没有看到向转向fork的goto语句，也没有看到循环语句，是什么原因导致fork反复执行？请说明理由（可以图示），并给出代码证据。**

- fork 为 inline 函数，其中调用了 sys_call0，产生 0x80 中断，将 ss, esp, eflags, cs, eip 压栈，其中 eip 为 int 0x80 的下一句的地址。在 copy_process 中，内核将进程 0 的 tss 复制得到进程 1 的 tss，并将进程 1 的 tss.eax 设为 0，而进程 0 中的 eax 为 1。在进程调度时 tss 中的值被恢复至相应寄存器中，包括 eip， eax 等。所以中断返回后，进程 0 和进程 1 均会从 int 0x80 的下一句开始执行，即 fork 执行了两次。
    

> copy_process：由于创建进程时新进程返 回值应为0，所以需要设置tss.eax = 0。新建进程内核态堆栈指针tss.esp0被设置成新进程任务数据结构 所在内存页面的顶端，而堆栈段tss.ss0被设置成内核数据段选择符。tss.ldt被设置为局部表描述符在GDT 中的索引值。如果当前进程使用了协处理器，把还需要把协处理器的完整状态保存到新进程的tss.i387 结构中。

- 由于 eax 代表返回值，所以进程 0 和进程 1 会得到不同的返回值，在fork返回到进程0后，进程0判断返回值非 0，因此执行代码for(;;) pause();
    

  

在sys_pause函数中，内核设置了进程0的状态为 TASK_INTERRUPTIBLE，并进行进程调度。由于只有进程1处于就绪态，因此调度执行进程1的指令。由于进程1在TSS中设置了eip等寄存器的值，因此从 int 0x80 的下一条指令开始执行，且设定返回 eax 的值作为 fork 的返回值（值为 0），因此进程1执行了 init 的 函数。导致反复执行，主要是利用了两个系统调用 sys_fork 和 sys_pause 对进程状态的设置，以及利用了进程调度机制。

代码如下：

```C++
//代码路径：init/main.c
void main(void)        {
        ......
        move_to_user_mode();
        if (!fork()) {//fork的返回值为1，if(!1)为假                /* we count on this going ok */
                init();//不会执行这一行
        }
//代码路径：include/unistd.h
int fork(void) \
{ \
long __res; \
__asm__ volatile ("int $0x80" \
        : "=a" (__res) \ //__res的值就是eax，是copy_process（）的返回值last_pid（1）
        : "0" (__NR_##name)); \
if (__res >= 0) \ //iret后，执行这一行！__res就是eax，值是1
        return (type) __res; \ //返回1！
errno = -__res; \
return -1; \
}
//代码路径：kernel/fork.c
int copy_process(int nr,long ebp,long edi,long esi,long gs,long none,
                long ebx,long ecx,long edx,
                long fs,long es,long ds,
                long eip,long cs,long eflags,long esp,long ss)
{
        struct task_struct *p;
        int i;
        struct file *f;

        p = (struct task_struct *) get_free_page();
        if (!p)
                return -EAGAIN;
        task[nr] = p;
        *p = *current;        /* NOTE! this doesn't copy the supervisor stack */
        p->state = TASK_UNINTERRUPTIBLE;
        p->pid = last_pid;
        p->father = current->pid;
        p->counter = p->priority;
        p->signal = 0;
        p->alarm = 0;
        p->leader = 0;                /* process leadership doesn't inherit */
        p->utime = p->stime = 0;
        p->cutime = p->cstime = 0;
        p->start_time = jiffies;
        p->tss.back_link = 0;
        p->tss.esp0 = PAGE_SIZE + (long) p;
        p->tss.ss0 = 0x10;
        p->tss.eip = eip;
        p->tss.eflags = eflags;
        p->tss.eax = 0;
        p->tss.ecx = ecx;
        p->tss.edx = edx;
        p->tss.ebx = ebx;
        p->tss.esp = esp;
        p->tss.ebp = ebp;
        p->tss.esi = esi;
        p->tss.edi = edi;
        p->tss.es = es & 0xffff;
        p->tss.cs = cs & 0xffff;
        p->tss.ss = ss & 0xffff;
        p->tss.ds = ds & 0xffff;
        p->tss.fs = fs & 0xffff;
        p->tss.gs = gs & 0xffff;
        p->tss.ldt = _LDT(nr);
        p->tss.trace_bitmap = 0x80000000;
        if (last_task_used_math == current)
                __asm__("clts ; fnsave %0"::"m" (p->tss.i387));
        if (copy_mem(nr,p)) {
                task[nr] = NULL;
                free_page((long) p);
                return -EAGAIN;
        }
        for (i=0; i<NR_OPEN;i++)
                if (f=p->filp[i])
                        f->f_count++;
        if (current->pwd)
                current->pwd->i_count++;
        if (current->root)
                current->root->i_count++;
        if (current->executable)
                current->executable->i_count++;
        set_tss_desc(gdt+(nr<<1)+FIRST_TSS_ENTRY,&(p->tss));
        set_ldt_desc(gdt+(nr<<1)+FIRST_LDT_ENTRY,&(p->ldt));
        p->state = TASK_RUNNING;        /* do this last, just in case */
        return last_pid;
}
```

  

**5、打开保护模式、分页后，线性地址到物理地址是如何转换的？**

![](https://calculet.feishu.cn/space/api/box/stream/download/asynccode/?code=ZjhjOWMxYjdiNmYyYTdiYTA1ZGRlYjE5ZjMxNTMxZmNfSHVhNTlCZzVyVHRKTTFXV1FXR21jMFR3eUs2ZmVJcXFfVG9rZW46VnFTcGIyWUc0bzh5WW54dXZ1V2NJeFZhbk5kXzE3ODE0MjM2OTk6MTc4MTQyNzI5OV9WNA&add_watermark=true&scene_type=CCM)

  

**6、getblk函数中，申请空闲缓冲块的标准就是b_count为0，而申请到之后，为什么在wait_on_buffer(bh)后又执行if（bh->b_count）来判断b_count是否为0？**

![](https://calculet.feishu.cn/space/api/box/stream/download/asynccode/?code=Y2M5MjhkOGMxNGQyNjU4MGI1ZmM3Y2QzNDk2NjEzY2NfME9uV1hOZ0VRZld5dExUWmN2OHgwaG9rTFBnUXQ2dkpfVG9rZW46TDd2UWJFMjlrb2hiNXF4eG5aMGNzMG05bjRjXzE3ODE0MjM2OTk6MTc4MTQyNzI5OV9WNA&add_watermark=true&scene_type=CCM)

P114

wait_on_buffer(bh)内包含睡眠函数，虽然此时已经找到比较合适的空闲缓冲块，但是可能在睡眠阶段该缓冲区被其他任务所占用，因此必须重新搜索，判断是否被修改，修改则写盘等待解锁。判断若被占用则重新repeat，继续执行if（bh->b_count）

```C++
//代码路径：fs/blk_dev.c:
int block_write(int dev, long * pos, char * buf, int count) //块设备文件内容写入缓冲块
{
      …
          offset= 0;
          *pos += chars;
          written += chars;
          count -= chars;
          while (chars-->0)
                *(p + +)= get_fs_byte(buf + +);
          bh->b_dirt= 1;
              brelse(bh);
      …
}
//代码路径：fs/file_dev.c:
int file_write(struct m_inode * inode, struct file * filp, char * buf, int count)
                                                          //普通文件内容写入缓冲块
{
      …
          c= pos % BLOCK_SIZE;
          p= c + bh->b_data;
          bh->b_dirt= 1;
          c= BLOCK_SIZE-c;
          if (c > count-i) c= count-i;
          pos += c;
          if (pos > inode->i_size) {
                inode->i_size= pos;
                inode->i_dirt= 1;
          }
          i += c;
          while (c-->0)
                *(p + +)= get_fs_byte(buf + +);
      …
}
//代码路径：fs/file_dev.c:
static struct buffer_head * add_entry(struct m_inode * dir,
      const char * name, int namelen, struct dir_entry ** res_dir)//目录文件需要加载
                                                                  //目录项，用到写缓冲块
```

  

**7、b_dirt已经被置为1的缓冲块，同步前能够被进程继续读、写？给出代码证据。**

![](https://calculet.feishu.cn/space/api/box/stream/download/asynccode/?code=ZDZjOWY2NGI1NGUwNzQ5MzE0OGJiZmQ1ZmMzOWFhNDFfODBEVmxCdzFyd0hibWFlNlk5MDQ1RGxxRFRlSUpxWlZfVG9rZW46QjhhNGJPUW5qbzUzQXV4bE1QQmNVZ3p6bkd6XzE3ODE0MjM2OTk6MTc4MTQyNzI5OV9WNA&add_watermark=true&scene_type=CCM)

uptodate表示数据有效（类似valid，OS中buffer需要从硬盘中读到有效数据才能置为1），dirt表示修改过

- 同步前能够被进程继续读、写
    
- b_uptodate设置为1后，内核就可以支持进程共享该缓冲块的数据了，读写都可以，读操作不会改变缓冲块的内容，所以不影响数据，而执行写操作后，就改变了缓冲块的内容，就要将b_dirt标志设置为1。由于此前缓冲块中的数据已经用硬盘数据块更新了，所以后续的同步未被改写的部分不受影响，同步是不更改缓冲块中数据的，所以b_uptodate仍为1。即进程在b_dirt置为1时，仍能对缓冲区数据进行读写。
    

```C++
//代码路径：fs/blk_dev.c:
int block_write(int dev, long * pos, char * buf, int count) //块设备文件内容写入缓冲块
{
      …
          offset= 0;
          *pos += chars;
          written += chars;
          count -= chars;
          while (chars-->0)
                *(p + +)= get_fs_byte(buf + +);
          bh->b_dirt= 1;
              brelse(bh);
      …
}
//代码路径：fs/file_dev.c:
int file_write(struct m_inode * inode, struct file * filp, char * buf, int count)
                                                          //普通文件内容写入缓冲块
{
      …
          c= pos % BLOCK_SIZE;
          p= c + bh->b_data;
          bh->b_dirt= 1;
          c= BLOCK_SIZE-c;
          if (c > count-i) c= count-i;
          pos += c;
          if (pos > inode->i_size) {
                inode->i_size= pos;
                inode->i_dirt= 1;
          }
          i += c;
          while (c-->0)
                *(p + +)= get_fs_byte(buf + +);
      …
}

struct buffer_head * bread(int dev,int block)
{
    struct buffer_head * bh;

    // 在高速缓冲区中申请一块缓冲块。如果返回值是NULL，则表示内核出错，停机。
    // 然后我们判断其中说是否已有可用数据。如果该缓冲块中数据是有效的（已更新）
    // 可以直接使用，则返回。
    if (!(bh=getblk(dev,block)))
        panic("bread: getblk returned NULL\n");
    if (bh->b_uptodate)
        return bh;
    ...
}
```

  

**8、分析panic函数的****源代码****，根据你学过的操作系统知识，完整、准确的判断panic函数所起的作用。假如操作系统设计为支持内核进程（始终运行在0特权级的进程），你将如何改进panic函数？**

panic()函数是当系统发现无法继续运行下去的故障时将调用它，会导致程序终止，然后由系统显示错误号。如果出现错误的函数不是进程0，那么就要进行数据同步，把缓冲区中的数据尽量同步到硬盘上。遵循了Linux尽量简明的原则。

改进panic函数：将死循环for(;;) 改进为跳转到内核进程（始终运行在0特权级的进程），让内核继续执行。

原本的panic函数

```C++
kernel/panic.c
#include <linux/kernel.h>
#include <linux/sched.h>
void sys_sync(void);
volatile void panic(const char * s)
{
         printk("Kernel panic: %s\n\r",s);
         if (current == task[0])
                   printk("In swapper task - not syncing\n\r");
         else
                   sys_sync();
         for(;;);
}
```

  

**9、详细分析进程调度的全过程。考虑所有可能（signal、alarm除外）**

1. 进程中有就绪进程，且时间片没有用完。

正常情况下，schedule()函数首先扫描任务数组。通过比较每个就绪（TASK_RUNNING）任务的运行时间递减滴答计数counter 的值来确定当前哪个进程运行的时间最少。哪一个的值大，就表示运行时间还不长，于是就选中该进程，最后调用switch_to()执行实际的进程切换操作

2. 进程中有就绪进程，但所有就绪进程时间片都用完（c=0）

如果此时所有处于TASK_RUNNING 状态进程的时间片都已经用完，系统就会根据每个进程的优先权值priority，对系统中所有进程（包括正在睡眠的进程）重新计算每个任务需要运行的时间片值counter。计算的公式是：

counter = counter + priority/2

然后 schdeule()函数重新扫描任务数组中所有处于TASK_RUNNING 状态，重复上述过程，直到选择出一个进程为止。最后调用switch_to()执行实际的进程切换操作。

3. 所有进程都不是就绪的c=-1

此时代码中的c=-1，next=0，跳出循环后，执行switch_to(0)，切换到进程0执行，因此所有进程都不是就绪的时候进程0执行。

  

**10、wait_on_buffer函数中为什么不用if（）而是用while（）？**

答：因为可能存在一种情况是，很多进程都在等待一个缓冲块。在缓冲块同步完毕，唤醒各等待进程到轮转到某一进程的过程中，很有可能此时的缓冲块又被其它进程所占用，并被加上了锁。此时如果用if()，则此进程会从之前被挂起的地方继续执行，不会再判断是否缓冲块已被占用而直接使用，就会出现错误；而如果用while()，则此进程会再次确认缓冲块是否已被占用，在确认未被占用后，才会使用，这样就不会发生之前那样的错误。

  

**11、操作系统如何利用b_uptodate保证缓冲块数据的正确性？new_block (int dev)函数新申请一个缓冲块后，并没有读盘，b_uptodate却被置1，是否会引起数据混乱？详细分析理由。**

答：b_uptodate是缓冲块中针对进程方向的标志位，它的作用是告诉内核，缓冲块的数据是否已是数据块中最新的。当b_update置1时，就说明缓冲块中的数据是基于硬盘数据块的，内核可以放心地支持进程与缓冲块进行数据交互；如果b_uptodate为0，就提醒内核缓冲块并没有用绑定的数据块中的数据更新，不支持进程共享该缓冲块。

当为文件创建新数据块，新建一个缓冲块时，b_uptodate被置1，但并不会引起数据混乱。此时，新建的数据块只可能有两个用途，一个是存储文件内容，一个是存储文件的i_zone的间接块管理信息。

- 如果是存储文件内容，由于新建数据块和新建硬盘数据块，此时都是垃圾数据，都不是硬盘所需要的，无所谓数据是否更新，结果“等效于”更新问题已经解决。
    
- 如果是存储文件的间接块管理信息，必须清零，表示没有索引间接数据块，否则垃圾数据会导致索引错误，破坏文件操作的正确性。虽然缓冲块与硬盘数据块的数据不一致，但同样将b_uptodate置1不会有问题。
    

综合以上考虑，设计者采用的策略是，只要为新建的数据块新申请了缓冲块，不管这个缓冲块将来用作什么，反正进程现在不需要里面的数据，干脆全部清零。这样不管与之绑定的数据块用来存储什么信息，都无所谓，将该缓冲块的b_uptodate字段设置为1，更新问题“等效于”已解决

  

**12、add_reques（）函数中有下列代码**

```C
    if (!(tmp = dev->current_request)) {
        dev->current_request = req;
        sti();
        (dev->request_fn)();
        return;
    }
```

**其中的**

```C
    if (!(tmp = dev->current_request)) {
        dev->current_request = req;
```

**是什么意思？**

检查设备是否正忙，若目前该设备没有请求项，本次是唯一一个请求，之前无链表，则将该设备当前请求项指针直接指向该请求项，作为链表的表头。

  

**13、do_hd_request()函数中dev的含义始终一样吗？**

122 页 不一样。

答： 不是一样的。 dev/=5 之前表示当前硬盘的逻辑盘号。 这行代码之后表示的实际的物理设备号。

Linux 0.11 规定：每块物理硬盘最多支持 5 个逻辑设备（1 个代表整盘 + 4 个分区）。

  

**14、read_intr（）函数中，下列代码是什么意思？为什么这样做？**

```C
    if (--CURRENT->nr_sectors) {
        do_hd = &read_intr;
        return;
    }
```

答案：参照P131

nr_sectors代表总的要读取的扇区数，如果没有读取完成，内核会再次把read_intr()绑定在硬盘中断服务程序上，以待下次使用，读取一个扇区的数据之后继续进入read_intr()，其实类似递归

  

**15、bread（）函数代码中为什么要做第二次if (bh->b_uptodate)判断？**

```C
    if (bh->b_uptodate)
        return bh;
    ll_rw_block(READ,bh);
    wait_on_buffer(bh);
    if (bh->b_uptodate)
        return bh;
```

![](https://calculet.feishu.cn/space/api/box/stream/download/asynccode/?code=ZTEwOWNmZWY4MjY4ODAyMDY1NWFiZDY4MDc4NTc3NmJfdGhkbzU4WlQ4NGJYY2QwMmNDSFdQcWp4M3N4WHZ1SHFfVG9rZW46VFNhSGJDZmRHb29VRG54MzgzZWM5Znp3bmxkXzE3ODE0MjM2OTk6MTc4MTQyNzI5OV9WNA&add_watermark=true&scene_type=CCM)

第一次从高速缓冲区中取出指定和设备和块号相符的缓冲块， 判断缓冲块数据是否有效， 有效则返回此块， 正当用。 如果该缓冲块数据无效（更新标志未置位） ， 则发出读设备数据块请求。会进行下面第二次判断

第二次，等指定数据块被读入，并且缓冲区解锁，睡眠醒来之后，要重新判断缓冲块是否有效，如果缓冲区中数据有效，则返回缓冲区头指针退出。否则释放该缓冲区返回 NULL,退出。这是因为在等待过程中，数据可能已经发生了改变，所以要第二次判断。

  

**16、getblk（）函数中，两次调用wait_on_buffer（）函数，两次的意思一样吗？**

代码在书上113和114

答： 一样。 都是等待缓冲块解锁。

第一次调用是在， 已经找到一个比较合适的空闲缓冲块， 但是此块可能是加锁的， 于是等待该缓冲块解锁。

第二次调用， 是找到一个缓冲块， 但是此块被修改过， 即是脏的， 还有其他进程在写或此块等待把数据同步到硬盘上， 写完要加锁， 所以此处的调用仍然是等待缓冲块解锁。

  

**17、getblk（）函数中**

```Java
    do {
        if (tmp->b_count)
            continue;
        if (!bh || BADNESS(tmp)<BADNESS(bh)) {
            bh = tmp;
            if (!BADNESS(tmp))
                break;
        }
/* and repeat until we find something good */
    } while ((tmp = tmp->b_next_free) != free_list);
```

**说明什么情况下执行continue、break。**

![](https://calculet.feishu.cn/space/api/box/stream/download/asynccode/?code=ZDg4Nzg1ZTliYmMwMTE2MjJhMDgzOTQ4MDA0Yzg1MzBfS3d2ODAxYzZ1NzdPeVhQMUhSR3ZHQTRlQWRoaUFGZjRfVG9rZW46UVB4U2JNZDVCb1U4bVF4cDdqdmNSbkpobmtmXzE3ODE0MjM2OTk6MTc4MTQyNzI5OV9WNA&add_watermark=true&scene_type=CCM)

Continue：if (tmp->b_count)在判断缓冲块的引用计数，如果引用计数不为0，那么继续判断空闲队列中下一个缓冲块（即continue），直到遍历完。

Break：如果有引用计数为0的块，那么判断空闲队列中那些引用计数为0 的块的badness，找到一个最小的，如果在寻找的过程中出现badness为0的块，那么就跳出循环（即break）。

如果利用函数get_hash_table找到了能对应上设备号和块号的缓冲块，那么直接返回。

如果找不到，那么就分为三种情况：

1.所有的缓冲块b_count=0，缓冲块是新的。

2.虽然有b_count=0，但是有数据脏了，未同步或者数据脏了正在同步加和既不脏又不加锁三种情况；

3.所有的缓冲块都被引用，此时b_count非0，即为所有的缓冲块都被占用了。

综合以上三点可知，如果缓冲块的b_count非0，则continue继续查找，知道找到b_count=0的缓冲块；如果获取空闲的缓冲块，而且既不加锁又不脏，此时break，停止查找。

  

**18、make_request（）函数**

```C
    if (req < request) {
        if (rw_ahead) {
            unlock_buffer(bh);
            return;
        }
        sleep_on(&wait_for_request);
        goto repeat;
```

**其中的sleep_on(&wait_for_request)是谁在等？等什么？**

这行代码是当前进程在等（如：进程1），在等空闲请求项。

make_request()函数创建请求项并插入请求队列，执行if的内容说明没有找到空请求项：如果是超前的读写请求，因为是特殊情况则放弃请求直接释放缓冲区，否则是一般的读写操作，此时等待直到有空闲请求项，然后从repeat开始重新查看是否有空闲的请求项。

  

  

# 额外函数

文件系统拿出一段代码来分析（写注释 讲原理）

- mount_root
    
    ![](https://calculet.feishu.cn/space/api/box/stream/download/asynccode/?code=MDc4Y2RlZTcyY2ZjMmQxNDYyMWRmN2VhYzk0NTgyOTBfSE9mZlI5TDNDWUdKMlhsamp2OTROUlpIdlpIZHhPS2hfVG9rZW46UG1aR2JKRXlkb2JrcU94NjdXSmN0M0hzbnRkXzE3ODE0MjM2OTk6MTc4MTQyNzI5OV9WNA&add_watermark=true&scene_type=CCM)
    
      
    
    ![](https://calculet.feishu.cn/space/api/box/stream/download/asynccode/?code=NjRjOTBhMjY5NWQ1MTUwMzdiNTYyN2JjZjNjYzJlNDBfRUExNXl0MFE0YkdwSnVXOFBPb2RGdmRzd2Zva2ZZU29fVG9rZW46V1d3VGI5RW9Vb09vbEF4RWJSb2MwRWU5bllnXzE3ODE0MjM2OTk6MTc4MTQyNzI5OV9WNA&add_watermark=true&scene_type=CCM)
    
    ![](https://calculet.feishu.cn/space/api/box/stream/download/asynccode/?code=YWNhYzhjYTI4OTVhOGM5NzM2MDg5MTA4OTU1MDJiZjdfZFpRbThvR1E1aWNvTnk3akI5MDNhZmM4Q0VEVkI3ZFNfVG9rZW46VDNJS2JlaGhmb0Q3UlV4VWxHVGNMYTJ3bmNkXzE3ODE0MjM2OTk6MTc4MTQyNzI5OV9WNA&add_watermark=true&scene_type=CCM)
    
    - 加载根文件系统的标志性动作： 将inode_table[32]中代表虚拟盘根i节点的 项挂接到super_block[8]中代表根设备虚拟盘的 项中的s_isup、s_imount指针上。这样，操作系 统在根设备上可以通过这里建立的关系，一步步 地把文件找到。
        
- sys_mount：往一个目录名上加载一个文件系统，经过很多判断，挂载实际上就是 s_imount = dir_i，然后可以通过
    
    ![](https://calculet.feishu.cn/space/api/box/stream/download/asynccode/?code=ZTMyMDgxYTA3ZjkzYjQ2NDY5NjI4YTQwN2JiZDY2MmFfbzRScWhncWtXak9JOG5QanhXaVFIY1UyUlBtVDI5VjJfVG9rZW46UUFLQWI3bXNlb25zY0J4NXNDdGNZdzFObk9oXzE3ODE0MjM2OTk6MTc4MTQyNzI5OV9WNA&add_watermark=true&scene_type=CCM)
    
    ![](https://calculet.feishu.cn/space/api/box/stream/download/asynccode/?code=ZDdhOTNmZjUwNDllOTJjMzM3YWQ3NjU0MDI5YWFiYjlfM1NlNVM0anUybEJaWDdPWTRKU1NubVY2SGc2Vm5rdTZfVG9rZW46WFEyV2JDOGJEb0hzQ1N4REM3MmNDa3l6bkFFXzE3ODE0MjM2OTk6MTc4MTQyNzI5OV9WNA&add_watermark=true&scene_type=CCM)
    
- sys_open：首先在自己进程的filp找到一个空闲的，然后在全局filr_table找到一个空闲的，把他俩连起来，然后openi写到这个空闲的
    
- sys_close
    
- sys_read
    
- sys_write
    

  

mount_root

```SQL
void mount_root(void)
{
    int i,free;
    struct super_block * p;
    struct m_inode * mi;

    if (32 != sizeof (struct d_inode))
        panic("bad i-node size");
    for(i=0;i<NR_FILE;i++)
        file_table[i].f_count=0;
    if (MAJOR(ROOT_DEV) == 2) {
        printk("Insert root floppy and press ENTER");
        wait_for_keypress();
    }
    for(p = &super_block[0] ; p < &super_block[NR_SUPER] ; p++) {
        p->s_dev = 0;
        p->s_lock = 0;
        p->s_wait = NULL;
    }
    if (!(p=read_super(ROOT_DEV)))
        panic("Unable to mount root");
    if (!(mi=iget(ROOT_DEV,ROOT_INO)))
        panic("Unable to read root i-node");
    mi->i_count += 3 ;  /* NOTE! it is logically used 4 times, not 1 */
    p->s_isup = p->s_imount = mi;
    current->pwd = mi;
    current->root = mi;
    free=0;
    i=p->s_nzones;
    while (-- i >= 0)
        if (!set_bit(i&8191,p->s_zmap[i>>13]->b_data))
            free++;
    printk("%d/%d free blocks\n\r",free,p->s_nzones);
    free=0;
    i=p->s_ninodes+1;
    while (-- i >= 0)
        if (!set_bit(i&8191,p->s_imap[i>>13]->b_data))
            free++;
    printk("%d/%d free inodes\n\r",free,p->s_ninodes);
}
```

sys_mount

```C++
int sys_mount(char * dev_name, char * dir_name, int rw_flag)
{
    struct m_inode * dev_i, * dir_i;
    struct super_block * sb;
    int dev;

    if (!(dev_i=namei(dev_name)))
        return -ENOENT;
    dev = dev_i->i_zone[0];
    if (!S_ISBLK(dev_i->i_mode)) {
        iput(dev_i);
        return -EPERM;
    }
    iput(dev_i);
    if (!(dir_i=namei(dir_name)))
        return -ENOENT;
    if (dir_i->i_count != 1 || dir_i->i_num == ROOT_INO) {
        iput(dir_i);
        return -EBUSY;
    }
    if (!S_ISDIR(dir_i->i_mode)) {
        iput(dir_i);
        return -EPERM;
    }
    if (!(sb=read_super(dev))) {
        iput(dir_i);
        return -EBUSY;
    }
    if (sb->s_imount) {
        iput(dir_i);
        return -EBUSY;
    }
    if (dir_i->i_mount) {
        iput(dir_i);
        return -EPERM;
    }
    sb->s_imount=dir_i;
    dir_i->i_mount=1;
    dir_i->i_dirt=1;        /* NOTE! we don't iput(dir_i) */
    return 0;           /* we do that in umount */
}
```

sys_open

```SQL
int sys_open(const char * filename,int flag,int mode)
{
    struct m_inode * inode;
    struct file * f;
    int i,fd;

    mode &= 0777 & ~current->umask;
    for(fd=0 ; fd<NR_OPEN ; fd++)
        if (!current->filp[fd])
            break;
    if (fd>=NR_OPEN)
        return -EINVAL;
    current->close_on_exec &= ~(1<<fd);
    f=0+file_table;
    for (i=0 ; i<NR_FILE ; i++,f++)
        if (!f->f_count) break;
    if (i>=NR_FILE)
        return -EINVAL;
    (current->filp[fd]=f)->f_count++;
    if ((i=open_namei(filename,flag,mode,&inode))<0) {
        current->filp[fd]=NULL;
        f->f_count=0;
        return i;
    }
/* ttys are somewhat special (ttyxx major==4, tty major==5) */
    if (S_ISCHR(inode->i_mode))
        if (MAJOR(inode->i_zone[0])==4) {
            if (current->leader && current->tty<0) {
                current->tty = MINOR(inode->i_zone[0]);
                tty_table[current->tty].pgrp = current->pgrp;
            }
        } else if (MAJOR(inode->i_zone[0])==5)
            if (current->tty<0) {
                iput(inode);
                current->filp[fd]=NULL;
                f->f_count=0;
                return -EPERM;
            }
/* Likewise with block-devices: check for floppy_change */
    if (S_ISBLK(inode->i_mode))
        check_disk_change(inode->i_zone[0]);
    f->f_mode = inode->i_mode;
    f->f_flags = flag;
    f->f_count = 1;
    f->f_inode = inode;
    f->f_pos = 0;
    return (fd);
}
```

sys_close

```SQL
int sys_close(unsigned int fd)
{   
    struct file * filp;

    if (fd >= NR_OPEN)
        return -EINVAL;
    current->close_on_exec &= ~(1<<fd);
    if (!(filp = current->filp[fd]))
        return -EINVAL;
    current->filp[fd] = NULL;
    if (filp->f_count == 0)
        panic("Close: file count is 0");
    if (--filp->f_count)
        return (0);
    iput(filp->f_inode);
    return (0);
}
```

sys_read

```C++
int sys_read(unsigned int fd,char * buf,int count)
{
    struct file * file;
    struct m_inode * inode;

    if (fd>=NR_OPEN || count<0 || !(file=current->filp[fd]))
        return -EINVAL;
    if (!count)
        return 0;
    verify_area(buf,count);
    inode = file->f_inode;
    if (inode->i_pipe)
        return (file->f_mode&1)?read_pipe(inode,buf,count):-EIO;
    if (S_ISCHR(inode->i_mode))
        return rw_char(READ,inode->i_zone[0],buf,count,&file->f_pos);
    if (S_ISBLK(inode->i_mode))
        return block_read(inode->i_zone[0],&file->f_pos,buf,count);
    if (S_ISDIR(inode->i_mode) || S_ISREG(inode->i_mode)) {
        if (count+file->f_pos > inode->i_size)
            count = inode->i_size - file->f_pos;
        if (count<=0)
            return 0;
        return file_read(inode,file,buf,count);
    }
    printk("(Read)inode->i_mode=%06o\n\r",inode->i_mode);
    return -EINVAL;
}
```

sys_write

```C++
int sys_write(unsigned int fd,char * buf,int count)
{
    struct file * file;
    struct m_inode * inode;
    
    if (fd>=NR_OPEN || count <0 || !(file=current->filp[fd]))
        return -EINVAL;
    if (!count)
        return 0;
    inode=file->f_inode;
    if (inode->i_pipe)
        return (file->f_mode&2)?write_pipe(inode,buf,count):-EIO;
    if (S_ISCHR(inode->i_mode))
        return rw_char(WRITE,inode->i_zone[0],buf,count,&file->f_pos);
    if (S_ISBLK(inode->i_mode))
        return block_write(inode->i_zone[0],&file->f_pos,buf,count);
    if (S_ISREG(inode->i_mode))
        return file_write(inode,file,buf,count);
    printk("(Write)inode->i_mode=%06o\n\r",inode->i_mode);
    return -EINVAL;
}
```

  

  

记录一下25年真题，一共七道题目

1. 第一题是啥忘了，可能是bootsect、setup、head程序之间是怎么衔接的？
    
2. Jumpi 0,8
    
3. copy_process的最后五个参数从哪里来的
    
4. fork init
    
5. 进程调度，switch_to
    
6. make_request和add_request
    
7. sys_mount
    

  

参考资料：

1. https://www.cnblogs.com/lolo9261/p/17136400.html
    
2. https://blog.csdn.net/weixin_46160781/article/details/135143520
    
3. https://blog.csdn.net/weixin_46160781/article/details/135170568?spm=1001.2014.3001.5502
    
4. https://github.com/LeoJhonSong/linux-0.11-Note
