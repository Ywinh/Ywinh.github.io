---
date: 2026-06-28
categories:
  - Software
tags:
  - Linker
  - ELF
---

# Link script



https://mcyoung\.xyz/2021/06/01/linker\-script/\#appendix

\.S只是目标文件的文本表示

ar：本质上是一个古老的tar，将多个\.o文件合并成一个库，可以看作是\.o文件的集合。其后缀是 \.a



# 工具链使用

objdump

- objdump \-x xxx\.o：显示目标文件的所有header信息

- Objdump \-h xxx\.o：显示目标文件的section列表

readelf

objcopy

- 可以充当一个简单loader的作用，适用于那些没有任何load操作的小型控制器，后文会详细介绍



# linker

linker的输入是多个\.o对象文件或者\.a，输出是可执行文件，具体分为多个步骤

1. 解析所有对象和静态库，并将它们的符号存入数据库。符号是函数和全局变量的命名地址。

2. 在 `.o` 文件中搜索所有未解析的符号引用并进行匹配。 使用步骤1数据库中的符号，递归地对任何`.a`中被引用的代码执行此操作。这会在各个部分之间形成一种依赖关系图。这一步骤称为*符号解析 *。

3. 通过追踪入口点符号（例如，Linux 上的 `_start` ）的依赖关系图，丢弃所有未被输入文件引用的代码。这一步骤称为*垃圾回收 *。

4. 执行链接器脚本link script以确定如何将最终二进制文件拼接在一起。包括确定所有文件的偏移量。

5. 解决*重定位问题 *，即二进制文件中需要知道段最终运行时地址的“空洞”。重定位是放置在目标文件中供链接器执行的指令。

6. 写出完整的二进制代码



Link script只关心第四步，如果想了解其他的步骤可以查看：https://lwn\.net/Articles/276782/



## section属性

Objdump \-x xxx\.o 可以查看所有的header信息

目标文件描述了程序如何加载到内存中。目标文件被划分为多个section，这些section被称为data blocks。section拥有类似文件的权限比如像是r，w，x。可以使用objdump \-h显示section的列表。以下是一个例子，删去了前导0

```C++
$ objdump -h "$(which clang)"
/usr/bin/clang:     file format elf64-x86-64

Sections:
Idx Name    Size      VMA       LMA       File off  Algn
 11 .init   00000017  00691ab8  00691ab8  00291ab8  2**2
            CONTENTS, ALLOC, LOAD, READONLY, CODE
 12 .plt    00006bb0  00691ad0  00691ad0  00291ad0  2**4
            CONTENTS, ALLOC, LOAD, READONLY, CODE
 13 .text   0165e861  00698680  00698680  00298680  2**4
            CONTENTS, ALLOC, LOAD, READONLY, CODE
 14 .fini   00000009  01cf6ee4  01cf6ee4  018f6ee4  2**2
            CONTENTS, ALLOC, LOAD, READONLY, CODE
 15 .rodata 0018ec68  01cf6ef0  01cf6ef0  018f6ef0  2**4
            CONTENTS, ALLOC, LOAD, READONLY, DATA
 24 .data   000024e8  021cd5d0  021cd5d0  01dcc5d0  2**4
            CONTENTS, ALLOC, LOAD, DATA
 26 .bss    00009d21  021cfac0  021cfac0  01dceab8  2**4
            ALLOC
```

- `ALLOC`：代表必须由操作系统分配空间，linker预留了这一片内存

- `LOAD`：loadable代表OS必须在之后用section的内容填充这块空间，该填充过程被称为load，由加载器loader来做

loader有时被称为动态链接器，并且通常与程序链接器是一个东西，这也是为什么linker被称为ld的原因

loading需要大量内存，对于一些很小的控制器来说可以预先使用binary进行loading，objcopy可以在该过程使用以及其他转换\.o文件的任务



对于每个section来说，如果不指定memory region，那么每个section会有**默认的属性**。一般来说都是alloc\+load，但是bss只是alloc，不load，具体情况可通过`objdump -h`分析

## **一些常见的section**

1. `.text`：代码放在这里，通常是loadable，readonly，executeable

2. `.data`：包含全局变量的初始值，通常是loadable

3. `.rodata`：readonly data，是常量；loadable，readonly

4. `.bss`：空的可分配段，一般存储未初始化的全局变量。因为全局变量如果未初始化默认为0，所以这可以避免存储大量无意义的0，节省空间。allocable

