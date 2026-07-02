---
title: 手搓十六进制转二进制文件：“有手就行”背后的思考
date: 2024-11-09 23:42:56
tags:
    - C语言
cover: /img/hex-to-file.jpg
top_img: /img/hex-to-file.jpg
---
# 题目：
在某个不太正经的群里，咱看到群友发了一道非常简单的编程入门题目：
有如下hexdump输出，请转换回二进制文件
```
00000000: 1f8b 0808 dfcd eb66 0203 6461 7461 322e  .......f..data2.
00000010: 6269 6e00 013e 02c1 fd42 5a68 3931 4159  bin..>...BZh91AY
00000020: 2653 59ca 83b2 c100 0017 7fff dff3 f4a7  &SY.............
00000030: fc9f fefe f2f3 cffe f5ff ffdd bf7e 5bfe  .............~[.
00000040: faff dfbe 97aa 6fff f0de edf7 b001 3b56  ......o.......;V
00000050: 0400 0034 d000 0000 0069 a1a1 a000 0343  ...4.....i.....C
00000060: 4686 4341 a680 068d 1a69 a0d0 0068 d1a0  F.CA.....i...h..
00000070: 1906 1193 0433 5193 d4c6 5103 4646 9a34  .....3Q...Q.FF.4
00000080: 0000 d320 0680 0003 264d 0346 8683 d21a  ... ....&M.F....
00000090: 0686 8064 3400 0189 a683 4fd5 0190 001e  ...d4.....O.....
000000a0: 9034 d188 0343 0e9a 0c40 69a0 0626 4686  .4...C...@i..&F.
000000b0: 8340 0310 d340 3469 a680 6800 0006 8d0d  .@...@4i..h.....
000000c0: 0068 0608 0d1a 64d3 469a 1a68 c9a6 8030  .h....d.F..h...0
000000d0: 9a68 6801 8101 3204 012a ca60 51e8 1cac  .hh...2..*.`Q...
000000e0: 532f 0b84 d4d0 5db8 4e88 e127 2921 4c8e  S/....].N..')!L.
000000f0: b8e6 084c e5db 0835 ff85 4ffc 115a 0d0c  ...L...5..O..Z..
00000100: c33d 6714 0121 5762 5e0c dbf1 aef9 b6a7  .=g..!Wb^.......
00000110: 23a6 1d7b 0e06 4214 01dd d539 af76 f0b4  #..{..B....9.v..
00000120: a22f 744a b61f a393 3c06 4e98 376f dc23  ./tJ....<.N.7o.#
00000130: 45b1 5f23 0d8f 640b 3534 de29 4195 a7c6  E._#..d.54.)A...
00000140: de0c 744f d408 4a51 dad3 e208 189b 0823  ..tO..JQ.......#
00000150: 9fcc 9c81 e58c 9461 9dae ce4a 4284 1706  .......a...JB...
00000160: 61a3 7f7d 1336 8322 cd59 e2b5 9f51 8d99  a..}.6.".Y...Q..
00000170: c300 2a9d dd30 68f4 f9f6 7db6 93ea ed9a  ..*..0h...}.....
00000180: dd7c 891a 1221 0926 97ea 6e05 9522 91f1  .|...!.&..n.."..
00000190: 7bd3 0ba4 4719 6f37 0c36 0f61 02ae dea9  {...G.o7.6.a....
000001a0: b52f fc46 9792 3898 b953 36c4 c247 ceb1  ./.F..8..S6..G..
000001b0: 8a53 379f 4831 52a3 41e9 fa26 9d6c 28f4  .S7.H1R.A..&.l(.
000001c0: 24ea e394 651d cb5c a96c d505 d986 da22  $...e..\.l....."
000001d0: 47f4 d58b 589d 567a 920b 858e a95c 63c1  G...X.Vz.....\c.
000001e0: 2509 612c 5364 8e7d 2402 808e 9b60 02b4  %.a,Sd.}$....`..
000001f0: 13c7 be0a 1ae3 1400 4796 4370 efc0 9b43  ........G.Cp...C
00000200: a4cb 882a 4aae 4b81 abf7 1c14 67f7 8a34  ...*J.K.....g..4
00000210: 0867 e5b6 1df6 b0e8 8023 6d1c 416a 28d0  .g.......#m.Aj(.
00000220: c460 1604 bba3 2e52 297d 8788 4e30 e1f9  .`.....R)}..N0..
00000230: 2646 8f5d 3062 2628 c94e 904b 6754 3891  &F.]0b&(.N.KgT8.
00000240: 421f 4a9f 9feb 2ec9 83e2 c20f fc5d c914  B.J..........]..
00000250: e142 432a 0ecb 0459 1b15 923e 0200 00    .BC*...Y...>...
```
# 尝试解题：
很显然，是道有手就行的题目，思路就是利用一个char类型能存储两位十六进制，创造一块char *类型的内存当数组，将这一堆字符解析，每两位放入一个char类型数据。
由于C解析数据太麻烦，我们用shell解析下这段数据：
写入文件xxx后，
```sh
 cat xxx|cut -d ' ' -f2-9|sed -e "s/ //g"|while read i; do echo \"$i\"; done
