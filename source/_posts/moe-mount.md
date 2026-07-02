---
title: C语言实现基本的mount命令挂载磁盘/镜像/目录
date: 2024-02-22 19:25:24
cover: /img/gnu.jpg
top_img: /img/gnu.jpg
tags:
  - Linux
  - C语言
---
mount(2)函数是个很简单的函数，原型如下：
```C
int mount(const char *source, const char *target,
                 const char *filesystemtype, unsigned long mountflags,
                 const void *_Nullable data);
```
对于目录和nodev文件系统(如proc和sysfs)的挂载，可以说是非常简单：
```C
// 把'/' bind-mount到'/'
mount("/", "/", NULL, MS_BIND, NULL);
// 把proc文件系统挂载到/proc
mount("proc", "/proc", "proc", MS_NOSUID | MS_NOEXEC | MS_NODEV, NULL);
```
可见mount(2)函数的使用非常简单。
由于bind-mount对于type没有任何要求，咱这个笨蛋就想当然地觉得mount()函数只要填上source和target就好了，真到实验时才发现并不是这样！！！
我们追踪一个mount命令挂载磁盘的过程：
```sh
strace mount /dev/block/by-name/logfs .
```
发现：
```C
openat(AT_FDCWD, "/proc/filesystems", O_RDONLY) = 3
read(3, "nodev\tsysfs\nnodev\ttmpfs\nnodev\tbd"..., 1024) = 419
mount("/dev/block/by-name/logfs", ".", "ext3", MS_SILENT, NULL) = -1 EINVAL (Invalid argument)
mount("/dev/block/by-name/logfs", ".", "ext2", MS_SILENT, NULL) = -1 EINVAL (Invalid argument)
mount("/dev/block/by-name/logfs", ".", "ext4", MS_SILENT, NULL) = -1 EINVAL (Invalid argument)
mount("/dev/block/by-name/logfs", ".", "vfat", MS_SILENT, NULL) = 0
```
可见当前咱用的这个mount命令也不知道挂载类型是什么，总结来讲挂载方式便是，一一试，无遗失。
当然还有一种方式那就是通过磁盘头部数据来判断分区格式，但是太难写了，咱不会。

然后我们需要知道/proc/filesystems的格式，它的每一行由如下数据组成：
```
[flag]\t[filesystem type]\n
```
其中flag为nodev时，表示这是一个虚拟文件系统，不用做磁盘设备挂载。\t分割了flag和filesystem type，于是我们可以写一个非常简单的trymount()函数：
```C
int trymount(const char *dev, const char *path)
{
        int ret = 0;
        // Get filesystems supported.
        int fssfd = open("/proc/filesystems", O_RDONLY | O_CLOEXEC);
        char buf[4096] = { '\0' };
        read(fssfd, buf, sizeof(buf));
        close(fssfd);
        char type[128] = { '\0' };
        char label[128] = { '\0' };
        char *out = label;
        int i = 0;
        bool nodev = false;
        for (size_t j = 0; j < sizeof(buf); j++) {
                // Reached the end of buf.
                if (buf[j] == '\0') {
                        break;
                }
                // Check for nodev flag.
                if (buf[j] == '\t') {
                        if (strcmp(label, "nodev") == 0) {
                                nodev = true;
                        }
                        out = type;
                        i = 0;
                        memset(label, '\0', sizeof(label));
                }
                // The end of current line.
                else if (buf[j] == '\n') {
                        if (!nodev) {
                                ret = mount(dev, path, type, 0, NULL);
                                if (ret == 0) {
                                        // mount(2) succeed.
                                        return ret;
                                }
                                memset(type, '\0', sizeof(type));
                        }
                        out = label;
                        i = 0;
                        nodev = false;
                }
                // Get fstype.
                else {
                        out[i] = buf[j];
                        out[i + 1] = '\0';
                        i++;
                }
        }
        return ret;
}
```
分区的挂载我们解决了，我们还需要解决镜像文件的挂载。
在终端中，镜像文件一般先用losetup命令和loop设备绑定后挂载，而这个过程的原理如下：
1.进程打开/dev/loop-control。
2.进程使用ioctl(2)向/dev/loop-control发送LOOP_CTL_GET_FREE指令，生成一个空闲的loop设备并获取设备编号。
3.进程打开/dev/loopx（在安卓系统中为/dev/block/loopx），其中x为设备编号。
4.进程打开img镜像文件。
5.进程使用ioctl(2)向/dev/loopx发送LOOP_SET_FD，将其与img镜像绑定。
该过程中按照国际惯例用open(2)得到的文件描述符来操作文件。
然后挂载img镜像就很简单了：
```C
int mountimg(const char *img, const char *dir)
{
        // Get a new loopfile for losetup.
        int loopctlfd = open("/dev/loop-control", O_RDWR | O_CLOEXEC);
        // It takes the same effect as `losetup -f`.
        int devnr = ioctl(loopctlfd, LOOP_CTL_GET_FREE);
        close(loopctlfd);
        char loopfile[PATH_MAX] = { '\0' };
        sprintf(loopfile, "/dev/loop%d", devnr);
        int loopfd = open(loopfile, O_RDWR | O_CLOEXEC);
        if (loopfd < 0) {
                // On Android, loopfile is in /dev/block.
                sprintf(loopfile, "/dev/block/loop%d", devnr);
                loopfd = open(loopfile, O_RDWR | O_CLOEXEC);
                if (loopfd < 0) {
                        error("\033[31mError: losetup error!\n");
                }
        }
        // It takes the same efferct as `losetup` command.
        int imgfd = open(img, O_RDWR | O_CLOEXEC);
        ioctl(loopfd, LOOP_SET_FD, imgfd);
        close(loopfd);
        close(imgfd);
        trymount(loopfile, dir);
}
```
