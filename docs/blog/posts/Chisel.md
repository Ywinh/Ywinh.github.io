---
date: 2026-04-01
categories:
  - HW
tags:
  - Writing
---

## 资料
1. [chisel cheatsheet](https://github.com/freechipsproject/chisel-cheatsheet/releases/latest/download/chisel_cheatsheet.pdf) 
2. https://github.com/schoeberl/chisel-book

#### Chisel学习资料

建议按照如下顺序学习:

1. [Chisel Bootcamp](https://mybinder.org/v2/gh/freechipsproject/chisel-bootcamp/master)是一个很不错的chisel教程, 还支持在线运行chisel代码, 你可以一边编写chisel代码一边学习. 其中
    - 第1章是scala入门
    - 第2章是chisel基础
    - 第3章是scala高级特性和chisel的混合使用
    - 第4章是FIRRTL后端相关内容 你需要完成前两章的学习, 同时我们强烈建议你学习第3章. 第4章和本课程没有直接关系, 可以作为课外阅读材料.
2. [Chisel Users Guide](https://www.chisel-lang.org/chisel3/docs/introduction.html)比较系统地整理了chisel的特性, 也是不错的入门教程.
3. [Chisel小抄](https://github.com/freechipsproject/chisel-cheatsheet/releases/latest/download/chisel_cheatsheet.pdf)简明地列出了chisel语言的大部分用法.
4. [Chisel API](https://www.chisel-lang.org/api/latest/)详细地列出了chisel库的所有API供参考.


# notes

## scala

var表示变量，val表示不变量

条件分支会返回所选择分支最后一行代码所对应的值
但在 Scala 中，表达式“ `if` ”会返回一个值。那个值是什么呢？它就是所选分支的最后一行代码所对应的值。这一特性非常实用，尤其是在用于在函数和类中初始化变量时。其用法如下：
```scala
val likelyCharactersSet = if (alphabet.length == 26)
    "english"
else 
    "not english"

println(likelyCharactersSet)
```

for循环：until表示不包括结尾数，to表示包括结尾数

scala是一种强类型语言，所有的类型转换都必须明确地进行。
val twotwo = 1.U + 1 会报错

## chisel
chisel是一个scala库
把面向对象和函数式语言之类的软件工程引入数字系统设计
对于chisel的硬件设计，verilog充当一个测试和综合中间语言

所有硬件模块必须继承`Module`类

macos安装chisel
依赖 AdoptOpenJDK、sbt、git、gtkwave、编辑器
gtkwave
```bash
brew install --HEAD randomplum/gtkwave/gtkwave
```

运算符
`:=`：表示右侧的信号驱动左侧的信号
和 = 有什么区别，= 是用来定义变量的？

数据类型：
* Bits：基本类型
* UInt：无符号整型
* SInt：有符号整型

```Chisel
Bits(8.W)  8-bit bits
UInt(8.W)  8-bit unsigned int
SInt(10.W) 10-bit signed int
```

常量：xxx.\[U,S\]
```
8.U(4.W) 表示4 width 的常量8
```

可能有误的地方 1.U(32) 不表示32位宽的常数1，而是表示从32位的bit提取，结果其实是0，1 => 000..01，其第32位为0

主要的逻辑就是 数+`.`+类型




bundle可以组织不同类型的信号（通过域访问，xx.xx），vec则组织相同类型的信号（每个元素可以通过索引访问）。bundle和vec可以任意交织

### control flow
chisel通过when来实现条件逻辑
```scala
when(someBooleanCondition) {
  // things to do when true
}.elsewhen(someOtherBooleanCondition) {
  // things to do on this condition
}.otherwise {
  // things to do if none of th boolean conditions are true
}
```

必须按照顺序出现when、else when、otherwise，后两个可以省略
与 Scala 中的 `if` 不同，与 `when` 相关的结构不会返回任何值。


### 组合逻辑
`Wire` 定义了一种电路组件，这种组件可以出现在 connect `:=` 运算符的右侧或左侧。
比如val row10 = Wire(UInt(16.W))

Chisel 提供了一个名为 `Enum` 的便捷函数，用于处理状态机的各种状态。在使用这些状态时，要把它们当作 `UInt` 字面量来处理。请记住，**硬件中的**相等判断是使用**三个等号**来表示的

### 时序逻辑

默认情况下，每个 Chisel 中的 `Module` 都拥有一个隐式的时钟信号，该时钟信号会被设计中的所有寄存器所使用。这样一来，就无需在代码中反复指定同一个时钟信号了。

`Reg` 会保持其当前的输出值，直到时钟信号出现上升沿为止。此时， `Reg` 会采用输入端的值。

clock.step(n) 会导致时钟被触发n次

Reg()和RegInit()
* Reg 中寄存器会被初始化为随机数据，也没有默认值和reset功能，只能指定width，不能指定初始值
* RegInit：创建一个能够被重置为特定值的寄存器，可以使用 `RegInit` 。

RegNext等价于先声明一个Reg，然后通过 := 赋值。表示下个上升沿更新

## 常用包装？

Valid()
* valid: Bool
* bits: 泛化类型


Decoupled() ：标准 valid/ready 握手接口
* output valid: Bool
* input ready: Bool
* output bits : t 的类型