```
于是我们获得了要解析的字符串。
```C
char *buf = "1f8b0808dfcdeb66020364617461322e"
            "62696e00013e02c1fd425a6839314159"
            "265359ca83b2c10000177fffdff3f4a7"
            "fc9ffefef2f3cffef5ffffddbf7e5bfe"
            "faffdfbe97aa6ffff0deedf7b0013b56"
            "04000034d00000000069a1a1a0000343"
            "46864341a680068d1a69a0d00068d1a0"
            "1906119304335193d4c6510346469a34"
            "0000d32006800003264d03468683d21a"
            "0686806434000189a6834fd50190001e"
            "9034d18803430e9a0c4069a006264686"
            "83400310d3403469a680680000068d0d"
            "006806080d1a64d3469a1a68c9a68030"
            "9a68680181013204012aca6051e81cac"
            "532f0b84d4d05db84e88e12729214c8e"
            "b8e6084ce5db0835ff854ffc115a0d0c"
            "c33d6714012157625e0cdbf1aef9b6a7"
            "23a61d7b0e06421401ddd539af76f0b4"
            "a22f744ab61fa3933c064e98376fdc23"
            "45b15f230d8f640b3534de294195a7c6"
            "de0c744fd4084a51dad3e208189b0823"
            "9fcc9c81e58c94619daece4a42841706"
            "61a37f7d13368322cd59e2b59f518d99"
            "c3002a9ddd3068f4f9f67db693eaed9a"
            "dd7c891a1221092697ea6e05952291f1"
            "7bd30ba447196f370c360f6102aedea9"
            "b52ffc4697923898b95336c4c247ceb1"
            "8a53379f483152a341e9fa269d6c28f4"
            "24eae394651dcb5ca96cd505d986da22"
            "47f4d58b589d567a920b858ea95c63c1"
            "2509612c53648e7d2402808e9b6002b4"
            "13c7be0a1ae3140047964370efc09b43"
            "a4cb882a4aae4b81abf71c1467f78a34"
            "0867e5b61df6b0e880236d1c416a28d0"
            "c4601604bba32e52297d87884e30e1f9"
            "26468f5d30622628c94e904b67543891"
            "421f4a9f9feb2ec983e2c20ffc5dc914"
            "e142432a0ecb04591b15923e020000";
