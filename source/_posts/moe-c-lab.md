---
title: 在Linux下优雅的调试C语言
date: 2023-06-01 16:21:47
cover: /img/c-lab.jpg
top_img: /img/c-lab.jpg
tags:
  - Linux
  - C语言
---
最近在开发ruri时遇到不少问题，咱也是第一次写C，早知道头顶这么发凉就去用某邪教了呜喵～       
至于学习C语言的心得嘛，        
```
陷入无法察觉的overflow
沦落于oom-killer之下的死尸
就连无法看懂的魔数
也错以为是莫名能跑的奇迹
被泄漏的内存所填满
内核惶恐
逐渐失去的可维护性
终于咕咕而终
「bug还在↗↘↗↘↗↘↗➔➔↘↘」       
```
~~（高速退学）~~
好了好了，C语言还是有许多优点的，只是可能入门成本高些罢了，如果善用测试工具的话还是没有那么糟糕的，话不多说我们开始今天的正文。       
## 首要前提：
代码没bug的就不要调试了，编程第一法则不就是能跑的代码不要动嘛喵～      
过早的优化是万恶之源，测试时不要开-O2，且尽量使用`-O0 -fno-omit-frame-pointer -z norelro -z execstack -no-pie -fno-stack-protector -Wall -Wextra -pedantic -Wconversion`来测试。      
至于O3。。。除非编码特别规范否则几乎是炸屎。      
那如果有bug呢？      
首先得能过编译器，编译器都报error的代码再高端的调试工具也无能为力。       
然后检查编译器的警告，加上参数`-Wall -Wextra`编译然后查看警告，若是编译器警告都无法修复的话。。。这bug咱还是别修了吧喵～      
如果编译器不报警呢？      
于是就是今天的主题了--如何面对编译时无法找出的bug。       
## 消极面对：
部分内存问题可以通过编译器参数被隐藏，编译时加上`-fPIE -z noexecstack -z now -fstack-protector-all -fstack-clash-protection -mshstk -O2`说不定就能跑了喵～      
好了本文完，下期再见～      
桥豆麻袋，自己的项目中的代码肯定不能挖坑埋雷啊喵～     
## 积极面对：
中国有句古话叫做，食食物者为俊杰，眼下的各种工具，我想一定能找到阁下的bug。      
### 使用clang-tidy检查代码
clang-tidy是llvm项目的一部分，用于代码静态检测。      
由于clang-tidy过于优秀，大部分简单的bug在这里就会被检测出来，根本用不到真的运行，当然，它无法检查代码的功能是否可以正确实现，所以偶尔也必须得上gdb。      
基本用法：     
```sh
clang-tidy xxx.c -- 编译参数
```
注意编译参数前的`--`，后面接clang/gcc编译时的参数。      
但是，很多规则不是有用的，比如对strlen.h中函数内存安全的报警就非常多余，甚至clang-tidy会建议使用BSD中的函数替代，对此咱建议还是不要建议了。     
因此我们需要关闭部分检测项目。      
使用`--checks=-检测项`来关闭检测项。       
建议关闭的检测项：      
```
--checks=-clang-analyzer-security.insecureAPI.strcpy,-clang-analyzer-security.insecureAPI.DeprecatedOrUnsafeBufferHandling 
```
这样大部分内存泄漏等问题都可以被找出了。      
如果需要更多检测：       
```
--checks=*
```
比如ruri中的`make check`：
```
 --checks=*,-clang-analyzer-security.insecureAPI.strcpy,-altera-unroll-loops,-cert-err33-c,-concurrency-mt-unsafe,-clang-analyzer-security.insecureAPI.DeprecatedOrUnsafeBufferHandling,-readability-function-cognitive-complexity,-cppcoreguidelines-avoid-magic-numbers,-readability-magic-numbers,-bugprone-easily-swappable-parameters
```
目前ruri已经过了这些检测，但愿读者的内存永远不要泄漏喵～      
由于clang-tidy检测项太多，有些是对理解难度甚至是对todo风格等的检查，有些检查根本没法满足比如函数使用嵌套也会有警告，因此检查时需要我们手动对我们认为无效的警告进行过滤。      
一些有用的警告：      
数组越界      
内存泄漏      
未初始化的指针      
free()后还在使用的内存或者使用后没被free()的内存      
未初始化的变量      
void函数最后的return      
### 使用ASAN查看内存问题：
ASAN全称Address Sanitizer，是google发明的一种内存地址错误检查器，用于在运行时检测代码内存问题。      
如何使用：      
编译时加入参数`-O0 -fsanitize=address,undefined -fno-stack-protector -fno-omit-frame-pointer`
如果运气好的话，设置环境变量`LSAN_OPTIONS="verbosity=1:log_threads=1" ASAN_OPTIONS="verbosity=1"`，你将看到一片fa的冥场面。      
```log
SUMMARY: AddressSanitizer: heap-buffer-overflow asan_interceptors.cpp.o in printf_common(void*, char const*, __va_list_tag*)
Shadow bytes around the buggy address:
  0x0c427fff81d0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x0c427fff81e0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x0c427fff81f0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x0c427fff8200: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x0c427fff8210: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
=>0x0c427fff8220:[fa]fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x0c427fff8230: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x0c427fff8240: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x0c427fff8250: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x0c427fff8260: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x0c427fff8270: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
```
貌似还.......挺好看......      
一般来讲出问题的行会在后面汇报。      
注意：建议使用clang-tidy检查有bug的代码，因为一旦内存有问题的话很可能程序不会在出问题那行崩溃。      
ASAN面对fork()后的程序貌似有点抽风，ruri好不容易跑起来了，结果退出时卡在`sched_yield()`这个系统调用，欺负萌新是吧呜呜呜～      
在有些教程中ASAN偶尔会配合addr2line使用，实测貌似也定位不到相关行，或许是咱太笨了喵～      
### 使用GDB调试工具
GDB全称The GNU Project Debugger，是GNU项目的一部分。建议使用来检测代码是否实现而非内存问题，除非clang-tidy无法检测出来，在面对踩内存/带goto的代码时用GDB可能会十分痛苦。      
在编译时加如参数`-ggdb`，不要开任何优化，然后就可以使用gdb来调试程序了。     
注意，代码里少写两个goto有助于调试，白皮书说C语言提供了可以随意滥用的goto语句，瞧瞧这说的，像话吗喵！！！      
注意，请先使用clang-tidy检查是否有leak of memory等内存问题，否则你可能会遇上这种冥场面：      
```logs
Breakpoint 2, check_container (
container_dir=0x7ffffffdb0 "./t") at container.c:601
601       if (strcmp(container_dir, "/") == 0)
(gdb) c
Continuing.

Program received signal SIGSEGV, Segmentation fault.
0x0000007ff4505a10 in __strlen_aarch64 ()
from /apex/com.android.runtime/lib64/bionic/libc.so
```
咱甚至还在未申请到内存的结构体中读到过一个ELF，估计是指针指向程序头之类的地方了。      
基本命令：      
```sh
gdb ./可执行文件
```
或者对于运行中的程序：      
```sh
gdb attach <pid>
```
回车同意协议，然后你获得了一个这样的终端：      
```
For help, type "help".
Type "apropos word" to search for commands related to "word"...
Reading symbols from ./ruri...
(gdb) 
```
基本命令：      
开始运行程序：      
```
r 程序的命令行参数
```
设置断点：
```
b 行号
```
继续执行：
```
c
```
追踪子进程：
```
set follow-fork-mode child
```
查看当前行号：
```
where
```
查看上面的行：
```
up
```
打印变量：
```
p 变量名/表达式
```
这个功能真的震惊到咱了，因为C语言表达式都能用。      
比如：
```
(gdb) p container_info->container_dir
$1 = 0x7c2ff7ca8500 "/home/moe-hacker/t"
(gdb)
```
监控变量：      
```
watch 变量名
```
一个特殊的breakpoint：      
```
在出口处：
b exit
```
### GDB插件推荐：peda
peda(Python Exploit Development Assistance for GDB)是隔壁PWN大佬们的玩具，虽然咱看不懂汇编，不过其自动打印信息的功能还是很有帮助的。并且最重要的是它很帅啊！！！

