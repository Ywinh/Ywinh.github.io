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
chisel是一个scala库
把面向对象和函数式语言之类的软件工程引入数字系统设计
对于chisel的硬件设计，verilog充当一个测试和综合中间语言


macos安装chisel
依赖 AdoptOpenJDK、sbt、git、gtkwave、编辑器
gtkwave
```bash
brew install --HEAD randomplum/gtkwave/gtkwave
```

