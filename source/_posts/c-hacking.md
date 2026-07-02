---
title: C语言的一些`守序善良`的写法
date: 2024-03-26 00:00:23
tags:
  - Linux
  - C语言
top_img: /img/c-hacking.jpg
cover: /img/c-hacking.jpg
---
很显然，这些写法大多并不规范，也不被提倡。
很显然，咱并没有在windows下试过这些代码，而且实测大部分在线编程网站用的是Linux，可以接受GNU C扩展支持。
~~如果有人问我为什么折腾，为什么以折腾这些无聊的东西作为目标，那他们完全可以问，为什么要登上最高峰？为什么人类要登月？………我选择去折腾，我，选择去折腾！~~（逃）
# 趋近符号：
很出名的写法，比如：
```C
char *str=strdup("hello;");
for(int i=strlen(str);i-->0;i=i){
	if(str[i]==';'){
		str[i]='\0';
		break;
	}
}
```
这将str中最后一个`;`的位置设置为结尾。
# 对指针解引用：
C语言提供方便的指针解引用，于是：
```C
if (strchr(p, ';') == NULL) {
		if(strchr(p, '\n')!=NULL) {
			*strchr(p, '\n') = '\0';
		}
	} else {
		*strchr(p, ';') = '\0';
}
```
由于strchr()返回的是在原字符串指针中的地址，因此只要p是可写的，就可以直接赋值strchr指向的位置。
编译器同意，编写者同意，而且能跑，皆大欢喜。
# 位运算达到同时成立/不成立：
使用p^q可以判断p和q是否同时成立或不成立，ruri中`-a`和`-q`需要同时设置，曾经判断是这么写的：
```C
if ((container->cross_arch == NULL) ^ (container->qemu_path == NULL)) {
		error("{red}Error: --arch and --qemu-path should be set at the same time QwQ\n");
}
```
事实上，更规范的写法是直接用!=判断两个表达式是否相等：
```C
if ((container->cross_arch == NULL) != (container->qemu_path == NULL)) {
		error("{red}Error: --arch and --qemu-path should be set at the same time QwQ\n");
}
```
不过如果你想要最简写法：
```C
if ((!!(container->cross_arch)) ^ (!!(container->qemu_path))) {
		error("{red}Error: --arch and --qemu-path should be set at the same time QwQ\n");
}
```
# `截断`的指针:
在rurima中，subcommand函数被设计为和main一样拥有两个参数`int argc, char **argv`,
在main函数中解析command，解析到subcommand后把subcommand参数直接传给相关函数，是这么写的：
```C
void lxc(int argc, char **_Nonnull argv);
else if (strcmp(argv[i], "lxc") == 0) {
			if (i + 1 >= argc) {
				error("{red}No subcommand specified!\n");
			}
			lxc(argc - i - 1, &argv[i + 1]);
			return 0;
}
```
# 三级指针控制数组：
在函数外面：
```C
char **array=NULL;
```
函数参数`char ***_Nullable array`
函数内部：
```C
(*array) = malloc(sizeof(char *));
(*array)[0] = NULL;
```
这样就可以改变数组指向的内存，于是可以用realloc新增数组成员，达到内存动态分配
# 对int值进行位运算：
我们知道，int值以二进制存储。
于是可以用位运算来乘除2的次方数。
```C
int i = 114514;
i <<= 1; //乘以2
printf("%d\n", i);
i >>= 1; //除以2
printf("%d\n", i);
```
\>\>=2会除以4，以此类推。
# Bang-Bang折叠布尔值：
我们知道，判断条件只有两个值，`0`(false)和`1`(true)。
我们知道，当一个数字前面被加上`!`，它将变成一个判断条件。
我们还知道，双重否定表示极度的肯定（~~sodayo～~~）。
于是，在C语言中：
```
!!0 == 0
!!114514 == 1
```
没错，双叹号下只有两个值，0和1。
ACM的文章Catch-23: The New C Standard Sets the World on Fire中，通过这个可能并非规范中有定义但确实有效的例子严厉批判了C2x规范`将realloc()内存为0设置为未定义行为而非free()内存的行为`这一条，原因是realloc()做free()的写法和bang-bang一样在民间已经广为流传。
# printf()中的三元表达式：
没错，至少GNU环境下，printf()中是可以套三元表达式的。
可以试下：
```C
printf(1 ? "true\n" : "false\n");
printf(0 ? "true\n" : "false\n");
```
十分，甚至九分的优雅。
# for()循环括号内执行代码：
C99放宽了对for()循环的限制。
于是就可以优（编）化（写）代（屎）码（山）。
init是一个表达式，但显然不做判断。
于是：
同时定义两个变量：
```C
for(int i = 0 | int j = 255; ; )
```
condition就是一个判断，不过对于i--的情况，可以用上面的!!判断是否为0。
```C
for (int i = 255; !!i; i--)
```
increment是一段代码，显然不会做任何限制
```C
for (int i = 255; !!i; printf("%d\n", i--))
                ;
```
所以再稍微发挥一下：
```C
for (int i = 255; !!i; printf((i % 2 == 0 ? "%d\n" : ""), i--))
                ;
```
非常离谱。
# GNU C扩展：
## 表达式中的语句和声明：
在GNU C中，你可以用`({})`来获得一段表达式的值，这种表达式的值等于最后面的那个以;结尾的变量。
比如：
```C
int foo = ({
        int bar = 1;
        bar++;
        bar;
});
```
于是foo变量的值等于最后那行语句bar;中bar的值。
## __VA_ARGS__宏：
这应该是GNU C最出名的特性了吧。
可以让宏定义接受不定参数。
比如：
```C
#define warning(...) fprintf(stderr, ##__VA_ARGS__)
```
## 三元表达式的缺省：
`bar ?: foo`等同于`bar ? bar : foo`
有什么用？
比如说：
```C
fd >= 0 ?: exit(1);
```
可以让fd在open()失败时退出程序。
还是有点用的。
## 零长度数组：
`char u[0];`这样的写法是可以被接受的，一般放在结构体最后面，使结构体可以变长，普通程序基本用不到。不过在内存寸土寸金的年代，这样显然可以节省不少内存。
## 属性（__attribute__()宏)：
__attribute__((constructor))：使函数在main()之前被执行。
__attribute__((destructor))：使函数在调用之后被执行。
__attribute__((unused))：函数或变量可能不会使用，可以作为注释用途或抑制编译警告。
# 后记：
(null)