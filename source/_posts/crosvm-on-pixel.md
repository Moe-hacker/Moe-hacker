---
layout: post
title: (需要ROOT)在Tensor设备上用crosvm跑Fedora，并结合magisk实现后台ssh服务
date: 2025-07-05 17:22:17
tags:
- crosvm
- Linux
- Android
- Termux
- AVF
top_img: /img/crosvm-on-pixel.jpg
cover: /img/crosvm-on-pixel.jpg
---
# 前言
整了台pixel7a，海蓝色，很软糯的颜色，很可爱。
升级了A16，有动画了，UI好看了不少。
然后就是，牢谷的terminal实在太难用了！谷歌！佩奇！
从Android beta升级到正式版需要清空数据，于是干脆root了。
然后研究下如何手动跑vm，
非常"幸运"的是，用牢谷的镜像牢谷的配置，运行terminal里的镜像，是没网的，寄。
用牢谷的kernel跑crosvm，磁盘驱动无法加载，寄。
牢谷貌似还给ttyd打了补丁，导致给terminal构建能跑的镜像还挺复杂的。有人做了个nix镜像（我勒个可复现性啊）
但是，很可惜，我看不懂（理直气壮）
~~那很坏了，我看未必，实则不然，恰恰相反......~~
总之天无绝人之路！
刷github看到这个项目
https://github.com/polygraphene/gunyah-on-sd-guide
于是问题就变得简单了。
# Fedora, 启动！
## 构建镜像
rootfs从lxc-images源下载一个就行，用我写的rurima也行。
大概就是：
```sh
dd if=/dev/zero of=fedora.img bs=1G count=8
mkfs.ext4 fedora.img
LOOPFILE=$(losetup -f)
losetup $LOOPFILE fedora.img
mkdir fedora
mount $LOOPFILE fedora
rurima pull fedora:42 ./fedora
rurima ruri -p --no-rurienv ./fedora
```
装上基本组件，重点是装systemd和dracut，linux-firmware和netplan。
然后是启用sshd，以及记得设置root密码。
Fedora默认提供的是签名的内核镜像，crosvm无法识别，所以我们手动构建一遍内核，这里我直接用defconfig构建的，就不细讲了, 大概是：
```sh
wget https://mirrors.tuna.tsinghua.edu.cn/kernel/v6.x/linux-6.15.tar.gz
tar -xvf linux-6.15.tar.gz
cd linux-6.15
make ARCH=arm64 defconfig
make ARCH=arm64 -j$(nproc)
make modules_install
cp arch/arm64/boot/Image /boot/vmlinux
make install
cd /boot
dracut -f initrd.img 6.15.0
```
(MacBook air：无所谓，我会发烫)
然后装上内核模块，复制出Image和dracut生成的initrd，fedora，启动！
这里我把需要的文件放在/data/vm目录下，其中fedora.img是rootfs的镜像，vmlinux是内核镜像，initrd.img是dracut生成的initrd。
## 启动并配置系统
将下一节的service.sh中的&符号去掉，直接运行脚本，即可启动crosvm
进入系统后,配置网络：
```sh
cat << EOF > /etc/netplan/90-default.yaml
network:
    version: 2
    ethernets:
        all-en:
            match:
                name: en*
            dhcp4: false

            addresses:
              - 192.168.8.2/24
            routes:
              - to: default
                via: 192.168.8.1
            nameservers:
                  addresses: [8.8.8.8]
            dhcp6: true
            dhcp6-overrides:
                use-domains: true
        all-eth:
            match:
                name: eth*
            dhcp4: true
            dhcp4-overrides:
                use-domains: true
            dhcp6: true
            dhcp6-overrides:
                use-domains: true
EOF
netplan apply
```
根据配置，您现在可用地址192.168.8.2进行ssh连接，可以在hosts模块中添加：
```
192.168.8.2 fedora
```
## 启动脚本
service.sh，用于启动。
```sh
#!/system/bin/sh
unset LD_PRELOAD
cd /data/vm
ifname=crosvm_tap
if [ ! -d /sys/class/net/$ifname ]; then
        ip tuntap add mode tap vnet_hdr $ifname
        ip addr add 192.168.8.1/24 dev $ifname
        ip link set $ifname up
        ip r a table wlan0 192.168.8.0/24 via 192.168.8.1 dev $ifname
        iptables -D INPUT -j ACCEPT -i $ifname
        iptables -D OUTPUT -j ACCEPT -o $ifname
        iptables -I INPUT -j ACCEPT -i $ifname
        iptables -I OUTPUT -j ACCEPT -o $ifname
        iptables -t nat -D POSTROUTING -j MASQUERADE -o wlan0 -s 192.168.8.0/24
        iptables -t nat -I POSTROUTING -j MASQUERADE -o wlan0 -s 192.168.8.0/24
        sysctl -w net.ipv4.ip_forward=1

        ip rule add from all fwmark 0/0x1ffff iif wlan0 lookup wlan0
        ip rule add iif $ifname lookup wlan0

        iptables -j ACCEPT -D FORWARD -i $ifname -o wlan0
        iptables -j ACCEPT -D FORWARD -m state --state ESTABLISHED,RELATED -i wlan0 -o $ifname
        iptables -j ACCEPT -D FORWARD -m state --state ESTABLISHED,RELATED -o wlan0 -i $ifname
        iptables -j ACCEPT -I FORWARD -i $ifname -o wlan0
        iptables -j ACCEPT -I FORWARD -m state --state ESTABLISHED,RELATED -i wlan0 -o $ifname
        iptables -j ACCEPT -I FORWARD -m state --state ESTABLISHED,RELATED -o wlan0 -i $ifname
fi

ulimit -l unlimited
/apex/com.android.virt/bin/crosvm run \
        --gpu-backend=virglrenderer \
        --disable-sandbox --swiotlb 64 \
        --params 'loglevel=0' --mem 4096 --cpus 8 \
        --net tap-name=$ifname \
        --initrd initrd.img --socket vm.sock \
        --block fedora.img,root vmlinux &
```
## 管理脚本：
action.sh，用于启动和停止vm。
注：主播比较懒，直接写入的magisk自带的systemless hosts模块。
```sh
if [ -e /data/vm/vm.sock ];then
    /apex/com.android.virt/bin/crosvm stop /data/vm/vm.sock
    rm /data/vm/vm.sock
    echo stop
    sleep 1
else
    /data/adb/modules/hosts/service.sh
    echo start
    sleep 1
fi
```
# 测试记录：
可能启动不太稳定，需要手动执行action.sh，
挂梯无法连接，您可能需要端口转发。
挂机23小时正常连接，但是中间我没用设备，一直待机。
总体来讲挺舒服的。
# 鸣谢
感谢 [polygraphene](https://github.com/polygraphene) 的项目提供了网络解决方案。
# 后记：
我们来细数Terminal的罪恶：
- 不支持自定义镜像
- 前端难用
- 挂载系统相关镜像和配置进虚拟机，我知道他的设计是想尽可能安全的，内核模块甚至在erofs分区，但显然安全性由此破缺
- 磁盘驱动有OOM问题，同时写入磁盘较大数据会直接卡死整个系统

显然，Terminal既没有做到安全与稳定，也没有做到易用和可定制。
`“在我看来，谷歌在这方面没有做到计算机学上的抽象，反而做到了网络用语上的抽象”`
EOF