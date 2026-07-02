---
title: 如何优雅地从Lxc镜像偷rootfs
date: 2024-10-31 20:10:15
tags:
  - Linux
  - Docker
  - container
cover: /img/lxc.jpg
top_img: /img/lxc.jpg
---
首先我们拿到一个lxc镜像的链接，咱觉得bfsu的速度就很不错。
https://mirrors.bfsu.edu.cn/lxc-images/images
点进去，顺着目录就能找到rootfs.tar.xz,下载就完了。
本文完，就这么简单。
当然不是啊喂！我们要做到自动获取rootfs链接。
# 镜像列表：
```
LXC_MIRROR_MAIN="http://images.linuxcontainers.org"
LXC_MIRROR_BFSU="https://mirrors.bfsu.edu.cn/lxc-images"
LXC_MIRROR_TUNA="https://mirrors.tuna.tsinghua.edu.cn/lxc-images"
LXC_MIRROR_NJU="https://mirror.nju.edu.cn/lxc-images"
LXC_MIRROR_ISCAS="https://mirror.iscas.ac.cn/lxc-images"
function select_mirror() {
  case ${MIRROR} in
  "bfsu")
    export LXC_MIRROR=${LXC_MIRROR_BFSU}
    ;;
  "tuna")
    export LXC_MIRROR=${LXC_MIRROR_TUNA}
    ;;
  "nju")
    export LXC_MIRROR=${LXC_MIRROR_NJU}
    ;;
  "iscas")
    export LXC_MIRROR=${LXC_MIRROR_ISCAS}
    ;;
  "main" | "")
    export LXC_MIRROR=${LXC_MIRROR_MAIN}
    ;;
  *)
    echo -e "\033[31mUnknow mirror!\033[0m"
    exit 1
    ;;
  esac
}
```
目前咱就搜集了这几个。
# index文件：
在镜像的/meta/1.0/index-system位置我们可以找到当前的镜像索引，这个文件长这样：
```
almalinux;8;amd64;cloud;20240929_23:08;/images/almalinux/8/amd64/cloud/20240929_23:08/
almalinux;8;amd64;default;20240929_23:08;/images/almalinux/8/amd64/default/20240929_23:08/
almalinux;8;arm64;cloud;20240929_23:08;/images/almalinux/8/arm64/cloud/20240929_23:08/
almalinux;8;arm64;default;20240929_23:08;/images/almalinux/8/arm64/default/20240929_23:08/
almalinux;9;amd64;cloud;20240929_23:08;/images/almalinux/9/amd64/cloud/20240929_23:08/
almalinux;9;amd64;default;20240929_23:08;/images/almalinux/9/amd64/default/20240929_23:08/
almalinux;9;arm64;cloud;20240929_23:08;/images/almalinux/9/arm64/cloud/20240929_23:08/
almalinux;9;arm64;default;20240929_23:08;/images/almalinux/9/arm64/default/20240929_23:08/
alpine;3.17;amd64;cloud;20240929_13:00;/images/alpine/3.17/amd64/cloud/20240929_13:00/
alpine;3.17;amd64;default;20240929_13:00;/images/alpine/3.17/amd64/default/20240929_13:00/
alpine;3.17;arm64;cloud;20240929_13:00;/images/alpine/3.17/arm64/cloud/20240929_13:00/
alpine;3.17;arm64;default;20240929_13:00;/images/alpine/3.17/arm64/default/20240929_13:00/
alpine;3.17;armhf;cloud;20240929_13:00;/images/alpine/3.17/armhf/cloud/20240929_13:00/
alpine;3.17;armhf;default;20240929_13:00;/images/alpine/3.17/armhf/default/20240929_13:00/
alpine;3.18;amd64;cloud;20240929_13:00;/images/alpine/3.18/amd64/cloud/20240929_13:00/
```
可以看到格式非常清晰明了：
```
os;version;arch;type;time;dir
os;version;arch;type;time;dir
```
# 获取cpu架构：
直接从tmoe抄一段就行：
```sh
function get_cpu_arch() {
  # It will create a global variable CPU_ARCH
  # From tmoe
  if [[ $(command -v dpkg) && $(command -v apt-get) ]]; then
    DPKG_ARCH=$(dpkg --print-architecture)
    case ${DPKG_ARCH} in
    armel) ARCH_TYPE="armel" ;;
    armv7* | armv8l | armhf | arm) ARCH_TYPE="armhf" ;;
    aarch64 | arm64* | armv8* | arm*) ARCH_TYPE="arm64" ;;
    i*86 | x86) ARCH_TYPE="i386" ;;
    x86_64 | amd64) ARCH_TYPE="amd64" ;;
    *) ARCH_TYPE=${DPKG_ARCH} ;;
    esac
  else
    UNAME_ARCH=$(uname -m)
    case ${UNAME_ARCH} in
    armv7* | armv8l) ARCH_TYPE="armhf" ;;
    armv[1-6]*) ARCH_TYPE="armel" ;;
    aarch64 | armv8* | arm64 | arm*) ARCH_TYPE="arm64" ;;
    x86_64 | amd64) ARCH_TYPE="amd64" ;;
    i*86 | x86) ARCH_TYPE="i386" ;;
    s390*) ARCH_TYPE="s390x" ;;
    ppc*) ARCH_TYPE="ppc64el" ;;
    mips64) ARCH_TYPE="mips64el" ;;
    mips*) ARCH_TYPE="mipsel" ;;
    risc*) ARCH_TYPE="riscv64" ;;
    *) ARCH_TYPE=${UNAME_ARCH} ;;
    esac
  fi
  export CPU_ARCH=${ARCH_TYPE}
}
```
# 获取镜像：
需要设置好LXC_MIRROR，DISTRO，VERSION，CPU_ARCH这三个变量，
```sh
PATH=$(curl -sL "${LXC_MIRROR}/meta/1.0/index-system" | grep ${DISTRO} | grep ${VERSION} | grep ${CPU_ARCH} | grep -v cloud | tail -n 1 | cut -d ";" -f 6)
echo ${LXC_MIRROR}${PATH}rootfs.tar.xz
```
这样就可以获取到rootfs的链接了，还是相当简单的。
# 搜索镜像：
## 列出所有镜像：
```sh
function list_distros() {
  # It will print the distro name if any version of distro is available for current ${CPU_ARCH}
  # $MIRROR and $CPU_ARCH are defined at main()
  if [[ ${CPU_ARCH} == "" ]]; then
    get_cpu_arch
  fi
  export MIRROR=${MIRROR}
  select_mirror
  for i in $(curl -sL ${LXC_MIRROR}/meta/1.0/index-system | grep ${CPU_ARCH} | grep -v cloud | cut -d ";" -f 1 | uniq); do
    echo -e "[${CPU_ARCH}] $i"
  done
}
```
## 搜索镜像版本号：
```sh
function list_distro_version() {
  # If the version of the distro is available, print it
  # $MIRROR, $DISTRO and $CPU_ARCH are defined at main()
  if [[ ${DISTRO} == "" ]]; then
    echo -e "\033[31mOS distro not set.\033[0m"
    exit 1
  fi
  if [[ ${CPU_ARCH} == "" ]]; then
    get_cpu_arch
  fi
  export MIRROR=${MIRROR}
  select_mirror
  if [[ $(curl -sL "${LXC_MIRROR}/meta/1.0/index-system" | grep ${DISTRO} | grep ${CPU_ARCH}) == "" ]]; then
    echo -e "\033[31mCould not found image for current cpu architecture.\033[0m"
    exit 1
  fi
  for i in $(curl -sL "${LXC_MIRROR}/meta/1.0/index-system" | grep ${DISTRO} | grep ${CPU_ARCH} | grep -v cloud | cut -d ";" -f 1,2); do
    echo -e "[${CPU_ARCH}] $(echo $i | cut -d ";" -f 1) : $(echo $i | cut -d ";" -f 2)"
  done
}
```
# 后记：
没有后记喵！散会！