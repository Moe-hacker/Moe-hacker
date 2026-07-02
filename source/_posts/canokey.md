---
title: 国产物理密钥Canokey踩坑记录
date: 2024-02-22 19:42:04
tags:
  - Linux
  - Termux
top_img: /img/canokey.jpg
cover: /img/canokey.jpg
---
前段时间咱本着`再不买以后就买不到了`的心态购入了国产物理密钥Canokey，不得不说这价格是真的坚挺，至死不降那种。。。
闪烁的蓝灯，优雅的签名，逼格算是拉满了，不过使用过程是真的曲折坎坷。
咱主要是买来用于git签名与ssh认证的，配置过程前辈们已经写的很清楚了，写好了有奖励，写不好有惩罚（悲）
所以咱就不怎么写了，主要写使用OpenPGP Card的过程中遇到的坑。
# 基本配置：
## 配置SSH验证：
编辑 ~/.gnupg/gpg-agent.conf，加入：
```
enable-ssh-support
```
然后：
```
gpg --list-keys --with-keygrip
```
将keygrip写入~/.gnupg/sshcontrol
然后：
```
export SSH_AUTH_SOCK=$(gpgconf --list-dirs agent-ssh-socket)
gpgconf --launch gpg-agent
```
## git签名：
```
git config --global user.signingkey [key]
git config --global commit.gpgsign true
```
# Github提示密钥已存在：
生成子密钥前咱git提交一直是主密钥签的。
子密钥生成完后在github添加了好几次都提示密钥已存在，但又不识别我子密钥。
这时候先把原来的删了再添加就好了。真是离谱的bug。
# 卡片url设置：
找了半天网上都没写怎么设置这个地址。
在上传完你的key到keyserver后，浏览器访问keyserver，一般有个搜索框，搜索你的密钥，会显示一串可以复制的链接，链接大概像这样：
https://keyserver.ubuntu.com/pks/lookup?op=get&search=0x5f3d5c278995c790
把这个填入card的url里面，在其他设备上就可以直接fetch了。
# 修复电脑usb权限：
创建/etc/udev/rules.d/10-canokey.rules
写入：
SUBSYSTEM=="usb", ATTR{manufacturer}=="canokeys.org", GROUP="moe-hacker"
由于咱电脑只有咱一个用户，所以干脆不新建group了，小朋友千万不要学，不严谨，会被带坏的。
# wsl1 使用：
你需要一个win-gpg-agent。
然后把windows的gpg-agent连接到wsl
```sh
rm /root/.gnupg/S.gpg-agent
socat UNIX-LISTEN:/root/.gnupg/S.gpg-agent,fork UNIX-CONNECT:/mnt/c/Users/moe-h/AppData/Local/gnupg/agent-gui/S.gpg-agent &
rm /root/.gnupg/S.gpg-agent.ssh
socat UNIX-LISTEN:/root/.gnupg/S.gpg-agent.ssh,fork UNIX-CONNECT:/mnt/c/Users/moe-h/AppData/Local/gnupg/agent-gui/S.gpg-agent.ssh &
```
moe-h是咱的用户名，辣鸡M$用户名是错的。
# Termux中使用：
需要安装scdaemon这个包。termux下的读取就算在root用户下也不是多很稳定，so f**k u android !
如果你的安卓版本够低，可以试试termux-usb。
咱实测能弹出授权窗口，但会卡死。
因此直接root使用了。
tsu下编辑~/.bashrc，这是目前我用的一个比较稳定的配置：
```
cd ~
killall -9 gpg-agent 2>&1 > /dev/null
killall -9 scdaemon 2>&1 > /dev/null
export GPG_TTY=$(tty)
scd=`/data/data/com.termux/files/usr/libexec/scdaemon --daemon|cut -d ";" -f 1|cut -d "=" -f 2`
export SCDAEMON_INFO=$scd
export SSH_AUTH_SOCK=$(gpgconf --list-dirs agent-ssh-socket)
gpgconf --launch gpg-agent
export GPG_TTY=$(tty)
gpg-connect-agent updatestartuptty /bye > /dev/null
```
如果你用的容器，可能需要在里面安装usb-utils(usbutils)这个包。
# 修复终端中无法输入密码：
在有些地方（比如chroot容器里），tty会提示当前未在tty中
此时只需要：
```
script -q -O /dev/null
```
script命令会自动创建一个pty。
# 最后的一些碎碎念：
不得不说物理密钥小众确实是有原因的，就连我这个只是没接触过这类产品的`半萌新`上手都得半天。不过拿这东西签git是真的帅23333