```
由于这里只是解题，所以buf咱当全局变量用的，大家千万不要学。。。

然后咱就拿bash写了自动生成十六进制匹配：
```sh
j=0;for i in 0 1 2 3 4 5 6 7 8 9 a b c d e f; do echo "if(p[1]=='$i'){ret+=$j}"; j=$((j+1)); done
```
咱的最初版题解长这样：
```C
#include <fcntl.h>
#include <unistd.h>
#include <stdlib.h>
#include <fcntl.h>
static char get_first_tow_hex_val(char *p){
char ret=0;
if(p[0]=='0'){ret+=16*0;}
if(p[0]=='1'){ret+=16*1;}
if(p[0]=='2'){ret+=16*2;}
if(p[0]=='3'){ret+=16*3;}
if(p[0]=='4'){ret+=16*4;}
if(p[0]=='5'){ret+=16*5;}
if(p[0]=='6'){ret+=16*6;}
if(p[0]=='7'){ret+=16*7;}
if(p[0]=='8'){ret+=16*8;}
if(p[0]=='9'){ret+=16*9;}
if(p[0]=='a'){ret+=16*10;}
if(p[0]=='b'){ret+=16*11;}
if(p[0]=='c'){ret+=16*12;}
if(p[0]=='d'){ret+=16*13;}
if(p[0]=='e'){ret+=16*14;}
if(p[0]=='f'){ret+=16*15;}
if(p[1]=='0'){ret+=0;}
if(p[1]=='1'){ret+=1;}
if(p[1]=='2'){ret+=2;}
if(p[1]=='3'){ret+=3;}
if(p[1]=='4'){ret+=4;}
if(p[1]=='5'){ret+=5;}
if(p[1]=='6'){ret+=6;}
if(p[1]=='7'){ret+=7;}
if(p[1]=='8'){ret+=8;}
if(p[1]=='9'){ret+=9;}
if(p[1]=='a'){ret+=10;}
if(p[1]=='b'){ret+=11;}
if(p[1]=='c'){ret+=12;}
if(p[1]=='d'){ret+=13;}
if(p[1]=='e'){ret+=14;}
if(p[1]=='f'){ret+=15;}
return ret;
}
int main(){
char *data=malloc(114514);
int len=0;
for(int i=0;buf[i]!='\0';i+=2){
data[len]=get_first_tow_hex_val(&buf[i]);
len++;
}
int fd=open("output",O_CREAT|O_RDWR,0777);
write(fd,data,len);
close(fd);
return 0;
}
```
确实，解出来了，能拿分，运行也不慢，又不是算法考试也不可能TLE。。。。
显然，这段代码简直就是一坨，咱绝对不会承认这是咱写出来的。
所以正确的代码应该怎么写呢？
首先咱想到了GNU C的switch连续匹配扩展（其实用if也可以）：
```C
char hex_to_char(char i)
{
        switch (i) {
        case '0' ... '9':
                return i - '0';
        case 'a' ... 'f':
                return i - 'a' + 10;
        }
        return 0;
}
```
也可以写成if的形式：
```C
char hex_to_char(char i)
{
        if (i >= '0' && i <= '9') {
                return i - '0';
        } else if (i >= 'a' && i <= 'f') {
                return i - 'a' + 10;
        }
        return 0;
}
```
然后改一下get_first_tow_hex_val()，
```C
char get_first_tow_hex_val(char *p)
{
        return (hex_to_char(p[0]) << 4) + hex_to_char(p[1]);
}
```
看起来好一点了，但是，还不是最好的写法！
在群友的提示下，hex_to_char()可以直接用宏魔法实现，由于我们“手动保证”了输入值完全为合法十六进制表示，我们可以直接用三元表达式：
```C
#define hex_to_char(x) ((x) >= 'a' ? (x) - 'a' + 10 : (x) - '0')
```
为啥这里的数据是合法的呢，因为由瞪眼法可得buf中只有0-9和a-f，如果怕有非法数据也可以先加一个判断：
```C
#define is_hex(x) (((x) >= '0' && (x) <= '9') || ((x) >= 'a' && (x) <= 'f'))
```
Btw，这个宏也可以用GNU扩展（三元表达式缺省）来调用（exit()可以换成自定义的error处理函数）,当然，这并不规范：
```
is_hex(p[0]) && is_hex(p[1]) ?: exit(1);
```
剩下的代码为：
```C
static char get_first_tow_hex_val(char *p)
{
        return (hex_to_char(p[0]) << 4) + hex_to_char(p[1]);
}
int main(void)
{
        char *data = malloc(114514);
        int len = 0;
        for (int i = 0; buf[i] != '\0'; i += 2) {
                data[len] = get_first_tow_hex_val(&buf[i]);
                len++;
        }
        int fd = open("output", O_CREAT | O_RDWR, 0777);
        write(fd, data, len);
        close(fd);
        return 0;
}
```
看上去好多了（长舒一口气）。
# 最终正解：
一觉起来天塌了，有大佬说写入文件后`cat dump|xxd -r > output`可以直接还原hexdump，白写了半天代码，呜呜呜呜呜。。。。。。。。。。。。。。。。。
