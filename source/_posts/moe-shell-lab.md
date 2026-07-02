---
title: 沨鸾的Shell小技巧
date: 2022-12-04 02:56:57
tags:
  - Linux
  - Shell
  - Termux
top_img: /img/cover.jpg
cover: /img/cover.jpg
---
跟着沨鸾学shell，学到最后只会喵喵喵。 
## 正经部分：
### 语法规范：
变量要加{}括起来。
函数最好加个function关键字。
头部一定要有释伴(shebang)。
记得写注释，要不然也就上帝能看懂你写的什么了。
退出时要有返回状态。
能用[[]]就别用[]。
尽量用printf代替echo使用以提供更好的兼容性。
没用的输出记得丢弃。
\> /dev/null丢不掉就2>&1 > /dev/null。
不要定义太复杂的架构，比如函数互相调用。
当然咱基本没怎么遵守过。
### 三元表达式：
比如你想要这样一段的功能：
```sh
if [[ $x == 1 ]];then
  echo test
else
  echo fail
fi
```
你可以这么写：
```sh
([[ $x == 1 ]]&&echo test)||echo fail
```
测试一下：
```sh
x=0
([[ $x == 1 ]]&&echo test)||echo fail
x=1
([[ $x == 1 ]]&&echo test)||echo fail
```
等下语法规范呢？
遵守是不可能遵守的了。。。
### 三元表达式plus：
比如你想要这一段的功能：
```sh
if [[ $x == 1 ]];then
  echo 1
elif [[ $x == 2 ]];then
  echo 2
else
  echo fail
fi
```
你可以写成这样：
```sh
([[ $x == 1 ]]&&echo 1)||([[ $x == 2 ]]&&echo 2)||echo fail
```
看你写的代码的人会感谢你的。
### 批量操作：
比如我们要下载一堆链接，却不想每次都敲命令剩下的部分，可以：
```sh
while read i
do
aria2c -x16 -c $i
done
```
所以作者到底在下载什么。。。
### 获取随机值：
我们在使用`$RANDOM`做文件名时会遇到一个问题：值太短，不够幸运的话可能还是有冲突。
这里我们可以加一个`$(date +%s)`来解决。
### 最高端的输出颜色自定义：
使用rgb代码定义输出颜色。
```
\033[1;38;2;R;G;Bm
```
比如moe-container里的这行：
```C
printf("\033[1;38;2;254;228;208mUsage:\n");
```
当然这是C语言。
等下这是shell技巧……对吧。 
### 输出居中：
首先你得知道要居中的输出有多长。
然后：
```sh
WIDTH=$(($(($(stty size|awk '{print $2}')))/2-居中字符长度的一半))
echo -e "\033[${WIDTH}C内容"
```
除了花哨点也没啥大用。
### 输出一行分割线：
```sh
WIDTH=$(stty size|awk '{print $2}')
echo $(yes "="|sed $WIDTH'q'|tr -d '\n')
```
当然可以玩的更花哨一点：
```sh
WIDTH=$(stty size|awk '{print $2}')
WIDTH=$((WIDTH/2-2))
echo "$(yes "="|sed $WIDTH'q'|tr -d '\n')xxxx$(yes "="|sed $WIDTH'q'|tr -d '\n')"
```
$WIDTH定义参照上一条。
或者像termux-container里这样：
```sh
WIDTH=$(stty size|awk '{print $2}')
WIDTH=$((WIDTH-13))
echo -e "\e[30;48;5;159mCONTAINER_RUN$(yes " "|sed $WIDTH'q'|tr -d '\n')\033[0m"
```
莫名科技感。
### sed正则匹配：
```
echo 123abc > test
sed -i "s/[0-9]*/数字替换/" test
cat test
```
正则表达式具体内容请自行利用搜索引擎。
想当年咱要是会用，termux-container里的屎山也能少点。
### 更改光标样式：
```sh
printf '\e[2 q'
printf '\e[6 q'
printf '\e[4 q'
```
仅在termux验证成功过。
### Ctrl+D信号捕获：
不是说好EOF不是信号的吗？
事实上read可以捕获。
read无论读到什么东西加回车都会将结果记录并正常退出。
但是，读到EOF却未换行会返回1。
可以read后用$?的值是否为0来作为条件进行捕获。
当然read逐字读取时不适用，但是我们还有方法专门针对逐字读取：
```sh
while :; do read -N 1 key&&if [[ ${key} == $(printf "\004") ]];then echo CTRL-D;fi; done
```
似乎挺没用的。
### 网易云歌曲名称格式化：
网易云默认下载的音乐命名格式是这样的：
```
Akie秋绘 - なんでもないや 没什么大不了的（翻自 Radwimps）.mp3
ENE - パズル.mp3
Hanser - 勾指起誓.mp3
のぶなが - 深海少女.mp3
南杉 - 樱花樱花想见你.mp3
鹿乃 - 小夜子.mp3
鹿乃 - 心拍数#0822.mp3
鹿乃 - 桜のような恋でした.mp3
```
(浓度过纯)
咱们可以这样：
```sh
ls *.mp3|while read music
do
artist=${music%% -*}
name=${music##*-\ }
name=${name%%.mp3}
name=${name%%"（"*}
name=${name%%"("*}
mv "$music" "$name-[$artist].mp3"
done
```
于是文件名就成了这样：
```
なんでもないや 没什么大不了的-[Akie秋绘].mp3
パズル-[ENE].mp3
勾指起誓-[Hanser].mp3
深海少女-[のぶなが].mp3
樱花樱花想见你-[南杉].mp3
小夜子-[鹿乃].mp3
心拍数#0822-[鹿乃].mp3
桜のような恋でした-[鹿乃].mp3
```
个人感觉好看多了。
### 萌新代码生成：
```sh
x(){
echo -e "number=input(\"请输入一个数字:\")"
echo -e "if number == 0:\n    print(\"0是一个偶数\")"
for i in {1..114514}
do
[[ $(($i%2)) == 0 ]]&&echo -e "elif number == $i:\n    print(\"$i是一个偶数\")"||echo -e "elif number == $i:\n    print(\"$i是一个奇数\")"
done
echo -e "else:\n    print(\"数太大了我还不会\")"
}
x > 1.py
```
逝python，但是运行会直接内存错误。
## 生草部分：
### 变量当函数/命令名执行：
```sh
test(){
  $@
}
test ls
```
不做类型检查你就可以为所欲为了是吧。
### 忽略Ctrl+C：
用户别想用Ctrl+C杀死你的进程(大草)。
```sh
trap "" SIGINT
```