5. 然后还有一些debug section，他们通常是不可load，不可分配。实际使用时不需要，调试时需要



linker决定了会保留\.o和\.a中的哪些部分（根据其需要），它会询问link script如何在输出中排布这些需要的部分

让我们来看第一个linker script吧！

```JSON
SECTIONS {  
    */* Define an output section ".text". */*  
    .text : {    
        */* Pull in all symbols in input sections named .text */*    
        *(.text)    
        */* Do the same for sections starting with .text., such as .text.foo */*    
         *(.text.*)  
    }
    */* Do the same for ".bss", ".rodata", and ".data". */*  
    .bss : { *(.bss); *(.bss.*) }  
    .data : { *(.data); *(.data.*) }  
    .rodata : { *(.rodata); *(.rodata.*) }
  }
```

上面的脚本创建了一个\.text段，其内部包含了所有输入文件中 \.text 和 \.text\.\*（比如\.text\.foo） 的段。所有的\.text段位于\.text\.\*之前，而具体到\*\(\.text\)内部的顺序则没有保证

需要注意上述脚本中的两个\.text是不同的，他们可以有不同的名称，linker只关心它的属性是什么

每个section都有一个对齐方式，比如下面的ALIGN\(16\)表示按照16字节对齐

```C++
SECTIONS {  
    .super_aligned : ALIGN(16) {    
    */* ... */*  
    }
}
```

可以通过`/DISCARD/`指示linker丢弃某些部分，可以用于丢弃gcc想保留的调试信息

也可以使用`KEEP(*(.text.*))` 来确保不会有 `.text` 段被垃圾回收丢弃



## **LMA和VMA**

每个section都关联了三个地址，分别是offset，VMA，LMA

offset：从文件开头到该节的距离

VMA：**运行地址**（或者叫链接地址），程序在运行时期望从这个位置找到内存段。程序中的指针和pc都使用这个地址

LMA：**加载地址**（或者叫存储地址），加载器（无论是运行时加载器还是 `objcpy` ）必须放置代码的位置。几乎总是与 VMA 相同

一个是运行时地址，一个是存储地址，其含义在后文的rom和ram中会体现得很明显



当在link scipt中声明一个新的section时，VMA和LMA都会设置为当前 location counter的值，也就是`.`这个符号代表的值，它会自动递增，比如从输入的\.o文件中复制了一些新的data进来，该符号会自动加对应的大小

我们可以通过在冒号前添加表达式来显式指定节的 VMA，通过 `AT(lma)` 说明符来显式指定 LMA。

```OpenGL Shading Language
SECTIONS {  
    .text 0x10008000: AT(0x40008000) {   
     */* ... */*  
     }
}
```

上面这种写法会修改`.`的值，等价于

```C++
SECTIONS {  
    . = 0x10008000;  
    .text : AT(0x40008000) {    
        */* ... */*  
    }
}
```

在SECTIONS中，可以在任何位置设置location counter，但是很少直接操作它，它会根据section的添加自动递增



## **内存区域和section分配**

一般来说，linker会从地址0开始分配内存段。可以使用`MEMORY`语句定义内存区域，以便更精细地控制 VMA 和 LMA 的分配方式，而无需显式地写入这些区域。

使用`MEMORY`的一个经典例子，把地址空间分为ROM和RAM

```C++
MEMORY {  
    rom (rx)   : ORIGIN = 0x8000,     LENGTH = 16K  
    ram (rw!x) : ORIGIN = 0x10000000, LENGTH = 256M
}
```

一块内存区域其实就是有一个name和一个具备rwx等属性的块

rw\!x表示允许rw，不允许x，即感叹号之前的属性允许，之后的属性不允许

所有的属性：rwxal

- rwx都很熟悉，a指的alloc已分配，l指的是load可加载



分配完内存区域后，还要对其对应的section做一个对应，以便section能够真正地位于这块memory region，主要是要有期望的属性。linker可以自动匹配，但是作者说他不太信任，最稳妥的做法是使用`> region`来把数据放入特定的区域

```C++
SECTION {  
    .data {    
        */* ... */*  
    } > ram AT> rom
}
```

上面表示data section，VMA = ram， LMA = rom。为什么要这样做？

1. **烧录时**：程序被烧录到 Flash \(ROM\) 中。`.data` 段的数据（例如变量的初始值 `10`）安静地躺在 Flash \(`AT> rom` 指定的位置\) 里。

