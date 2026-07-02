---
title: 为你的手机内核开启docker支持
date: 2022-12-04 23:42:50
tags:
  - Linux
  - Termux
  - Docker
  - container
cover: /img/docker-lab.jpg
top_img: /img/docker-lab.jpg
---
注意：文章所述方法仅适用于非GKI或qgki的设备，新设备咱也不会搞。
文章内所述手机为arm64架构，上古时期的32位架构请自行修改。      
注：pixel系列设备请换用repo工具以及官方构建工具并使用ThinLTO（在内存小于32G的设备上）。      
好了让我们开始吧喵！      
### 首要前提：
- 手机能够解锁bl并获取root权限      
- 手机内核开源，尽量是有大佬维护源码的
- 拥有一定Linux基础

如果设备或个人不满足以上条件者请自行退出喵！      
### 前期准备：
你可能需要准备如下内容：      
- Linux系统环境(理论上手机电脑均可，电脑最佳)      
- 熟练使用搜索工具      
- git和make以及代码编辑工具的使用      
- 基本了解cpu架构差异     

这些内容咱是不会教你的，毕竟这不是文章重点喵唔……     
当然最好有个脑子，可惜咱没有呜QAQ………      
### 正式操作：
#### 0x0001 root手机，不必多说
#### 0x0002 获取手机代号和cpu代号
这一步请通过搜索工具进行。      
如小米10ultra代号cas，cpu代号SM8250。       
#### 0x0003 查找内核源码
可以去官方仓库，当然建议用第三方的，因为官方内核很多时候还不如第三方好跑呢。       
查找方式：官方仓库查找设备代号或github搜索关键字kernel + 设备代号或者反过来或者搜索cpu代号+手机厂商+kernel，各种组合和命名方式都试过了还找不到的话，多半是没有了，洗洗睡吧。       
#### 0x0004 编译工具选择
手机执行：
```sh
cat /proc/version
```
示例输出：
```
Linux version 4.19.260-Moe-hacker-g0bb1c026ee65-dirty (root@localhost) (Ubuntu clang version 14.0.6-2, GNU ld (GNU Binutils for Ubuntu) 2.39) #3 SMP PREEMPT Sun Oct 2 10:48:46 CST 2022
```
当然咱已经完成内核编译了，仔细观察会发现内核由clang-14编译。      
对应llvm版本也为14。     
于是你确认了要用的编译器版本。     
如果内核是由谷歌的安卓开发工具构建，请自行查找并下载。      
小技巧：使用原系统内核使用的编译器版本可以降低出错概率。      
#### 0x0005 源码获取：
使用git clone项目仓库，如果是官方仓库需要加入-b选项克隆机型独立的分支。      
国内用户访问github不方便的可以换用ssh协议（git clone ssh@github.com:用户or组织名称/代码仓库）      
或者换用镜像站kgithub.com或ghproxy.com等。      
#### 0x0006 依赖安装：
主要依赖有：clang/gcc构建工具,跨架构binutils工具(跨架构编译需要),make,python,libssl-dev,build-essential,bc,bison,flex,unzip,libssl-dev,ca-certificates,xz-utils,mkbootimg,cpio,device-tree-compiler，请自行安装，否则编译会出错。      
编译出现command not found大概率是工具没有安装。      
debian系的系统解决文件缺失推荐`apt-file search`命令。      
#### 0x0007 尝试编译：
进入项目目录
```sh
ls arch/arm64/configs
```
康一康有没有你的机型代号相关的文件，一般是[机型代号]_defcofig，也有带stock或者perf的命名，选一个就行。      
没有的话也不要着急，再看一下vendor目录：      
```sh
ls arch/arm64/configs/vendor
```
桥豆麻袋，还是找不到啊！！！      
github去arch/arm64/configs目录下看看提交记录，最近变更最多的大概率是。比如nothing的三方内核源码配置文件是vendor/lahaina-qgki_defconfig。      
或者根据版本号，三方内核配置中CONFIG_LOCALVERSION值大概率不是默认。      
然后，呐，现在要开始编译了哦喵！      
```sh
export ARCH=arm64
export SUBARCH=arm64
make O=out CC=[clang/gcc-版本号] (vendor/)xxxxxx_defconfig ［可选参数］
```
可选参数详解：
```sh
#非电脑跨架构编译省略
ARCH=arm64
CROSS_COMPILE=aarch64-linux-gnu- CROSS_COMPILE_ARM32=arm-linux-gnueabi-
#基本没用到过，按需开启
AR=llvm-ar-版本号
OBJDUMP=llvm-objdump-版本号
STRIP=llvm-strip-版本号
NM=llvm-nm-版本号
OBJCOPY=llvm-objcopy-版本号
LD=ld.lld-版本号
```
以上可选参数可用于报错处理以及确保llvm工具版本与clang一致，酌情加入。      
然后，其他参数不变，删掉(vendor/)xxxxxx_defconfig这个，改为-j$(nproc)，开始构建内核。      
#### 0x0008 基本异常处理：
找不到头文件：      
安装相应库。      
找不到命令：      
安装相应软件。      
-Werror,xxxxxxx：
找报错的文件相应Makefile,把含有-werror的都删了(每一层目录都有Makefile，建议从报错文件那一层往父目录找)，或者make选项改为CC="[clang/gcc]-版本号 -w"      
未定义函数或其他未定义：      
查找函数定义开启所依赖的配置项一并开启，可能在头文件或kconfig/makefile中。      
最后一步生成vmlinux时报错大概率是因为配置没开全。      
#### 0x0009 玄学异常：
在编译pixel3内核时，咱删了一行源码成功生成内核，开机功能一切正常。      
在编译小米10Ultra内核时一行源码少了一个地址符&，手动添加后一切正常。      
遇到这种不可预知的玄学异常建议动用搜索工具，或者学会放弃。      
引用当年沨鸾在酷安的原文：会修的就修，修不了换源码，换编译器版本，手机电脑换着试，最后放弃就好了。      
如果你跨过了首次编译这道坎，那么恭喜，你离成功不远了喵！      
#### 0x000A 功能开启：
下载check-config.sh
```sh
wget https://github.com/moby/moby/raw/master/contrib/check-config.sh
```
网络不好请使用kgithub镜像站，目前可用。      
然后：
```sh
sh check-config.sh out/.config|grep missing|sed -E 's/\-//g'|sed -E "s/ //g"|sed -r 's/://'|sed -E "s/missing/=y/"
```
咱甚至帮大家写好了字符替换。     
于是你得到了内核未开启的的配置列表。      
```
CONFIG_AUFS_FS=y
/dev/zfs=y
zfscommand=y
zpoolcommand=y
```
以上这几个输出不用管，删了就好，这几个的源码实现均未并入linux4.x主分支。      
然后把缺失的config加入你的(vendor/)xxxxxx_defconfig中，并将里面带有is not set的字样全部删除，执行编译第一步，再次生成配置。      
这一步你可以更改local version值为你的名字或者你喜欢的单词。      
再次执行扫描命令，获取缺失项目。      
使用make menuconfig命令，按下/键搜索缺失项目的依赖与冲突，依赖添加开启选项，冲突关闭。
注意：menuconfig配置默认不带CONFIG_头，需要手动添加。      
然后，将配置中所有=m替换为=y，目的是将内核模块built-in。      
请确认最终生成out/.config中不包含=m字样。      
#### 0x000B 再次编译：
请自行repeat上文所述编译步骤。      
生成文件在out/arch/arm64/boot/目录下，大部分命名为Image.xxx-dtb，但是注意，少数机型只能刷入Image.xx格式镜像。      
#### 0x000C 验证config： 
scripts目录下有个extract-ikconfig，用它把Image的配置扫出来输出到一个文件，check-config.sh除上文所讲述的无法开启的配置全绿即可。      
如果遇到内核config和out/.config内容不一致，查找kernel/Makefile，找到\$(obj)/config_data.gz:xxxxxxxxxx，把xxxxxxxx改成\$(obj)/config_data      
#### 0x000D 刷入：
下载刷入工具：
```sh
git clone https://github.com/osm0sis/AnyKernel3
```
编辑anykernel.sh，修改如下内容：      
```
device.name1=设备代号      
block=/dev/block/bootdevice/by-name/boot;      
is_slot_device=如果是ab架构分区设备填1，否则填0      
```
将Image.xxx-dtb复制到anykernel根目录下，打包anykernel根目录，twrp刷入。      
注意，少数机型只能刷入Image.xx格式镜像。      
于是你就到了最终环节：开机，验证。      
教程完毕，相信你也可以在手机上运行自己的内核了喵！      
### 基本报错处理：
如果报错如下：      
```log
docker: Error response from daemon: OCI 
runtime create failed: container_linux.go:370: starting 
container process caused: process_linux.go:326: applying 
cgroup configuration for process caused: mountpoint for 
devices not found: unknown. 
``` 
那么您需要手动挂载cgroupfs(root权限执行):      
```sh
mount -t tmpfs -o mode=755 tmpfs /sys/fs/cgroup
mkdir -p /sys/fs/cgroup/devices
mount -t cgroup -o devices cgroup /sys/fs/cgroup/devices
```
然后，重启docker即可。       
咱自己没有遇到过的两个异常解决方式：      
添加--iptables=false参数      
设置DOCKER_RAMDISK=true      
容器中没网可通过在容器中执行以下脚本解决root用户联网问题：       
```sh
# Fix networking and other permission issues on Android.
# From Tmoe: https://github.com/2moe/Tmoe
groupadd aid_system -g 1000 || groupadd aid_system -g 1074
groupadd aid_radio -g 1001
groupadd aid_bluetooth -g 1002
groupadd aid_graphics -g 1003
groupadd aid_input -g 1004
groupadd aid_audio -g 1005
groupadd aid_camera -g 1006
groupadd aid_log -g 1007
groupadd aid_compass -g 1008
groupadd aid_mount -g 1009
groupadd aid_wifi -g 1010
groupadd aid_adb -g 1011
groupadd aid_install -g 1012
groupadd aid_media -g 1013
groupadd aid_dhcp -g 1014
groupadd aid_sdcard_rw -g 1015
groupadd aid_vpn -g 1016
groupadd aid_keystore -g 1017
groupadd aid_usb -g 1018
groupadd aid_drm -g 1019
groupadd aid_mdnsr -g 1020
groupadd aid_gps -g 1021
groupadd aid_media_rw -g 1023
groupadd aid_mtp -g 1024
groupadd aid_drmrpc -g 1026
groupadd aid_nfc -g 1027
groupadd aid_sdcard_r -g 1028
groupadd aid_clat -g 1029
groupadd aid_loop_radio -g 1030
groupadd aid_media_drm -g 1031
groupadd aid_package_info -g 1032
groupadd aid_sdcard_pics -g 1033
groupadd aid_sdcard_av -g 1034
groupadd aid_sdcard_all -g 1035
groupadd aid_logd -g 1036
groupadd aid_shared_relro -g 1037
groupadd aid_dbus -g 1038
groupadd aid_tlsdate -g 1039
groupadd aid_media_ex -g 1040
groupadd aid_audioserver -g 1041
groupadd aid_metrics_coll -g 1042
groupadd aid_metricsd -g 1043
groupadd aid_webserv -g 1044
groupadd aid_debuggerd -g 1045
groupadd aid_media_codec -g 1046
groupadd aid_cameraserver -g 1047
groupadd aid_firewall -g 1048
groupadd aid_trunks -g 1049
groupadd aid_nvram -g 1050
groupadd aid_dns -g 1051
groupadd aid_dns_tether -g 1052
groupadd aid_webview_zygote -g 1053
groupadd aid_vehicle_network -g 1054
groupadd aid_media_audio -g 1055
groupadd aid_media_video -g 1056
groupadd aid_media_image -g 1057
groupadd aid_tombstoned -g 1058
groupadd aid_media_obb -g 1059
groupadd aid_ese -g 1060
groupadd aid_ota_update -g 1061
groupadd aid_automotive_evs -g 1062
groupadd aid_lowpan -g 1063
groupadd aid_hsm -g 1064
groupadd aid_reserved_disk -g 1065
groupadd aid_statsd -g 1066
groupadd aid_incidentd -g 1067
groupadd aid_secure_element -g 1068
groupadd aid_lmkd -g 1069
groupadd aid_llkd -g 1070
groupadd aid_iorapd -g 1071
groupadd aid_gpu_service -g 1072
groupadd aid_network_stack -g 1073
groupadd aid_shell -g 2000
groupadd aid_cache -g 2001
groupadd aid_diag -g 2002
groupadd aid_oem_reserved_start -g 2900
groupadd aid_oem_reserved_end -g 2999
groupadd aid_net_bt_admin -g 3001
groupadd aid_net_bt -g 3002
groupadd aid_inet -g 3003
groupadd aid_net_raw -g 3004
groupadd aid_net_admin -g 3005
groupadd aid_net_bw_stats -g 3006
groupadd aid_net_bw_acct -g 3007
groupadd aid_readproc -g 3009
groupadd aid_wakelock -g 3010
groupadd aid_uhid -g 3011
groupadd aid_everybody -g 9997
groupadd aid_misc -g 9998
groupadd aid_nobody -g 9999
groupadd aid_app_start -g 10000
groupadd aid_app_end -g 19999
groupadd aid_cache_gid_start -g 20000
groupadd aid_cache_gid_end -g 29999
groupadd aid_ext_gid_start -g 30000
groupadd aid_ext_gid_end -g 39999
groupadd aid_ext_cache_gid_start -g 40000
groupadd aid_ext_cache_gid_end -g 49999
groupadd aid_shared_gid_start -g 50000
groupadd aid_shared_gid_end -g 59999
groupadd aid_isolated_start -g 99000
groupadd aid_isolated_end -g 99999
groupadd aid_user_offset -g 100000
# Fix root user's permissions.
usermod -a -G aid_system,aid_radio,aid_bluetooth,aid_graphics,aid_input,aid_audio,aid_camera,aid_log,aid_compass,aid_mount,aid_wifi,aid_adb,aid_install,aid_media,aid_dhcp,aid_sdcard_rw,aid_vpn,aid_keystore,aid_usb,aid_drm,aid_mdnsr,aid_gps,aid_media_rw,aid_mtp,aid_drmrpc,aid_nfc,aid_sdcard_r,aid_clat,aid_loop_radio,aid_media_drm,aid_package_info,aid_sdcard_pics,aid_sdcard_av,aid_sdcard_all,aid_logd,aid_shared_relro,aid_dbus,aid_tlsdate,aid_media_ex,aid_audioserver,aid_metrics_coll,aid_metricsd,aid_webserv,aid_debuggerd,aid_media_codec,aid_cameraserver,aid_firewall,aid_trunks,aid_nvram,aid_dns,aid_dns_tether,aid_webview_zygote,aid_vehicle_network,aid_media_audio,aid_media_video,aid_media_image,aid_tombstoned,aid_media_obb,aid_ese,aid_ota_update,aid_automotive_evs,aid_lowpan,aid_hsm,aid_reserved_disk,aid_statsd,aid_incidentd,aid_secure_element,aid_lmkd,aid_llkd,aid_iorapd,aid_gpu_service,aid_network_stack,aid_shell,aid_cache,aid_diag,aid_oem_reserved_start,aid_oem_reserved_end,aid_net_bt_admin,aid_net_bt,aid_inet,aid_net_raw,aid_net_admin,aid_net_bw_stats,aid_net_bw_acct,aid_readproc,aid_wakelock,aid_uhid,aid_everybody,aid_misc,aid_nobody,aid_app_start,aid_app_end,aid_cache_gid_start,aid_cache_gid_end,aid_ext_gid_start,aid_ext_gid_end,aid_ext_cache_gid_start,aid_ext_cache_gid_end,aid_shared_gid_start,aid_shared_gid_end,aid_isolated_start,aid_isolated_end,aid_user_offset root
# Fix apt.
usermod -g aid_inet _apt 2>/dev/null
# Fix gentoo emerge.
usermod -a -G aid_inet,aid_net_raw portage 2>/dev/null
```
# 后记：
没有后记，散会！
