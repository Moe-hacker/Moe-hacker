---
layout: post
title: 记录为Nothing Phone(2)构建LXC内核
date: 2025-07-13 23:26:17
tags:
- Android
- Kernel
- LXC
- Linux
cover: /img/pong-lxc.jpg
top_img: /img/pong-lxc.jpg
---
# 参考源码：
## Patch：
https://github.com/lateautumn233/Common-Android-Kernel-Tree
## Kernel：
https://github.com/arter97/android_kernel_nothing_sm8475
## Might useful：
https://github.com/xiaoleGun/KernelSU_Action

# 构建：
和我之前的文章几乎差不多，可以看一下[为你的手机内核开启docker支持](https://blog.crack.moe/2022/12/04/moe-docker-lab/)。
pick了参考源码打的补丁,defconfig不要pick，可以参考：
```diff
diff --git a/defconfig b/defconfig
index ac2b801694e6..23139c0181e9 100644
--- a/defconfig
+++ b/defconfig
@@ -26,7 +26,7 @@ CONFIG_THREAD_INFO_IN_TASK=y
 CONFIG_INIT_ENV_ARG_LIMIT=32
 # CONFIG_COMPILE_TEST is not set
 CONFIG_WERROR=y
-CONFIG_LOCALVERSION="-arter97-'$(cat version)'"
+CONFIG_LOCALVERSION="-nekofeng"
 CONFIG_LOCALVERSION_AUTO=y
 CONFIG_UNAME_OVERRIDE=y
 CONFIG_UNAME_OVERRIDE_TARGET="com.google.android.gms"
@@ -35,8 +35,8 @@ CONFIG_BUILD_SALT=""
 CONFIG_DEFAULT_INIT=""
 CONFIG_DEFAULT_HOSTNAME="(none)"
 CONFIG_SWAP=y
-# CONFIG_SYSVIPC is not set
-# CONFIG_POSIX_MQUEUE is not set
+CONFIG_SYSVIPC
+CONFIG_POSIX_MQUEUE
 # CONFIG_WATCH_QUEUE is not set
 CONFIG_CROSS_MEMORY_ATTACH=y
 # CONFIG_USELIB is not set
@@ -172,7 +172,7 @@ CONFIG_UCLAMP_TASK_GROUP=y
 CONFIG_CGROUP_FREEZER=y
 CONFIG_CPUSETS=y
 CONFIG_PROC_PID_CPUSET=y
-# CONFIG_CGROUP_DEVICE is not set
+CONFIG_CGROUP_DEVICE=y
 CONFIG_CGROUP_CPUACCT=y
 # CONFIG_CGROUP_PERF is not set
 CONFIG_CGROUP_BPF=y
@@ -181,8 +181,8 @@ CONFIG_SOCK_CGROUP_DATA=y
 CONFIG_NAMESPACES=y
 CONFIG_UTS_NS=y
 CONFIG_TIME_NS=y
-# CONFIG_USER_NS is not set
-# CONFIG_PID_NS is not set
+CONFIG_USER_NS=y
+CONFIG_PID_NS=y
 CONFIG_NET_NS=y
 # CONFIG_CHECKPOINT_RESTORE is not set
 # CONFIG_SCHED_AUTOGROUP is not set
@@ -1218,7 +1218,7 @@ CONFIG_NETFILTER_XT_CONNMARK=y
 #
 # Xtables targets
 #
-# CONFIG_NETFILTER_XT_TARGET_CHECKSUM is not set
+CONFIG_NETFILTER_XT_TARGET_CHECKSUM=y
 CONFIG_NETFILTER_XT_TARGET_CLASSIFY=y
 CONFIG_NETFILTER_XT_TARGET_CONNMARK=y
 CONFIG_NETFILTER_XT_TARGET_CONNSECMARK=y
@@ -1248,7 +1248,7 @@ CONFIG_NETFILTER_XT_TARGET_TCPMSS=y
 #
 # Xtables matches
 #
-# CONFIG_NETFILTER_XT_MATCH_ADDRTYPE is not set
+CONFIG_NETFILTER_XT_MATCH_ADDRTYPE=y
 CONFIG_NETFILTER_XT_MATCH_BPF=y
 # CONFIG_NETFILTER_XT_MATCH_CGROUP is not set
 # CONFIG_NETFILTER_XT_MATCH_CLUSTER is not set
@@ -1360,7 +1360,8 @@ CONFIG_IP6_NF_TARGET_REJECT=y
 CONFIG_IP6_NF_MANGLE=y
 CONFIG_IP6_NF_RAW=y
 # CONFIG_IP6_NF_SECURITY is not set
-# CONFIG_IP6_NF_NAT is not set
+CONFIG_IP6_NF_NAT=y
+CONFIG_IP6_TARGET_MASQUERADE=y
 # end of IPv6: Netfilter Configuration
 
 CONFIG_NF_DEFRAG_IPV6=y
@@ -1632,7 +1633,7 @@ CONFIG_BT_HCIUART_QCA=y
 # CONFIG_BT_HCIBCM203X is not set
 # CONFIG_BT_HCIBPA10X is not set
 # CONFIG_BT_HCIBFUSB is not set
-# CONFIG_BT_HCIVHCI is not set
+CONFIG_BT_HCIVHCI=y
 # CONFIG_BT_MRVL is not set
 # CONFIG_BT_MTKUART is not set
 CONFIG_MSM_BT_POWER=m
@@ -1819,7 +1820,7 @@ CONFIG_PCI_ENDPOINT=y
 # Generic Driver Options
 #
 # CONFIG_UEVENT_HELPER is not set
-# CONFIG_DEVTMPFS is not set
+CONFIG_DEVTMPFS=y
 CONFIG_STANDALONE=y
 CONFIG_PREVENT_FIRMWARE_BUILD=y
 
@@ -2836,7 +2837,7 @@ CONFIG_SERIO_LIBPS2=y
 # Character devices
 #
 CONFIG_TTY=y
-# CONFIG_VT is not set
+CONFIG_VT=y
 CONFIG_UNIX98_PTYS=y
 # CONFIG_LEGACY_PTYS is not set
 CONFIG_LDISC_AUTOLOAD=y
@@ -2905,7 +2906,7 @@ CONFIG_SERIAL_MCTRL_GPIO=y
 # CONFIG_SERIAL_NONSTANDARD is not set
 # CONFIG_N_GSM is not set
 # CONFIG_NOZOMI is not set
-# CONFIG_NULL_TTY is not set
+CONFIG_NULL_TTY=y
 # CONFIG_TRACE_SINK is not set
 CONFIG_HVC_DRIVER=y
 CONFIG_HVC_DCC=y
```
如何下载补丁呢？
github上commit链接后面加.patch或者.diff就行了。
附上构建命令：
```shell
cp defconfig arch/arm64/configs/gki_defconfig
make ARCH=arm64 CC=clang-20 HOSTCC=clang-20 LD=ld.lld-20 HOSTLD=ld.lld-20 OBJCOPY=llvm-objcopy-20 OBJDUMP=llvm-objdump-20 AR=llvm-ar-20 NM=llvm-nm-20 AS=llvm-as-20 STRIP=llvm-strip-20 gki_defconfig
make ARCH=arm64 CC=clang-20 HOSTCC=clang-20 LD=ld.lld-20 HOSTLD=ld.lld-20 OBJCOPY=llvm-objcopy-20 OBJDUMP=llvm-objdump-20 AR=llvm-ar-20 NM=llvm-nm-20 AS=llvm-as-20 STRIP=llvm-strip-20 -j$(nproc --all)
```
# 运行：
记得挂载cgroup，否则会报错。
```shell
mount -t tmpfs -o mode=755 tmpfs /sys/fs/cgroup
sudo mkdir -p /sys/fs/cgroup/devices
sudo mount -t cgroup -o devices cgroup /sys/fs/cgroup/devices
sudo mkdir -p /sys/fs/cgroup/systemd
sudo mount -t cgroup cgroup -o none,name=systemd /sys/fs/cgroup/systemd
```
如果没网，把lxc.net.0.type改成none就好了。
# 鸣谢：
感谢[Arter97](https://github.com/arter97)的内核源码，感谢[lateautumn233](https://github.com/lateautumn233)提供的源码中的补丁参考。    
# Prebuild：
A prebuilt kernel is available at https://github.com/Moe-sushi/misc/blob/main/np2/magisk_patched-28100_cKsqi.img.   
Tested on NothingOS 3.0 Android 15, works fine.    
~~等下我怎么突然说嘤语了~~