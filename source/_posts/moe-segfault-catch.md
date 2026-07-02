---
title: 论如何在Segmentation fault时优雅地结束程序
date: 2024-01-30 14:26:59
cover: /img/moe-segfault.jpg
top_img: /img/moe-segfault.jpg
tags:
  - Linux
  - C语言
---
作为开发者，咱自然是不喜欢程序发生Segfault的“喜报”的，但是万一有些用户非闹着要在酒吧里点炒饭，程序还是大概率会崩。（不崩才怪呢喵！）
glibc下还好，崩了最起码显示个cmdline，bionic就可能啥也没有了。
于是，虽然咱不希望出问题，最起码出问题时程序走的安详点，不要出现啥信息也没有的“死不瞑目”的场面。
然后就是捕捉段错误的原理了：
众所周知，段错误是当前这段错误，程序其他段大概率还是能跑的。在发生段错误时，内核就会给进程发送相应SIGSEGV信号：这位进程你已经寄了，准备下后事吧喵～
其它信号如SIGABRT，SIGBUS，SIGFPE，SIGILL，SIGQUIT，SIGSYS，SIGTRAP，SIGXCPU和SIGXFSZ，对应了不同的异常情况，默认行为也是core dump，具体可以看signal(7)。
事实上这些信号并不是说必须强制终止程序，当然你也可以选择直接忽略，然后内核会不停发送信号，最后就是死循环，核心思想便是“只要我不主动崩溃，我就没有崩溃”的精神胜利(逃)。。。
收到相应信号后，进程是可以选择自己到底要干什么的，默认是按照系统配置生成core dump文件，也可以自定义要执行什么。
既然能自定义，那么我们就简单的写一个在段错误时输出进程基本信息的程序：
```C
#include <signal.h>
#include <stdio.h>
#include <unistd.h>
// Show some extra info when segfault.
void sighandle(int sig) {
  signal(sig, SIG_DFL);
  int clifd = open("/proc/self/cmdline", O_RDONLY);
  char buf[1024];
  size_t bufsize = read(clifd, buf, sizeof(buf));
  fprintf(stderr, "\033[1;38;2;254;228;208m");
  fprintf(stderr, "SIG: %d\n", sig);
  fprintf(stderr, "UID: %u\n", getuid());
  fprintf(stderr, "PID: %d\n", getpid());
  fprintf(stderr, "CLI: ");
  for (size_t i = 0; i < bufsize - 1; i++) {
    if (buf[i] == '\0') {
      fputc(' ', stderr);
    } else {
      fputc(buf[i], stderr);
    }
  }
  fprintf(stderr, "\nThis message might caused by an internal error.\n\033[0m");
}
// Catch coredump signal.
void register_signal(void) {
  signal(SIGSEGV, sighandle);
}
int main(void) {
  register_signal();
  char *i = NULL;
  *i = 114;
}
```
这里提一句/proc/self/cmdline文件的解析，这个文件非常邪门，因为命令行参数用'\0'分割，因此只能通过read(2)的返回值来判断内容的长度。
sighandle()函数中需要重新将我们收到的错误信号的处理函数注册为默认的，这样程序执行完sighandle()后再次收到相同信号，就会被libc默认处理函数捕获，当然你也可以选择在这里直接退出。
至于段错误的触发，这里咱选择向NULL地址写入值。正确的代码写多了，想写个不正确的都得憋半天。
最后我们直接编译运行：
```
~ $ cc -O0 -fno-omit-frame-pointer -z norelro -z execstack -fno-stack-protector t.c
~ $ ./a.out
SIG: 11
UID: 10467
PID: 8191
CLI: ./a.out
This message might caused by an internal error.
Segmentation fault
~ $
```
在ruri中：
```
~/ruri # ./ruri -U xxxx
  .^.   .^.
  /⋀\_ﾉ_/⋀\
 /ﾉｿﾉ\ﾉｿ丶)|
 ﾙﾘﾘ >  x )ﾘ
ﾉノ㇏  ^ ﾉ|ﾉ
      ⠁⠁
Seems that it's time to abort.
SIG: 11
UID: 0
PID: 26406
CLI: ./ruri -U xxxx
This message might caused by an internal error.
If you think something is wrong, please report at:
https://github.com/Moe-hacker/ruri/issues

Segmentation fault
~/ruri #
```
