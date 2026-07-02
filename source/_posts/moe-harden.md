---
title: Clang/GCC安全编译与代码优化选项（合集）
date: 2023-12-29 16:09:31
cover: /img/harden.jpg
top_img: /img/harden.jpg
tags:
  - Linux
  - C语言
---
好久不见喵～
实在想不出开头就不想了，本期文章咱们来讲讲Clang/GCC的安全编译与代码优化选项。
注意：优化选项建立在代码正确的前提下，且最好不要在使用GDB等工具调试时开启任何优化选项。
# LTO（Link-Time Optimization）：
中文是`链接时优化`，最初由LLVM实现，可做到在编译时跨模块执行代码优化，功能有：
- 函数自动内联
- 去除无用代码
- 全局优化

LTO有大型LTO（monolithic LTO）和增量LTO（ThinLTO）两种实现，其中ThinLTO内存占用较少。
虽然对于小型项目几乎无影响，但Google的内核源码由于默认是大型LTO，曾将咱的电脑整OOM了三回啊三回。
据说ThinLTO有时反而有更好的性能，具体咱不清楚。
编译参数（二选一）：
```
# monolithic LTO：
-flto
# ThinLTO：
-flto=thin
```
# PIE（Position-Independent Executables）：
中文是`位置无关可执行程序`，配合内核的ASLR（Address Space Layout Randomization，地址空间随机化）功能将代码段地址随机化，以增大攻击难度。
编译参数：
```
-fPIE
```
# NX（No-eXecute）：
中文是`不可执行`，将数据段内存标记为不可执行以防御数据溢出漏洞。
编译参数（注：用于链接阶段）：
```
-z noexecstack
```
# RELRO（Relocation Read-Only）：
中文是`只读重定向`，用于保护GOT（Global Offset Table，全局偏移表）避免其符号被篡改。
编译参数（注：用于链接阶段）：
```
-z now
```
# Stack Canary：
中文是`栈溢出哨兵`，用于验证栈空间是否完整。
编译参数：
```
-fstack-protector-all
```
# Stack Clash Protection：
中文是`栈冲突保护`，保护栈地址不受冲突的内存破坏。
编译参数：
```
-fstack-clash-protection
```
# Shadow Stack：
中文是`影子堆栈`，用于分割栈空间以增大跨栈攻击的难度。
编译参数：
```
-mshstk
```
# 变量自动初始化：
对于未初始化的变量可以自动填0，咱建议还是在代码里初始化为0比较好。
此特性可能需要较新版本编译器。
编译参数：
```
-ftrivial-auto-var-init=zero
```
# Fortified Source：
中文是`强化源代码`...emmm越来越抽象了的说。
实际上就是自动替换glibc中的函数为安全实现。
编译参数（需要优化级别大于0）：
```
-D_FORTIFY_SOURCE=3
```
# “一键”优化：
这个就很熟悉了，O2到底是不是优化全靠运气，O3强力炸屎你值得拥有。。。
编译参数：
```
# 关闭优化：
-O0
# 少量优化，推荐对代码正确性没有信心时使用：
-O1
# 普通优化，对代码有信心时使用：
-O2
# 激进优化，建议小白鼠使用：
-O3
# 更加激进地优化速度：
-Ofast
# Clang的下一代激进优化，作用是可能让代码面对多年后的编译器突然炸了：
-O4
# 基于O2兼顾体积与速度：
-Os
# 基于Os尽量减小体积：
-Oz
# 用于调试：
-Og
```
# strip（object stripping tool）：
开启优化必然会导致生成的二进制文件体积增大，所以最后给大家推荐下strip工具，用于删除二进制文件冗余信息以减小体积。
用法非常简单，直接`strip in-file`即可。
注：strip仅用于Release。
# 后记：
原谅咱实在是才疏学浅，有些内容作者也不理解。。。。
不过只要代码能过clang-tidy，优化选项开就完了。
至于代码不规范的，千万不要动，因为这样违反了能跑就行原则。