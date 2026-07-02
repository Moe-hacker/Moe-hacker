---
title: Re：从零开始的容器安全——ruri开发笔记
date: 2023-07-31 10:14:26
cover: /img/container-lab.jpg
top_img: /img/container-lab.jpg
tags:
- Linux
- C语言
- Container
---
# 前言：
ruriv2.0刚发了rc1（现在是3.1-rc1了都），之前一直咕咕咕着的开发笔记差不多也该写写了喵～        
笔记主要讲容器及安全原理，使用C语言实现。      
头图是项目最早的版本，真是怀念呢喵，那时候咱连数组都不会用，现在ruri代码都突破4k行了。      
# 容器基本原理：
## Linux挂载点/设备文件：
众嗦粥汁，Linux下的/proc，/sys与/dev均在开机时由init或其子服务创建，部分系统同时会将/tmp挂载为tmpfs，它们都需要被手动挂载到容器才可保证容器中程序正常运行。     
其中，/proc为procfs，/sys为sysfs，/dev为tmpfs。      
你还需要在容器中创建/dev下的设备节点文件。
部分文章在创建chroot/unshare容器时都会直接映射宿主机的/dev目录，这是十分危险的，正确的做法是参照docker容器默认创建的设备文件列表去手动创建这些节点。      
当然了，docker也会将/sys下部分目录挂载为只读,ruri借鉴了其挂载点，详情可以去看ruri源码或者运行个docker容器看看它的挂载点。      
需要注意的是，只读挂载需要先bind-mount再remount-ro，示例写法如下：      
```C
// Bind-mount.
mount(source, target, NULL, mountflags | MS_BIND, NULL);
// Remount.
mount(source, target, NULL, mountflags | MS_BIND | MS_REMOUNT, NULL);
```
## 其他容器注意事项：   
Android的/data默认为nosuid挂载，/sdcard甚至是noexec，所以在安卓/data下创建容器时请将/data重挂载为suid，不要在/sdcard创建容器。      
Archlinux的根目录需要在挂载点上，也就是说需要将容器目录自身bind-mount到自身再进行chroot(2)，否则pacman无法运行。      
/proc/mounts到/etc/mtab的链接可能需要手动创建。     
大部分rootfs的resolv.conf为空需手动创建，安卓系统内容器联网需手动设置，授予需要联网的网络用户组(aid_inet,aid_net_raw)权限。      
## chroot(2):
chroot可是个老家伙了，从1979年Seventh Edition Unix的开发时便产生了这项技术。
### 函数调用：
chroot(2)函数来自`unistd.h`，原型为
```C
int chroot(const char *path);
```
它的作用是改变程序自己认为的根目录为path，需要root权限(其实是CAP_SYS_CHROOT，后面会讲)。       
chroot(2)类似chroot(8)命令，但是它在变更完根目录后什么都不会做。      
~~貌似我们不是C语言入门教学~~      
不装了，直接上代码吧。      
### 一个完整的chroot程序：
```C
#include <errno.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/dir.h>
#include <unistd.h>
int main(int argc, char **argv)
{
  // getuid()返回当前用户uid，类似`id -u`
  if (getuid() != 0)
  {
    // 使用fprintf()输出到stderr更加规范
    fprintf(stderr, "\033[31mError: this program should be run with root privileges !\033[0m\n");
    exit(1);
  }
  if (argc <= 1)
  {
    fprintf(stderr, "\033[31mError: too few arguments !\033[0m\n");
    exit(1);
  }
  // LD_PRELOAD所指定的库如果不存在的话exec()会失败，因此不能设置LD_PRELOAD
  char *ld_preload = getenv("LD_PRELOAD");
  if (ld_preload != NULL)
  {
    fprintf(stderr, "\033[31mError: please unset $LD_PRELOAD before running this program or use su -c `COMMAND` to run.\033[0m\n");
    exit(1);
  }
  char *container_dir = argv[1];
  char *command[1024] = {0};
  // 未指定执行的命令的话默认为`/bin/su -`
  if (argc == 2)
  {
    command[0] = "/bin/su";
    command[1] = "-";
    command[2] = NULL;
  }
  else
  {
    int i = 0;
    for (int j = 2; j < argc; j++)
    {
      command[i] = argv[j];
      i++;
    }
    command[i + 1] = NULL;
  }
  DIR *direxist;
  // opendir()用于判断容器目录是否存在
  if ((direxist = opendir(container_dir)) == NULL)
  {
    fprintf(stderr, "\033[31mError: container directory does not exist !\033[0m\n");
    exit(1);
  }
  else
  {
    closedir(direxist);
  }
  // 切换根目录
  chroot(container_dir);
  // chdir()类似cd命令
  chdir("/");
  // execv()用于执行命令，strerror(errno)用于捕获异常
  if (execv(command[0], command) == -1)
  {
    fprintf(stderr, "\033[31mFailed to execute %s\n", command[0]);
    fprintf(stderr, "execv() returned: %d\n", errno);
    fprintf(stderr, "error reason: %s\033[0m\n", strerror(errno));
    exit(1);
  }
}
```
用法为：
```sh
cc 文件名.c
./a.out NEWROOT [COMMAND [ARG]...]
```
十分的简单，甚至九分的简单。      
### 安全性：
chroot实现了根目录隔离，但是chroot()后的进程会继承父进程特权，而且不幸的是chroot()必须以root(CAP_SYS_CHROOT)特权执行。chroot()后容器仍可访问外部资源，包括但不限于在容器内执行以下操作：      
- kill外部进程
- 创建设备节点并挂载磁盘设备
- 当场逃逸出容器
### 逃逸：
在/proc被挂载的chroot容器中，可能可以通过`chroot /proc/1/root`直接逃逸出容器，可行的解决方式是开启PID NS来避免这一问题。     
实测wsl1有此逃逸漏洞，wsl2的Linux内核已经修复。        
因此不要将chroot容器用于生产，老老实实地pull个docker image吧还是。
## unshare(2):
Linux内核自2.4版本引入第一个namespace，即mount ns，当初估计作者没打算再加其他隔离就命名为CLONE_NEWNS了，此宏定义一直被沿用至今。      
（Linus：我们不破坏用户空间23333）      
### Linux父子进程：
众所周知(读者：喵喵喵？我怎么不知道？)，Linux下运行的所有进程都是init的子进程，子进程由父进程经fork(2)或clone(2)创建，继承父进程的文件描述符与UID/GID/权限(特权)等。      
子进程死亡了若父进程没有对其wait(2)或waitpid(2)，则成为僵尸进程。       
父进程先走一步的话，子进程被init直接接管。     
在pid ns中被执行的第一个进程在该ns中被认为与init等效(具有pid 1)，当其死亡时pid ns被内核销毁，其子进程被一同销毁。      
### 函数调用：
unshare(2)函数来自`sched.h`，需要先`#define _GNU_SOURCE`。      
原型为：
```C
int unshare(int flags);
```
它的作用是将进程以及其后的子进程的特定flag隔离。      
Flags:      
```
CLONE_NEWNS (since Linux 2.4)
CLONE_NEWUTS (since Linux 2.6.19)
CLONE_NEWIPC (since Linux 2.6.19)
CLONE_NEWNET (since Linux 2.6.24)
CLONE_SYSVSEM (since Linux 2.6.26)
CLONE_NEWUSER (since Linux 3.8)
CLONE_NEWPID (since Linux 3.8)
CLONE_NEWCGROUP (since Linux 4.6)
CLONE_NEWTIME (since Linux 5.6)
```
### 使用unshare(2)：
ruri没有开启net和user命名空间，它里面unshare相关的代码为：      
```C
pid_t init_unshare_container(bool no_warnings)
{
  pid_t unshare_pid = INIT_VALUE;
  // 创建命名空间
  unshare(CLONE_NEWNS);
  unshare(CLONE_NEWUTS);
  unshare(CLONE_NEWIPC);
  unshare(CLONE_NEWPID);
  unshare(CLONE_NEWCGROUP);
  unshare(CLONE_NEWTIME);
  unshare(CLONE_SYSVSEM);
  unshare(CLONE_FILES);
  unshare(CLONE_FS);
  // 修复`can't fork: out of memory`问题
  // fork()后进程彻底被隔离
  unshare_pid = fork();
  // 修复`can't access tty`
  if (unshare_pid > 0)
  {
    usleep(200000);
    waitpid(unshare_pid, NULL, 0);
  }
  else if (unshare_pid < 0)
  {
    error("Fork error QwQ?");
  }
  return unshare_pid;
}
```
### setns(2)：
在容器已经运行后，我们可以使用setns()加入容器的namespace，用法如下：
首先我们知道容器的ns所有者pid（ns_pid）,然后：
open()打开`/proc/{ns_pid}/ns/{namespace}`,获取ns_fd, 其中namespace是想加入的ns名称，
获取到ns_fd后，`setns(ns_fd,0);`即可。
注意：mount ns应该始终最后一个被加入，因为加入后/proc下的内容在开启pid ns的情况下会变化导致无法setns(2)。
### 安全性：
即使在ns全开的设备中，unshare()后容器中的进程中的root权限依然等于宿主系统中的root权限，因此最简单的攻击方式便是直接修改磁盘中的文件，当然也有其它逃逸方式可进行攻击。
## pivot_root(2):
pivot_root()类似于chroot(),但更加安全，它的原理是更改当前mount ns的根目录，并将原根目录移动到新的目录。      
ruri v3.7正式为unshare容器启用pivot_root替代chroot。      
他的使用方式如下：
首先我们需要一个mount ns，可以`unshare(CLONE_NEWNS)`后fork()一下自己。
其次，pivot_root()的new_root以及其父挂载点需要是MS_PRIVATE属性。
最后，new_root需要是一个挂载点。
在mount ns的所有者进程中，可以这样使用pivot_root:
```C
// 将根目录挂载为MS_PRIVATE
mount("none", "/", NULL, MS_REC | MS_PRIVATE, NULL);
// 挂载容器目录自身
mount(container_dir, container_dir, NULL, MS_BIND | MS_PRIVATE, NULL);
// 切换到容器目录
chdir(container_dir);
// 运行pivot_root
syscall(SYS_pivot_root,".",".");
// 切换到根目录
chdir("/");
// 取消原root的挂载
umount2(".",MNT_DETACH);
```
在非ns所有者进程中，我们可以通过setns(2)加入已经运行过pivot_root的mount namespace, 然后直接`chdir("/")`即可。
## Rootless容器：
建议配合咱的另一篇文章阅读：[rootless容器开发指北](https://blog.crack.moe/2024/10/31/rootless-container/)      
首先我们需要CLONE_NEWUSER创建一个user ns。
然后，为了使用mount(2),我们还需要CLONE_NEWNS来成为当前mount ns的所有者。
最后，为了挂载procfs，我们需要CLONE_NEWPID，很怪，但实测没有这个不行。
然后我们设置uidmap和gidmap：
```C
uid_t uid = geteuid();
gid_t gid = getegid();
// Set uid map.
char uid_map[32] = { "\0" };
sprintf(uid_map, "0 %d 1\n", uid);
int uidmap_fd = open("/proc/self/uid_map", O_RDWR | O_CLOEXEC);
write(uidmap_fd, uid_map, strlen(uid_map));
close(uidmap_fd);
// Set gid map.
int setgroups_fd = open("/proc/self/setgroups", O_RDWR | O_CLOEXEC);
write(setgroups_fd, "deny", 5);
close(setgroups_fd);
char gid_map[32] = { "\0" };
sprintf(gid_map, "0 %d 1\n", gid);
int gidmap_fd = open("/proc/self/gid_map", O_RDWR | O_CLOEXEC);
write(gidmap_fd, gid_map, strlen(gid_map));
close(gidmap_fd);
// Maybe needless.
setuid(0);
setgid(0);
```
最后我们将宿主机的sysfs挂载上：
```C
mount("/sys", "./sys", NULL, MS_BIND | MS_REC, NULL);
```
然后是procfs：
```C
mount("proc", "./proc", "proc", MS_NOSUID | MS_NOEXEC | MS_NODEV, NULL);
```
最后，/dev下的设备需要手动从宿主机映射：
```C
open("./dev/tty", O_WRONLY | O_CREAT | O_TRUNC | O_CLOEXEC, S_IRUSR | S_IWUSR | S_IROTH | S_IWOTH | S_IRGRP | S_IWGRP);
mount("/dev/tty", "./dev/tty", NULL, MS_BIND, NULL);
```
就可以正常chroot(2)了，还是很简单的。
## Cgroup：
cgroup即control group，用于限制进程资源。
目前ruri只支持cpuset和memory这两个cgroup。
首先我们需要挂载cgroup的apifs：
对于cgroup v1：
```C
mkdir("/sys/fs/cgroup", S_IRUSR | S_IWUSR);
// Mount /sys/fs/cgroup as tmpfs.
mount("tmpfs", "/sys/fs/cgroup", "tmpfs", MS_NOSUID | MS_NODEV | MS_NOEXEC | MS_RELATIME, NULL);
// We only need cpuset and memory cgroup.
mkdir("/sys/fs/cgroup/memory", S_IRUSR | S_IWUSR);
mkdir("/sys/fs/cgroup/cpuset", S_IRUSR | S_IWUSR);
mount("none", "/sys/fs/cgroup/memory", "cgroup", MS_NOSUID | MS_NODEV | MS_NOEXEC | MS_RELATIME, "memory");
mount("none", "/sys/fs/cgroup/cpuset", "cgroup", MS_NOSUID | MS_NODEV | MS_NOEXEC | MS_RELATIME, "cpuset");
```
而对于v2:
```C
mkdir("/sys/fs/cgroup", S_IRUSR | S_IWUSR);
mount("none", "/sys/fs/cgroup", "cgroup2", MS_NOSUID | MS_NODEV | MS_NOEXEC | MS_RELATIME, NULL);
```
最后，我们在cgroup目录下创建子cgroup，将进程自身加入cgroup.procs然后设置相关限制即可生效。
这部分不细讲了，因为ruri也只是对于cpuset和memory限制简单实现了下。      
# 容器安全：
## 进程属性查看：
```
cat /proc/$$/status
```
CapEff等表示当前进程capabilities，使用`capsh --decode=xxxxx`来解码。      
NoNewPrivs，Seccomp和Seccomp_filters后面会讲。      
## capabilities(7)与libcap(3):
从2.2版本开始，Linux内核将进程root特权分割为可独立控制的部分，称之为capability，capability是线程属性(per-thread attribute)，可从父进程继承。      
通常所说的root权限其实是拥有相关capability，如chroot(2)其实需要的是CAP_SYS_CHROOT。      
目前内核中定义的capability：
```
CAP_CHOWN 变更文件所有权
CAP_DAC_OVERRIDE 忽略文件的DAC访问限制
CAP_DAC_READ_SEARCH 忽略文件读及目录搜索的DAC访问限制
CAP_FOWNER 忽略文件属主ID必须和进程用户ID相匹配的限制
CAP_FSETID 允许设置文件的setuid位
CAP_IPC_LOCK 允许锁定共享内存片段
CAP_IPC_OWNER 忽略IPC所有权检查
CAP_KILL 向非当前用户进程发送信号（杀死）
CAP_LEASE 允许修改文件锁的FL_LEASE标志
CAP_LINUX_IMMUTABLE 允许修改文件的IMMUTABLE和APPEND属性标志
CAP_NET_ADMIN 允许执行网络管理任务
CAP_NET_BIND_SERVICE 允许绑定到小于1024的端口
CAP_NET_BROADCAST 允许网络广播和多播访问
CAP_NET_RAW 使用原始套接字
CAP_SETGID 设置gid
CAP_SETPCAP 设置其他进程capability
CAP_SETUID 设置uid
CAP_SYS_ADMIN 允许执行系统管理任务，如加载或卸载文件系统、设置磁盘配额等
CAP_SYS_BOOT 重启设备
CAP_SYS_CHROOT 调用chroot
CAP_SYS_MODULE 加载/删除内核模块
CAP_SYS_NICE 允许提升优先级及设置其他进程的优先级
CAP_SYS_PACCT 允许执行进程的BSD式审计
CAP_SYS_PTRACE 允许对任意进程进行ptrace
CAP_SYS_RAWIO 访问原始块设备
CAP_SYS_RESOURCE 忽略资源限制
CAP_SYS_TIME 设置系统时间
CAP_SYS_TTY_CONFIG 设置tty设备
CAP_MKNOD (since Linux 2.4) 创建设备节点
CAP_AUDIT_CONTROL (since Linux 2.6.11) 启用和禁用内核审计；改变审计过滤规则；检索审计状态和过滤规则
CAP_AUDIT_WRITE (since Linux 2.6.11) 将记录写入内核审计日志
CAP_SETFCAP (since Linux 2.6.24) 允许为文件设置任意的capabilities
CAP_MAC_ADMIN (since Linux 2.6.25) 允许MAC配置或状态更改
CAP_MAC_OVERRIDE (since Linux 2.6.25) 覆盖MAC设置
CAP_SYSLOG (since Linux 2.6.37) 允许使用syslog()系统调用
CAP_WAKE_ALARM (since Linux 3.0) 允许触发一些能唤醒系统的东西
CAP_BLOCK_SUSPEND (since Linux 3.5) 使用可以阻止系统挂起的特性
CAP_AUDIT_READ (since Linux 3.16) 允许通过 multicast netlink 套接字读取审计日志
CAP_BPF (since Linux 5.8) 使用bpf()
CAP_PERFMON (since Linux 5.8) 使用perf_event_open()
CAP_CHECKPOINT_RESTORE (since Linux 5.9) 调用checkpoint/restore
```
具体哪些capability需要移除那些保留可直接参照docker。      
### 函数调用：
说实话这东西有点抽象，理论上只移除CapBnd即可生效，Linux 6.6（Archlinux）下其他值会被一并移除，但KernelSU貌似有Bug导致不检查CapBnd。
咱strace追踪了containerd的调用，发现它会同时移除CapAmb和清零CapInh。
首先是CapBnd的移除，使用libcap：
```C
int cap_drop_bound(cap_value_t cap);
```
cap值是上面讲到的宏。
CapAmb的移除需要prctl(2):
```C
prctl(PR_CAP_AMBIENT, PR_CAP_AMBIENT_LOWER, cap, 0, 0);
```
最后是CapInh的清除：
```C
// Clear CapInh.
cap_user_header_t hrdp = (cap_user_header_t)malloc(sizeof *hrdp);
cap_user_data_t datap = (cap_user_data_t)malloc((sizeof *datap) * 10);
hrdp->pid = getpid();
hrdp->version = _LINUX_CAPABILITY_VERSION_3;
syscall(SYS_capget, hrdp, datap);
datap->inheritable = 0;
syscall(SYS_capset, hrdp, datap);
free(hrdp);
free(datap);
```
这段着实有点抽象，hrdp和datap事实上是两个指针，datap需要是一个数组，实测只分配一个datap大小的内存会崩。
## seccomp(2)与libseccomp
Secommp (SECure COMPuting，安全计算模式)，自Linux 2.6.12被引入，用于对进程的系统调用进行限制，个人理解为：      
在启用了Seccomp的设备上：      
进程可执行的系统调用 ==（基本系统调用+进程capability所授予的特权调用）∩ seccomp允许的系统调用           
Seccomp有strict mode和filter mode两种开启模式。
Strict mode:严格模式，怕是已经被弃用了，只是由于历史原因留着，对咱没啥用处。
Filter mode:BPF过滤器模式，对白名单外的系统调用进行过滤。
### 函数调用：
用到`seccomp.h`中的函数来开启bpf模式，完整操作过程为：
```C
// 初始化规则，ruri自3.1起使用黑名单，因为白名单咱也不会写23333
scmp_filter_ctx ctx = seccomp_init(SCMP_ACT_ALLOW);
// 添加黑名单
seccomp_rule_add(ctx, SCMP_ACT_KILL, SCMP_SYS(系统调用), 0);
// ......
// ......
// 关闭默认NO_NEW_PRIV位
seccomp_attr_set(ctx, SCMP_FLTATR_CTL_NNP, 0);
// 载入规则
seccomp_load(ctx);
```
## NO_NEW_PRIV位：
也是进程属性，它就像一个奴籍(作者自己都觉得奇怪的比喻，但是不觉得很合理吗？)，一旦被设置，进程自身及其子进程均无法主动取消此标志。此属性的作用是限制进程的特权集始终小于等于其父进程，也就是说在设置后进程特权只能减少不能增加。此标志设置后可执行文件suid位以及capability属性均无法生效，就连切换用户为普通用户后执行sudo命令也会失效。      
它的设置十分简单，直接使用`sys/prctl.h`中的prctl(2)函数：      
```
prctl(PR_SET_NO_NEW_PRIVS, 1, 0, 0, 0);
```
## Secure bit:
内核对进程有额外的secure bit属性，可以去man里面了解下，目前作者只加入了SECBIT_NO_CAP_AMBIENT_RAISE：
```C
prctl(PR_SET_SECUREBITS, SECBIT_NO_CAP_AMBIENT_RAISE);
```
# 后记：
文采太差，没有后记，散会。。。

<p align="center">優しさも笑顔も夢の語り方も、</p>
<p align="center">知らなくて全部、</p>
<p align="center">君を真似たよ</p>