看下对比：
```log
[----------------------------------registers-----------------------------------]
RAX: 0x1 
RBX: 0x7fffffffd658 --> 0x7fffffffda73 ("/home/moe-hacker/ruri/ruri")
RCX: 0x0 
RDX: 0x0 
RSI: 0x5555555682a0 ("\nor a full user guide, see `man ruri`\033[0m\nis program\n\nk if init command is like `su -`\n")
RDI: 0x1 
RBP: 0x1 
RSP: 0x7fffffff6398 --> 0x55555555f1c0 (mov    r14,QWORD PTR [rip+0x7e31]        # 0x555555566ff8)
RIP: 0x7ffff7dd0df0 (<exit>:    endbr64)
R8 : 0x410 
R9 : 0x1 
R10: 0x4 
R11: 0x202 
R12: 0x1 
R13: 0x7fffffffd668 --> 0x7fffffffda8e ("SHELL=/bin/zsh")
R14: 0x7ffff7ffd000 --> 0x7ffff7ffe2d0 --> 0x555555554000 --> 0x10102464c457f 
R15: 0x7fffffffd658 --> 0x7fffffffda73 ("/home/moe-hacker/ruri/ruri")
EFLAGS: 0x202 (carry parity adjust zero sign trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
   0x7ffff7dd0ddd:      call   0x7ffff7e19670 <__lll_lock_wait_private>
   0x7ffff7dd0de2:      jmp    0x7ffff7dd0bac
   0x7ffff7dd0de7:      nop    WORD PTR [rax+rax*1+0x0]
=> 0x7ffff7dd0df0 <exit>:       endbr64
   0x7ffff7dd0df4 <exit+4>:     push   rax
   0x7ffff7dd0df5 <exit+5>:     pop    rax
   0x7ffff7dd0df6 <exit+6>:     mov    ecx,0x1
   0x7ffff7dd0dfb <exit+11>:    mov    edx,0x1
[------------------------------------stack-------------------------------------]
0000| 0x7fffffff6398 --> 0x55555555f1c0 (mov    r14,QWORD PTR [rip+0x7e31]        # 0x555555566ff8)
0008| 0x7fffffff63a0 --> 0x0 
0016| 0x7fffffff63a8 --> 0x0 
0024| 0x7fffffff63b0 --> 0x0 
0032| 0x7fffffff63b8 --> 0x0 
0040| 0x7fffffff63c0 --> 0x0 
0048| 0x7fffffff63c8 --> 0x0 
0056| 0x7fffffff63d0 --> 0x0 
[------------------------------------------------------------------------------]
Legend: code, data, rodata, value

Breakpoint 1, 0x00007ffff7dd0df0 in exit () from /usr/lib/libc.so.6
gdb-peda$ 
```
不带peda：

```
Breakpoint 1, 0x00007ffff7dd0df0 in exit () from /usr/lib/libc.so.6
(gdb) 
```
最起码有种黑客的感觉喵～

### 使用strace工具：
strace全称Linux Syscall Tracer，听名字就知道，用于追踪进程的系统调用。众所周知，进程总要有系统调用，追踪这部分内容有时可以帮助我们发现问题。      
基本用法：      
```
对于已有进程：
strace -p 进程id
用strace来创建：
strace ./可执行文件
```
所以咱的程序在ASAN下卡在`sched_yield() = 0`是为什么啊喵！！！      
## 总结：
C语言虽然很容易写出bug，但是善用工具，养成良好的代码风格还是可以避免大部分问题的。还有就是，得会点英语。     
群里曾经有一位萌新问道：    
"如果我想入门编程语言，学哪种比较好？"    
大佬答："英语"    
本文完，我们下期再见喵～    
EOF