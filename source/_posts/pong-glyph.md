---
title: Nothing Phone(2)的灯带驱动研究笔记
date: 2024-01-10 00:37:43
tags:
  - Linux
cover: /img/pong-glyph.jpg
top_img: /img/pong-glyph.jpg
---
最近整了部Nothing Phone(2)，bl秒解的设定是真的舒服，所以买来第一时间就透了一遍（指root了）。
然后半夜睡不着，就打算研究一下这个灯带是怎么调用的。
然后就开始了，
一段孤独的旅程充满烦恼～
# 内核源码：
很不幸，除了知道了灯带型号是aw20036之外没啥收获，原因无他，单纯看不懂代码，注释都不怎么写这不欺负萌新吗。。。
仓库里相关代码文件甚至具有可执行权限，看来开发者也是和咱一样只知道无脑777的杂鱼呢。
在leds目录下有个TODO文件(这是忘了配置gitignore吗？)，然后看到一段离谱的留言：
```
* Command line utility to manipulate the LEDs?

/sys interface is not really suitable to use by hand, should we have
an utility to perform LED control?
```
好家伙，你还有脸说呢！！！
你倒是说说具体怎么用啊！！！
# 正式尝试：
## 找到接口位置：
好吧内核源码看不懂，咱来到user-space（笑）。
首先find|grep找到sysfs接口在`/sys/bus/i2c/drivers/aw20036_led/0-003a/leds/led_strips/`
（我不管我就要find配grep！！！）
我们进去：
```
.../leds/led_strips # cd /sys/bus/i2c/drivers/aw20036_led/0-003a/leds/led_strips/
.../leds/led_strips # ls
all_brightness        imax
all_white_brightness  max_brightness
always_on             operating_mode
brightness            power
dev_color             reg
device                single_brightness
effect                subsystem
factory_test          trigger
frame_brightness      uevent
hwen                  vip_notification
hwid
```
好吧这不是我熟悉的灯带驱动。
跟一代一点也不一样，或许不是同一个人写的。。。
## 尝试调用：
最后找到的方法不是一般人能想出来的，所以略过。
比较有意思的是`factory_test`这个文件，看名字用于工厂测试？不记得这灯品控差啊。不过和`all_brightness`作用几乎没太大差距。
## 查找系统中的调用方式：
`ps -A`大法没任何发现，遂使用`logcat`大法。
```
.../leds/led_strips # logcat|grep aw20036
（省略）
01-10 01:06:38.195  1481  1481 I aw20036_operating_mode_store: 1
01-10 01:06:38.198  1481  1481 I         : aw20036_ioctl LED_STRIPS_ALWAYS_ON
01-10 01:06:38.198  1481  1481 I aw20036_ioctl aw20036->always_on: 1
01-10 01:06:38.198  1481  1481 I aw20036_all_white_brightness_store: 160
01-10 01:06:39.014  1481  1481 I         : aw20036_ioctl LED_STRIPS_ALWAYS_ON
01-10 01:06:39.014  1481  1481 I aw20036_ioctl aw20036->always_on: 0
01-10 01:06:39.014  1481  1481 I aw20036_all_white_brightness_store: 0
01-10 01:06:39.517  1519  1519 I aw20036_operating_mode_store: 2
```
开关了下Glyph torch发现新增上述日志，看来灯带是pid 1481管的。
至于这个1481是什么呢？
```
.../leds/led_strips # cat /proc/1481/cmdline
/vendor/bin/hw/vendor.qti.hardware.lights.service
```
看来是和高通py的～
《和高通联合调教的灯带》
（所以发布会为啥不吹一吹。。。）
然后直接上strace，去Glyph composer里随便点一下，反正让灯闪一下就是了，得到如下日志：
```c
.../leds/led_strips # strace -s 114514 -v -p 1481
strace: Process 1481 attached
ioctl(3, BINDER_WRITE_READ, 0x7fcb957048) = 0
getuid()                                = 1000
writev(4, [{iov_base="\0\311\5Q~\235e\341\356\321\r", iov_len=11}, {iov_base="\3", iov_len=1}, {iov_base="LightsExt\0", iov_len=10}, {iov_base="setLightFrame enter id:110 state:0x1 brightness:160 colorSize:33\n\0", iov_len=66}], 4) = 88
openat(AT_FDCWD, "/sys/devices/platform/soc/994000.i2c/i2c-0/0-003a/leds/led_strips/frame_brightness", O_WRONLY) = 15
openat(AT_FDCWD, "/sys/devices/platform/soc/994000.i2c/i2c-0/0-003a/leds/led_strips/operating_mode", O_RDONLY) = 16
read(16, "2", 1)                        = 1
close(16)                               = 0
getuid()                                = 1000
writev(4, [{iov_base="\0\311\5Q~\235eT\377\355\r", iov_len=11}, {iov_base="\3", iov_len=1}, {iov_base="LightsExt\0", iov_len=10}, {iov_base="setLightFrame mode:2\n\0", iov_len=22}], 4) = 44
openat(AT_FDCWD, "/sys/devices/platform/soc/994000.i2c/i2c-0/0-003a/leds/led_strips/operating_mode", O_WRONLY) = 16
write(16, "1\n", 2)                     = 2
close(16)                               = 0
write(15, "0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 ", 66) = 66
close(15)                               = 0
ioctl(3, BINDER_WRITE_READ, 0x7fcb957048) = 0
getuid()                                = 1000
writev(4, [{iov_base="\0\311\5Q~\235e \253\335\16", iov_len=11}, {iov_base="\3", iov_len=1}, {iov_base="LightsExt\0", iov_len=10}, {iov_base="setLightFrame enter id:110 state:0x1 brightness:160 colorSize:33\n\0", iov_len=66}], 4) = 88
openat(AT_FDCWD, "/sys/devices/platform/soc/994000.i2c/i2c-0/0-003a/leds/led_strips/frame_brightness", O_WRONLY) = 15
openat(AT_FDCWD, "/sys/devices/platform/soc/994000.i2c/i2c-0/0-003a/leds/led_strips/operating_mode", O_RDONLY) = 16
read(16, "1", 1)                        = 1
close(16)                               = 0
getuid()                                = 1000
writev(4, [{iov_base="\0\311\5Q~\235e\23\6\357\16", iov_len=11}, {iov_base="\3", iov_len=1}, {iov_base="LightsExt\0", iov_len=10}, {iov_base="setLightFrame mode:1\n\0", iov_len=22}], 4) = 44
write(15, "0 148 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 ", 68) = 68
close(15)                               = 0
```
strace默认会折叠代码为省略号，所以需要手动设置最大长度（众所周知114514是一个经典大幻数）。
这样一来灯的调用方式一目了然了：
```sh
echo 1 > /sys/devices/platform/soc/994000.i2c/i2c-0/0-003a/leds/led_strips/operating_mode
printf "0 148 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 " > /sys/devices/platform/soc/994000.i2c/i2c-0/0-003a/leds/led_strips/frame_brightness
```
果然，摄像头左下角的灯亮了。
## 大胆假设：
点灯时先把`operating_mode`设置成1
这一串编码一样的东西代表了每个灯的亮度，更改每一位的数值可修改相应灯的亮度。
最大亮度依旧是`max_brightness`（255）
## 代码验证：
C语言，启动！
```C
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <fcntl.h>
#include <sys/stat.h>
#include <sys/types.h>
#define operating_mode "/sys/devices/platform/soc/994000.i2c/i2c-0/0-003a/leds/led_strips/operating_mode"
#define frame_brightness "/sys/devices/platform/soc/994000.i2c/i2c-0/0-003a/leds/led_strips/frame_brightness"
void write_leds(short leds[33])
{
        int fd = open(operating_mode, O_RDWR);
        write(fd, "1\n", 2);
        close(fd);
        char buf[8] = { '\0' };
        char to_write[128] = { '\0' };
        for (int i = 0; i < 33; i++) {
                sprintf(buf, "%d ", leds[i]);
                strcat(to_write, buf);
        }
        fd = open(frame_brightness, O_WRONLY);
        write(fd, to_write, strlen(to_write));
        close(fd);
}
int main(void)
{
        short leds[33] = { 0 };
        for (int i = 0; i <= 33; i++) {
                printf("\r\033[34mLED \033[32m%2d \033[34m ON\033[0m\a", i);
                fflush(stdout);
                memset(leds, 0, sizeof(leds));
                leds[i] = 114;
                write_leds(leds);
                sleep(1);
        }
        printf("\n");
}
```
代码还是非常简单的，运行也没啥问题，但是跑完一遍发现又有了新的问题：视频指示灯还没亮过啊！
## 继续研究:
strace追踪1481进程，发现居然不是这个进程调用的灯带：
```C
.../files/home # strace -s 114514 -v -p 1481
strace: Process 1481 attached
ioctl(3, BINDER_WRITE_READ, 0x7fcb957048) = 0
getuid()                                = 1000
writev(4, [{iov_base="\0\311\5\1a\236eM\375\343\3", iov_len=11}, {iov_base="\3", iov_len=1}, {iov_base="LightsExt\0", iov_len=10}, {iov_base="setLightExtensionState enter id:105, state:0x1, brightness 2570\n\0", iov_len=65}], 4) = 87
openat(AT_FDCWD, "/sys/devices/platform/soc/994000.i2c/i2c-0/0-003a/leds/led_strips/operating_mode", O_RDONLY) = 21
read(21, "2", 1)                        = 1
close(21)                               = 0
getuid()                                = 1000
writev(4, [{iov_base="\0\311\5\1a\236e/W\7\4", iov_len=11}, {iov_base="\3", iov_len=1}, {iov_base="LightsExt\0", iov_len=10}, {iov_base="setLightExtensionState mode:2\n\0", iov_len=31}], 4) = 53
openat(AT_FDCWD, "/sys/devices/platform/soc/994000.i2c/i2c-0/0-003a/leds/led_strips/operating_mode", O_WRONLY) = 21
write(21, "1\n", 2)                     = 2
close(21)                               = 0
getuid()                                = 1000
writev(4, [{iov_base="\0\311\5\1a\236e\247\370.\4", iov_len=11}, {iov_base="\3", iov_len=1}, {iov_base="LightsExt\0", iov_len=10}, {iov_base="set_light_video enter on:0x1 cvsId:0x0\n\0", iov_len=40}], 4) = 62
futex(0x5e0e9ea6e8, FUTEX_WAKE_PRIVATE, 2147483647) = 1
ioctl(3, BINDER_WRITE_READ, 0x7fcb957048) = 0
getuid()                                = 1000
writev(4, [{iov_base="\0\311\5\1a\236eF\264\262\22", iov_len=11}, {iov_base="\3", iov_len=1}, {iov_base="LightsExt\0", iov_len=10}, {iov_base="setLightExtensionState enter id:105, state:0x0, brightness 2570\n\0", iov_len=65}], 4) = 87
getuid()                                = 1000
writev(4, [{iov_base="\0\311\5\1a\236e\306$\272\22", iov_len=11}, {iov_base="\3", iov_len=1}, {iov_base="LightsExt\0", iov_len=10}, {iov_base="set_light_video enter on:0x0 cvsId:0x0\n\0", iov_len=40}], 4) = 62
futex(0x5e0e9ea6f8, FUTEX_WAIT_BITSET_PRIVATE, 4294967294, NULL, FUTEX_BITSET_MATCH_ANY) = 0
futex(0x5e0e9ea6a8, FUTEX_WAKE_PRIVATE, 2147483647) = 1
ioctl(3, BINDER_WRITE_READ
```
那就logcat一下：
```
.../files/home # logcat|grep 1481
01-10 17:21:01.318  1481  1481 D LightsExt: setLightExtensionState enter id:105, state:0x1, brightness 2570
01-10 17:21:01.318  1481  1481 D LightsExt: setLightExtensionState mode:2
01-10 17:21:01.100  1481  1481 I aw20036_operating_mode_store: 1
01-10 17:21:01.320  1481  1481 D LightsExt: set_light_video enter on:0x1 cvsId:0x0
01-10 17:21:01.320  1481  1521 D LightsExt: Lights_video_effect_thread ready play
01-10 17:21:01.320  1481  1521 D LightsExt: Lights_video_effect_thread brightness:140
01-10 17:21:02.328  1481  1521 D LightsExt: Lights_video_effect_thread brightness:140
01-10 17:21:02.454  1481  1481 D LightsExt: setLightExtensionState enter id:105, state:0x0, brightness 2570
01-10 17:21:02.454  1481  1481 D LightsExt: set_light_video enter on:0x0 cvsId:0x0
01-10 17:21:02.470  1481  1521 D LightsExt: Lights_video_effect_thread break
01-10 17:21:02.470  1481  1521 D LightsExt: Lights_video_effect_thread break
01-10 17:21:02.471  1481  1519 D LightsExt: Lights_power_thread ready
01-10 17:21:02.471  1481  1519 D LightsExt: Lights_power_thread mode:1
01-10 17:21:02.471  1481  1519 D LightsExt: Lights_power_thread start
01-10 17:21:02.471  1481  1521 D LightsExt: Lights_video_effect_thread wait
01-10 17:21:02.971  1481  1519 D LightsExt: Lights_power_thread end
01-10 17:21:02.972  1481  1519 D LightsExt: Lights_power_thread wait
```
看来是进程1521在调用，我们追踪它：
```C
.../files/home # strace -s 114514 -v -p 1521
strace: Process 1521 attached
futex(0x5e0e9ea6e8, FUTEX_WAIT_BITSET_PRIVATE, 4294967294, NULL, FUTEX_BITSET_MATCH_ANY) = 0
getuid()                                = 1000
writev(4, [{iov_base="\0\361\5\24b\236e\323p\3173", iov_len=11}, {iov_base="\3", iov_len=1}, {iov_base="LightsExt\0", iov_len=10}, {iov_base="Lights_video_effect_thread ready play\n\0", iov_len=39}], 4) = 61
openat(AT_FDCWD, "/sys/devices/platform/soc/994000.i2c/i2c-0/0-003a/leds/led_strips/single_brightness", O_WRONLY) = 21
getuid()                                = 1000
writev(4, [{iov_base="\0\361\5\24b\236e\241g\3323", iov_len=11}, {iov_base="\3", iov_len=1}, {iov_base="LightsExt\0", iov_len=10}, {iov_base="Lights_video_effect_thread brightness:140\n\0", iov_len=43}], 4) = 65
write(21, "13 140", 6)                  = 6
nanosleep({tv_sec=0, tv_nsec=20000000}, NULL) = 0
nanosleep({tv_sec=0, tv_nsec=20000000}, NULL) = 0
nanosleep({tv_sec=0, tv_nsec=20000000}, NULL) = 0
nanosleep({tv_sec=0, tv_nsec=20000000}, NULL) = 0
nanosleep({tv_sec=0, tv_nsec=20000000}, NULL) = 0
nanosleep({tv_sec=0, tv_nsec=20000000}, NULL) = 0
getuid()                                = 1000
writev(4, [{iov_base="\0\361\5\24b\236e\312\2428;", iov_len=11}, {iov_base="\3", iov_len=1}, {iov_base="LightsExt\0", iov_len=10}, {iov_base="Lights_video_effect_thread break \n\0", iov_len=35}], 4) = 57
write(21, "13 0", 4)                    = 4
getuid()                                = 1000
writev(4, [{iov_base="\0\361\5\24b\236e\327\37`;", iov_len=11}, {iov_base="\3", iov_len=1}, {iov_base="LightsExt\0", iov_len=10}, {iov_base="Lights_video_effect_thread break \n\0", iov_len=35}], 4) = 57
write(21, "13 0", 4)                    = 4
futex(0x5e0e9ea6f8, FUTEX_WAKE_PRIVATE, 2147483647) = 1
close(21)                               = 0
getuid()                                = 1000
writev(4, [{iov_base="\0\361\5\24b\236e\227m\223;", iov_len=11}, {iov_base="\3", iov_len=1}, {iov_base="LightsExt\0", iov_len=10}, {iov_base="Lights_video_effect_thread wait\n\0", iov_len=33}], 4) = 55
futex(0x5e0e9ea6e8, FUTEX_WAIT_BITSET_PRIVATE, 4294967294, NULL, FUTEX_BITSET_MATCH_ANY
```
然后无语的事来了，居然是和一代一样的接口。。。
原来之前尝试的时候是没有更改operating_mode的值，导致操作不生效。
## 继续写代码：
```C
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <fcntl.h>
#include <sys/stat.h>
#include <sys/types.h>
#define operating_mode "/sys/devices/platform/soc/994000.i2c/i2c-0/0-003a/leds/led_strips/operating_mode"
#define single_brightness "/sys/devices/platform/soc/994000.i2c/i2c-0/0-003a/leds/led_strips/single_brightness"
void write_single_led(short led, u_int brightness)
{
        int fd = open(operating_mode, O_RDWR);
        write(fd, "1\n", 2);
        close(fd);
        char buf[16] = { '\0' };
        sprintf(buf, "%d %d", led, brightness);
        fd = open(single_brightness, O_WRONLY);
        write(fd, buf, strlen(buf));
        close(fd);
}
int main(void)
{
        for (int i = 0; i <= 33; i++) {
                printf("\r\033[34mLED \033[32m%2d \033[34m ON\033[0m\a", i);
                fflush(stdout);
                write_single_led(i, 114);
                sleep(1);
                write_single_led(i, 0);
        }
        printf("\n");
}
```
在接口single_brightness下，灯带的编号居然是乱的！！！
Nothing：我去除了大部分上一代的代码，但是我保留了一部分，只有这样你才知道自己品鉴的是雪。
已经品鉴的够多的了，快点端下去罢（悲）。
# 总结：
总结就是没啥好总结的，下期再见吧。
