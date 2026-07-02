---
title: 在arm64设备上使用qemu(kvm)运行aarch64 ubuntu虚拟机
date: 2024-03-15 13:01:13
tags:
  - Linux
  - Termux
top_img: /img/kvm.jpg
cover: /img/kvm.jpg
---

最近咱换了k40外观增强版，这一代联发科芯片漏洞不是一般的多，不仅有mtkclient中众所周知的bootrom漏洞，老版本系统lk中的cpu地址还是错的，真是“红米配天玑，越用越懵逼”。
不过lk的漏洞导致这手机在miui12.5下是能开kvm的，嗯，不折腾会死星人狂喜。
所以就有了这篇通用的在支持kvm的arm64设备上运行ubuntu虚拟机的文章，好了咱们开始：
首先创建一个img镜像并格式化为ext4：
```
dd if=/dev/zero of=ubuntu.img bs=1G count=16 status=progress
mkfs.ext4 ubuntu.img
```
然后把它挂载：
```
LOOP_FILE=$(losetup -f)
losetup $LOOP_FILE ubuntu.img
mkdir ubuntu
mount $LOOP_FILE ubuntu
```
然后咱还需要一个rootfs，去
[lxc mirror](https://mirrors.bfsu.edu.cn/lxc-images/images/ubuntu/)
可以找到你需要的版本的rootfs.tar.xz
下载后将rootfs解压：
```
tar -xvf rootfs.tar.xz -C ubuntu
```
然后你需要把这个系统作为容器chroot进去，可以使用咱写的[ruri](https://github.com/Moe-hacker/ruri):
```
LD_PRELOAD= ruri ./ubuntu
```
进去后，安装必要软件 systemd net-tools linux-firmware networkmanager dhcpcd linux-image-virtual xterm ，大概就这几个。
然后记得passwd设置好root密码
这时候你应该能在/boot下找到vmlinuz和initrd.img
把它们分别复制为镜像同目录下的linux和initrd文件
然后我们正式启动qemu:
```
taskset -c 0-3 qemu-system-aarch64 \
    -machine virt --enable-kvm \
    -nographic \
    -m size=1024M \
    -cpu host -smp 4 \
    -net user -net nic,model=virtio \
    -drive format=raw,file=ubuntu.img,if=virtio \
    -kernel linux -initrd initrd \
    -append "root=/dev/vda rw"
```
参数注释版本（不可执行）：
```
#使用cpu的0-3核心，不使用taskset强制绑核会导致只有单核
#实测k40g只有小核能成功启动，其他编号不行,会导致cpu跑路。
taskset -c 0-3 qemu-system-aarch64 \
# 开启虚拟化支持
    -machine virt --enable-kvm \
# 输出到终端,serial stdio会导致CTRL-C无法正确传入
    -nographic \
# 内存大小
    -m size=1024M \
# 使用宿主CPU，4核心
    -cpu host -smp 4 \
# 设置网络
    -net user -net nic,model=virtio \
# 系统镜像
    -drive format=raw,file=ubuntu.img,if=virtio \
# 内核和initrd
    -kernel linux -initrd initrd \
# 内核cmdline
    -append "root=/dev/vda rw"
```
这时候不出意外就能启动了，kvm的速度还是非常可以的，舒服的很。
启动后，终端大小是错的，可以执行resize命令重新设置大小，记得将它加入自启脚本：
```
echo "resize > /dev/null" >> ~/.profile
```
另外，默认终端不会显示彩色，需要手动将$TERM设置为xterm-256color
```
echo "export TERM=xterm-256color" >> ~/.profile
```