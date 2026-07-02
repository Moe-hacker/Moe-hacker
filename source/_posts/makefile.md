---
title: 小型C语言项目：从手写configure脚本开始的构建系统编写
date: 2024-10-03 14:59:11
tags:
  - Linux
  - C语言
top_img: /img/configure.jpg
cover: /img/configure.jpg
---
我们在开发C语言项目的构建系统部分时，单用Makefile可能会出现很多难题：
我想使用一个CFLAG来提高安全性，可有些编译器不支持怎么办？
我想用的头文件不同平台有不同版本怎么办？
我想在编译前检查依赖库怎么办？
老实说，一个build.sh或许是个很好的选择。
或者你会说，为啥不用meson/CMake？因为咱是传统派23333
GNU项目一般都用configure来生成项目配置，但咱懒得学autotools的用法，于是我们这篇文章的主题是：
使用最原始的方法手写一个configure脚本！！！！
# 成品展示：
## 输出：
```
checking for make... /usr/bin/make
checking for strip... /usr/bin/strip
checking for compiler... aarch64-linux-gnu-gcc-11
checking whether the compiler supports GNU C11... ok
checking for header fcntl.h... ok
checking for header sys/ioctl.h... ok
checking for header sys/mount.h... ok
checking for header sys/socket.h... ok
checking for header unistd.h... ok
checking for header sys/capability.h... ok
checking for header seccomp.h... ok
checking for header pthread.h... ok
checking whether the compiler supports -ftrivial-auto-var-init=pattern... no
checking whether the compiler supports -fcf-protection=full... no
checking whether the compiler supports -flto=auto... ok
checking whether the compiler supports -fPIE... ok
checking whether the compiler supports -pie... ok
checking whether the compiler supports -Wl,-z,relro... ok
checking whether the compiler supports -Wl,-z,noexecstack... ok
checking whether the compiler supports -Wl,-z,now... ok
checking whether the compiler supports -fstack-protector-all... ok
checking whether the compiler supports -fstack-clash-protection... ok
checking whether the compiler supports -mshstk... no
checking whether the compiler supports -Wno-unused-result... ok
checking whether the compiler supports -O2... ok
checking whether the compiler supports -Wl,--build-id=sha1... ok
checking whether the compiler supports -ffunction-sections... ok
checking whether the compiler supports -fdata-sections... ok
checking whether the compiler supports -Wl,--gc-sections... ok
checking whether the compiler supports -U_FORTIFY_SOURCE -D_FORTIFY_SOURCE=3... ok
checking for -lcap... ok
checking for -lseccomp... ok
checking for -lpthread... ok
create config.mk... ok
```
可以看到，和大佬们的项目比起来，就像哈士奇与狼一般相似！
## 预期产物：
```config.mk
config.mk 
CFLAGS =  -flto=auto -fPIE -pie -Wl,-z,relro -Wl,-z,noexecstack -Wl,-z,now -fstack-protector-all -fstack-clash-protection -Wno-unused-result -O2 -Wl,--build-id=sha1 -ffunction-sections -fdata-sections -Wl,--gc-sections -U_FORTIFY_SOURCE -D_FORTIFY_SOURCE=3 -DRURI_COMMIT_ID=\"1c3a064\"
LD_FLAGS =  -lcap -lseccomp -lpthread
CC = aarch64-linux-gnu-gcc-11
```
# 开搓：
## 检查依赖命令：
```sh
  printf "checking for make... "
  if ! command -v make; then
    error "not found"
  fi
```
输出：
```
checking for make... /usr/bin/make
```
## 检查编译器：
```sh
  printf "checking for compiler... "
  if [ ! $CC ]; then
    if [ ! $(command -v cc) ]; then
      error "not found"
    fi
    CC=$(realpath $(command -v cc))
    export CC=${CC##*/}
  fi
  printf "$CC\n"
```
输出：
```
checking for compiler... aarch64-linux-gnu-gcc-11
```
## 检查C标准与静态链接支持：
```sh
  printf "checking whether the compiler supports GNU C11... "
  (echo "int main(){}" | $CC -x c -o /dev/null -std=gnu11 -) >/dev/null 2>&1 && printf "ok\n" || error "no"
  if [ $STATIC_COMPILE ]; then
    printf "checking whether the compiler supports -static compile... "
    echo "int main(){}" | $CC -static -x c -o /dev/null -std=gnu11 - >/dev/null 2>&1 && printf "ok\n" || error "no"
  fi
```
输出：
```
checking whether the compiler supports GNU C11... ok
checking whether the compiler supports -static compile... ok
```
## 检查头文件存在，不存在会报错：
```sh
check_header() {
  printf "checking for header $i... "
  printf "#include <$1>\nint main(){}" | $CC -x c -o /dev/null - >/dev/null 2>&1 && printf "ok\n" || error "not found"
}

check_header "sys/ioctl.h"
```
输出：
```
checking for header sys/ioctl.h... ok
```
## 检查CFLAG，不支持会自动跳过：
```sh
test_and_add_cflag() {
  printf "checking whether the compiler supports $1... "
  if echo "int main(void){}" | $CC $1 -Werror -x c -o /dev/null - >/dev/null 2>&1; then
    printf "ok\n" && export CFLAGS="$CFLAGS $1"
  else
    printf "no\n"
  fi
}

test_and_add_cflag "-ftrivial-auto-var-init=pattern"
```
输出：
```
checking whether the compiler supports -ftrivial-auto-var-init=pattern... no
```
## 检查链接库，这里不会报error，因为安卓的pthread库内置了
```sh
test_and_add_ldflag() {
  printf "checking for $1... "
  if [ $STATIC_COMPILE ]; then
    if echo "int main(){}" | $CC -static -x c -o /dev/null - $1 >/dev/null 2>&1; then
      printf "ok\n" && export LD_FLAGS="$LD_FLAGS $1"
    else
      printf "no\n"
    fi
  else
    if echo "int main(){}" | $CC -x c -o /dev/null - $1 >/dev/null 2>&1; then
      printf "ok\n" && export LD_FLAGS="$LD_FLAGS $1"
    else
      printf "no\n"
    fi
  fi
}

test_and_add_ldflag "-lpthread"
```
输出：
```
checking for -lpthread... ok
```
## 为C语言代码提供commit信息：
```sh
if command -v git >/dev/null; then
  CFLAGS="$CFLAGS -DRURI_COMMIT_ID=\\\"`git rev-parse --short HEAD`\\\""
fi
```
这会让编译器定义一个RURI_COMMIT_ID宏，值是最后的git commit id。

# 后记：
懒得写了，散会！