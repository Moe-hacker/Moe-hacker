---
title: hexo博客配置教程
date: 2022-12-04 11:27:43
tags:
  - Linux
  - hexo
  - blog
top_img: /img/gura.jpg
cover: /img/gura.jpg
---
咱自己的博客配置教程喵～
### 前期准备：
本博客在linux环境下搭建，部分内容于windows下稍有不同。
你需要：git，ssh，nodejs，npm，github-cli。
你可能还需要：一个脑子。
可惜咱是没有脑子的喵呜………
去github账号设置=>开发者设置=>令牌中获取一个token。
在你的github账户下创建 用户名.github.io这个仓库。
注：github-cli的token为明文存储，请勿在不受您本人信任的设备上用这种方式登录。
然后：
```sh
gh auth login
git config --global user.email "你的电子邮件地址"
git config --global user.name "你的github用户名"
```
### hexo部署：
```sh
cd ~
mkdir hexo
npm config set registry https://registry.npmmirror.com/
npm install -g hexo
cd hexo
hexo init
npm install hexo-theme-butterfly hexo-renderer-pug hexo-renderer-stylus cheerio hexo-deployer-git hexo-wordcount --save
```
### hexo配置：
```sh
nano ~/hexo/_config.yml
```
更改以下配置：
```yaml
title: 网站标题
subtitle: 网站子标题
description: 网站描述
keywords:
    - 关键字
    - 关键字
author: 你的名字
language: zh-CN
timezone: Asia/Shanghai
url: 你的网站地址
theme: butterfly
deploy:
  type: git
  repository: 'git@github.com:你的用户名/你的博客仓库'
```
### 主题配置：
请熟练使用nano等编辑器的跳转功能！！！
```sh
nano ~/hexo/node_modules/hexo-theme-butterfly/_config.yml
```
目前个人修改：
```yaml
menu:
   主页: / || fas fa-home
   归档: /archives/ || fas fa-archive
   标签: /tags/ || fas fa-tags
   分类: /categories/ || fas fa-folder-open
  # List||fas fa-list:
  #   Music: /music/ || fas fa-music
  #   Movie: /movies/ || fas fa-video
   友情链接: /link/ || fas fa-link
   关于: /about/ || fas fa-heart
social:
   fab fa-github: https://github.com/Moe-hacker || Github
   fas fa-envelope: mailto:moe-hacker@outlook.com || Email
favicon: /img/face.png
avatar:
  img: /img/face.jpg
  effect: false
index_img: /img/fufu.jpg
default_top_img: /img/cover.jpg
archive_img: /img/cover.jpg
tag_img: /img/cover.jpg
tag_per_img: /img/cover.jpg
category_img: /img/cover.jpg
category_per_img: /img/cover.jpg

default_cover: /img/cover.jpg
wordcount:
  enable: true
  post_wordcount: true
  min2read: true
  total_wordcount: true
theme_color:
   enable: true
   main: "#FEE4D0"
   paginator: "#fee4d0"
   button_hover: "#fee4d0"
   text_selection: "#000000"
   link_color: "#99a9bf"
   meta_color: "#fee4d0"
   hr_color: "#A6E3A1"
#   code_foreground: "#F47466"
#   code_background: "rgba(27, 31, 35, .05)"
   toc_color: "#fee4d0"
#   blockquote_padding_color: "#49b1f5"
#   blockquote_background_color: "#49b1f5"
#   scrollbar_color: "#49b1f5"
   meta_theme_color_light: "ffffff"
   meta_theme_color_dark: "#0d0d0d"
background: url(https://moe-hacker.github.io/img/fufu_background.jpg)
# Footer Background
footer_bg: true
enter_transitions: true
activate_power_mode:
  enable: true
  colorful: true # open particle animation (冒光特效)
  shake: true #  open shake (抖動特效)
  mobile: false
fireworks:
  enable: true
  zIndex: 9999 # -1 or 9999
  mobile: true
beautify:
  enable: true
  field: post # site/post
  title-prefix-icon: # '\f0c1'
  title-prefix-icon-color: # '#F47466'
hr_icon:
  enable: true
  icon: # the unicode value of Font Awesome icon, such as '\3423'
  icon-top:
subtitle:
  enable: true
  # Typewriter Effect (打字效果)
  effect: true
  # Effect Speed Options (打字效果速度參數)
  startDelay: 300 # time before typing starts in milliseconds
  typeSpeed: 150 # type speed in milliseconds
  backSpeed: 50 # backspacing speed in milliseconds
  # loop (循環打字)
  loop: true
  # source 調用第三方服務
  # source: false 關閉調用
  # source: 1  調用一言網的一句話（簡體） https://hitokoto.cn/
  # source: 2  調用一句網（簡體） http://yijuzhan.com/
  # source: 3  調用今日詩詞（簡體） https://www.jinrishici.com/
  # subtitle 會先顯示 source , 再顯示 sub 的內容
  source: false
  # 如果關閉打字效果，subtitle 只會顯示 sub 的第一行文字
  sub:
    - Keep moe.
    - Keep cool.
    - keep hacking.
    - ——Talk is cheap，
    - show me the code.
aside:
  enable: true
  hide: false
  button: true
  mobile: true # display on mobile
  position: right # left or right
  display:
    archive: true
    tag: true
    category: true
  card_author:
    enable: true
    description:
    button:
      enable: true
      icon: fab fa-github
      text: Follow Me
      link: https://github.com/Moe-hacker
  card_announcement:
    enable: true
    content: 沨鸾的小窝
  card_recent_post:
    enable: true
    limit: 5 # if set 0 will show all
    sort: date # date or updated
    sort_order: # Don't modify the setting unless you know how it works
```
几张图片具体内容在咱博客仓库里有，请在github打开本项目仓库。
### 链接配置：
```
hexo new page link
hexo new page tags
hexo new page categories
hexo new page about
```
友链页面：
nano ~/hexo/source/link/index.md
两行横线中间添加：
type: "link"
另外三个页面同理。
about页面记得写点你的自我介绍啥的，要不然关于页面会是空的。
### 友链配置：
```sh
mkdir -p ~/hexo/source/_data
nano ~/hexo/source/_data/link.yml
```
示例：
```yaml
- class_name: 友情链接
  class_desc: 欢迎互关喵～
  link_list:
    - name: Moe-hacker
      link: https://moe-hacker.github.io
      avatar: /img/face.jpg
      descr: 咱自己喵～
```
### 开始写博客：
```sh
rm /hexo/source/_posts/hello-world.md
hexo new test
nano /hexo/source/_posts/test.md
```
首先更改以下内容，加入两行横线中间：
```md
title: 标题
tags:
  - tag1
  - tag2
  - tag3
top_img: /img/cover.jpg
cover: /img/cover.jpg
```
然后便是markdown书写了，具体语法自行百度，仅展示两个技巧。
#### 居中文字：
```html
<p align="center">文字</p>
```
#### 行首缩进：
本人的博客遇到了&emsp;等字符不生效的问题，事实上这个符号就是这么直接打出来的。
不过咱有办法：
```html
<style>
.sj{ text-indent:2em}
</style>
<div class="sj">文字</div>
```
可以看到文字缩进了两个空格。
### 博客部署：
```sh
hexo g
hexo s
```
去访问localhost:4000，内容满意就可以部署到仓库了：
```
hexo d
```
### 注释：
图片放在 ~/hexo/public/img中，命名别搞混就行。
配置开启了一些动效，可以关闭。
部分配置自己试一下就知道是什么内容了。
配置里面图片混用一张是因为这只咱太懒，千万别学习。
最后就是，学习下md语法，写博客别跟咱一样水就好了。