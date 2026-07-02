---
title: termux配置文件分享
date: 2022-12-04 14:28:47
tags:
  - Linux
  - Termux
cover: /img/termux-motd.jpg
top_img: /img/termux-motd.jpg
---
### termux版本：
这里咱比较喜欢termux-monet，带有monet取色支持和背景自定义。
链接：[HardcodedCat/termux-monet](https://github.com/HardcodedCat/termux-monet)
### 欢迎信息：
原版：[Generator/termux-motd](https://github.com/Generator/termux-motd)           
修改版：[Moe-hacker/termux-motd](https://github.com/Moe-hacker/termux-motd)
修改内容不介绍了，效果见仓库。
```
git clone https://github.com/Moe-hacker/termux-motd ~/.motd
echo ~/.motd/init.sh >> ~/.bashrc
echo ~/.motd/init.sh >> ~/.zshrc
```
如果手机“恰好”有docker支持：
```
mv ~/.motd/26-docker.disabled ~/.motd/26-docker
```
自启动docker并显示信息。
### 配色修改：
贴出咱的配色：
```
background:     #1E1E2E
foreground:     #CDD6F4
cursor:         #A6E3A1
color0:         #45475A
color8:         #585B70
color1:         #F38BA8
color9:         #F38BA8
color2:         #A6E3A1
color10:        #A6E3A1
color3:         #F9E2AF
color11:        #F9E2AF
color4:         #89B4FA
color12:        #89B4FA
color5:         #F5C2E7
color13:        #F5C2E7
color6:         #94E2D5
color14:        #94E2D5
color7:         #BAC2DE
color15:        #A6ADC8
```
来自catppuccin mocha，更改光标为淡绿色。
### 配置修改：
键盘扩展(来自tmoe)：
```
extra-keys = [ \
    ['ESC','<','>','BACKSLASH','=','^','$','()','{}','[]','ENTER'], \
    ['TAB','&',';','/','~','%','*','HOME','UP','END','PGUP'], \
    ['CTRL','FN','ALT','|','-','+','QUOTE','LEFT','DOWN','RIGHT','PGDN'] \
    ]
```
其他配置：
```
disable-terminal-session-change-toast = true
volume-keys = volume
use-fullscreen-workaround = true
terminal-cursor-style = underline
extra-keys-style = arrows-all
extra-keys-text-all-caps = true
use-black-ui = false
disable-hardware-keyboard-shortcuts = true
bell-character = vibrate
enforce-char-based-input = true
```
### signal 9修复：
咱有特权就直接动用就好啦！
```
su -c /system/bin/device_config set_sync_disabled_for_tests persistent
su -c /system/bin/device_config put activity_manager max_phantom_processes 2147483647
su -c setprop persist.sys.fflag.override.settings_enable_monitor_phantom_procs false
```
话说安卓系统连root进程都敢杀可真够狂的喵！