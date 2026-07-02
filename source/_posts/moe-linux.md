---
title: 安装Linux后必须要做的n件事
date: 2024-01-17 21:50:44
tags:
  - Linux
cover: /img/moe-linux.png
top_img: /img/moe-linux.png
---
这次不是好久不见了喵～
最近一次配置GNU/Linux系统是今天，昨天给我的小米平板5刷了map220v大佬的ubuntu包，终于不用吃灰了喵～感谢map220v大佬喵～
话不多说开始正文：
# 软件安装：
桌面我一般用kde-plasma，这个不必多说了，推荐几款常用软件：
- htop：显示进程信息(TUI)
- conky-all:显示系统占用(GUI)
- batcat:高亮版cat
- latte-dock：比kde自带的更好看的dock和panel
- onboard：屏幕键盘
- spectacle：截图工具
- cloc：代码行数统计
- VSCode：不用多说

### 常用开发/日常软件：
git wget aria2 curl make cmake clang clang-tidy clang-format gdb binutils openssh zip unzip gzip p7zip nano manpages zsh android-tools zsh
### VSCode插件：
- C/C++：C语言官方扩展
- Codesnap：创建代码截图
- filesize：显示文件大小
- Jetbrains Mono：Jetbrains的字体，挺适合开发，装完要在弹出的文件夹里选中所有字体手动安装
- Path Intellisense：补全路径
- Todo Tree：代办事项
- Markdown All in One：Markdown支持
- Tokyo Night：目前在用的主题
- Trailing Space：高亮空格，强迫症患者狂喜
- GlassIt-VSC：透明VSCode窗口，不过影响观感所以我停用了
### VSCode设置：
window.dialogStyle和window.titleBarStyle要设置为custom，不然的话UI会巨丑。
### 桌面设置：
kde默认的panel太难看，删了（据说新版本kde6有屎诗级更新，但是Bugs will happen,if they don't happen in hardware,they will happen in software and if they don't happen in your software and they will happen in somebody else's software --Linus）
latte-dock开透明，关背景阴影，开圆角，背景大小开到100,恭喜你获得了一个和MIUI12.5难分秋毫的特效dock。
kde系统设置的桌面特效直接全开《难分秋毫.jpg》
然后我们需要一张壁纸，一个conky配置：
https://github.com/moe-sushi/conkyrc
我们可能还需要一张头像23333
### 输入法：
这里我用的fcitx-googlepinyin，需要fcitx-configtool和fcitx-config-gtk3这两个包。
### zsh设置：
https://gitee.com/mo2/linux
用惯了这个，挺好用的。。。
### 魔法工具：
(这是可以说的吗/w\\)
这里咱用Geph：
```sh
cargo install geph4-client
```