2. **上电时**：

    - CPU 开始执行代码。此时，RAM 里全是随机值或零，那个 `10` 还在 Flash 里。

    - 如果程序直接去读 RAM 里的变量地址（`> ram` 指定的位置），读不到正确的初始值。=

3. **启动代码**：

    - 在进入 `main()` 函数之前，启动文件（通常是 `startup.s` 或 `crt0.s`）会执行一段**数据搬运**代码。

    - 它会把 `.data` 段的内容，从 **Flash \(LMA\)** 复制到 **RAM \(VMA\)**。

    - 复制完成后，程序才能在 RAM 中读写这些已初始化的变量。

简单来说`AT> rom`告知从哪里找到这段data，`>ram`告知运行时这段data暂存在哪里



**其他可以放在section中的内容**

- 可以使用BYTE、 `SHORT` 、LONG 和 QUAD等将任意类型的数据放入section中

- 还可以使用fill来填充section，这通常用于填充未使用的section，fill with junk value

比如下面这段脚本表示把4k的section都充满0xaa这个值

```C++
SECTIONS {  
    .scream_page : {    
        FILL(0xaa)    
        . += 4K;  
    }
}
```

其他示例

```C++
SECTIONS {  
    .scream_page : {    
        FILL(0x0a)    
        . += 2K;    
        FILL(0xa0)    
        . += 2K;  
    }
}

SECTIONS {  
    .scream_page : {    
    . += 4K;  
    } = 0xaa;
}
```



**Linker symbols：链接符号**

可以直接在脚本中声明符号，比如声明一个入口或者栈的地址

```C++
SECTIONS {  
    .text : {    
        __text_start = .;    
        */* stuff */*    
        __text_end = .;  
    }
}
```

这样一来，用于初始化的代码就能够找到该部分的地址和长度

建议在这样使用时，前面下划线，避免C声明的符号冲突

需要区分开符号和section的区别！！！只在前面差了一个点，我曾经在编写ld脚本时，错把section写成了符号，这会导致根本不会分配一个section，一旦访问到这个section地址的内容就会page fault，并且bug也不是很好找，需谨记。





## **`PROVIDE()`****弱符号**

- 可以将符号定义封装在 `PROVIDE()` 函数中，使其成为“弱符号”，类似于 Clang 中的“弱符号”特性。它的逻辑是：**“如果你没有定义这个变量，那就用我给你的这个默认值；如果你定义了，那就听你的，我这个就当没看见。”像一个备胎或者default值**



**对比：强赋值 vs PROVIDE**

写法 A：强赋值（Strong Assignment）

```Plain Text
_stack_top = 0x20001000;
```

- 含义：强制将符号 `_stack_top` 定义为这个地址。

- 冲突：如果你的 C 代码里也写了 `int _stack_top = 0;`，链接器会报错："multiple definition of `_stack_top`"（多重定义错误）。

写法 B：PROVIDE（弱赋值）

```Plain Text
PROVIDE(_stack_top = 0x20001000);
```

- 含义：如果没人定义 `_stack_top`，那它就是 `0x20001000`。

- 冲突：如果你的 C 代码里写了 `int _stack_top = 0;`，链接器会优先使用你的 C 代码定义，忽略脚本里的 `0x20001000`，且不会报错。



**使用符号和LMA**

如前所述，LMA 和 VMA 不一致的情况极其罕见。最常见的情况是，在类似微控制器的系统中运行时，内存被分为两部分：ROM 和 RAM。ROM 中烧录了可执行文件，而 RAM 初始状态则包含随机垃圾数据

链接的可执行文件的大部分内容是只读的，因此它们的 VMA 可以放在 ROM 中。

- `.data` 和 `.bss` 段需要放在 RAM 中，因为它们是可写的。

    - 对 `.bss` 来说，这很容易，因为它没有可加载的内容。

    - 对于 `.data` ，我们需要将 VMA 和 LMA 分开：VMA 必须放在 RAM 中，而 LMA 放在 ROM 中。

这种区别对于初始化 RAM 的代码至关重要：而对于 `.bss` 文件，它只需要将其清零；而对于 `.data` ，它需要从 ROM 复制到 RAM！LMA 使我们能够区分复制源和复制目标。

```SQL
MEMORY {
  rom : /* ... */
  ram : /* ... */
}

SECTIONS {
  /* .text and .rodata just go straight into the ROM. We don't need
     to mutate them ever. */
  .text : { *(.text) } > rom
  .rodata : { *(.rodata) } > rom

  /* .bss doesn't have any "loadable" content, so it goes straight
     into RAM. We could include `AT> rom`, but because the sections
     have no content, it doesn't matter. */
  .bss : { *(.bss) } > ram

  /* As described above, we need to get a RAM VMA but a ROM LMA;
     the > and AT> operators achieve this. */
  .data : { *(.data) } > ram AT> rom
}

/* The initialization code will need some symbols to know how to
   zero the .bss and copy the initial .data values. We can use the
   functions from the previous section for this! */

bss_start = ADDR(.bss);
bss_end = bss_start + SIZEOF(.bss);

data_start = ADDR(.data);
data_end = data_start + SIZEOF(.data);

rom_data_start = LOADADDR(.data);
```

通常我们会用汇编编写初始化代码，因为C代码执行之前需要初始化\.bss和\.data，以及设置一些栈，但是为了便于说明，下面用C语言编写了一段初始化代码

```C++
#include <string.h>

extern char bss_start[];
extern char bss_end[];
extern char data_start[];
extern char data_end[];
extern char rom_data_start[];

void init_sections(void) {
  // Zero the .bss.
  memset(bss_start, 0, bss_end - bss_start);

  // Copy the .data values from ROM to RAM.
  memcpy(data_start, rom_data_start, data_end - data_start);
}
```



## linker相关杂项

其他链接器脚本功能

- `ENTRY()` 函数设置程序入口点，可以是符号或原始地址。一般设置为一个外部符号比如`ENTRY(_start)`

- `INPUT(foo.o)` 会将 `foo.o` 添加为链接器输入，就像它是通过命令行传递的一样

- `UTPUT()` 会覆盖通常的 `a.out` 默认输出名称

- `ASSERT()` 提供静态断言



**脚本可使用的一些函数**

- `ADDR` 、 `LOADADDR` 、 `SIZEOF` 和 `ALIGNOF` 分别用于生成先前定义的部分的 VMA、LMA、大小和对齐方式。

- `ORIGIN` 和 `LENGTH` ，分别用于生成内存区域的起始地址和长度。

- `MAX` 、 `MIN` 是显而易见的； `LOG2CEIL` 计算以 2 为底的对数，向上取整。

- `ALIGN(expr, align)` 将 `expr` 四舍五入到 `align` 的下一个倍数。 `ALIGN(align)` 大致等价于 `ALIGN(., align)` 但在 PIC 方面有一些细微差别。 `. = ALIGN(align);` 会将location counter与 `align` 对齐。



## example

1. 说了那么多，我们终于有了所有的前置知识，可以来看一个真正的ld脚本示例了

https://github\.com/tock/tock/blob/master/boards/build\_scripts/tock\_kernel\_layout\.ld



2. 下面给出一个linker脚本的错误示例：

我在使用gem5官方脚本时出现了访问栈错误，debug了很久，才定位到是ld脚本的问题

```C++
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

对于上面这样一个脚本，在编译成可执行程序之后会在访问栈的时候爆pagefault错误，试分析为什么

如何查看elf中某个地址是否有分配或者有映射，可以通过objdump \-h看这个section是否有ALLOC属性

```C++
Sections:
Idx Name          Size      VMA               LMA               File off  Algn
  0 .text         000006ce  0000000000000000  0000000000000000  00001000  2**1
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
  1 .bss          02000e40  00000000000006d0  00000000000006d0  000016ce  2**3
                  ALLOC
```

改成这样，就没错了，为什么

```C++
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
  .heap_low :{
    . = . + 0x1000000;
  }
  heap_top = .;
  .stack :{
    . = . + 0x1000000;
  }
  stack_top = .;
}
```

可以看到新增的section

```C++
Sections:
Idx Name          Size      VMA               LMA               File off  Algn
  0 .text         00000300  0000000000000000  0000000000000000  00001000  2**1
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
  1 .text.startup 000003ce  0000000000000300  0000000000000300  00001300  2**1
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
  2 .bss          00000e40  00000000000006d0  00000000000006d0  000016ce  2**3
                  ALLOC
  3 .heap_low     01000000  0000000000001510  0000000000001510  000016ce  2**0
                  ALLOC
  4 .stack        01000000  0000000001001510  0000000001001510  000016ce  2**0
                  ALLOC
```
