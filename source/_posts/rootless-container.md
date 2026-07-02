---
title: rootless容器开发指北
date: 2024-10-31 20:06:14
tags:
  - Linux
  - Docker
  - container
cover: /img/rootless.jpg
top_img: /img/rootless.jpg
---
# 前言：
ruri前不久通过使用uidmap二进制的方式修好了rootless容器无法setgroups()的问题，差不多也该讲讲rootless容器的创建了。
# rootless容器创建流程：
## 1.设置uidmap
我们可以通过读取/etc/subuid和/etc/subgid来获取uid_lower,uid_count和gid_lower,gid_count,
他们的格式为：`foo:lower:count`.
然后，我们的父进程fork出子进程，父进程unshare(CLONE_NEWUSER),子进程稍作等待，等父进程拥有了user ns后调用uidmap更改父进程的uidmap，然后父进程获得user ns中的root权限，继续容器创建过程。
```C
pid_t pid_1 = fork();
if (pid_1 > 0) {
		// Enable user namespace.
		try_unshare(CLONE_NEWUSER);
		int stat = 0;
		waitpid(pid_1, &stat, 0);
	} else {
		// To ensure that unshare(2) finished in parent process.
		usleep(1000);
		int stat = try_setup_idmap(ppid, uid, gid);
		exit(stat);
	}
```
具体如何设置uidmap：
```C
static int try_execvp(char *_Nonnull argv[])
{
	/*
	 * fork(2) and then execvp(3).
	 * Return the exit status of the child process.
	 */
	int pid = fork();
	if (pid == 0) {
		execvp(argv[0], argv);
		// If execvp(3) failed, exit as error status 114.
		exit(114);
	}
	int status = 0;
	waitpid(pid, &status, 0);
	return WEXITSTATUS(status);
}
static int try_setup_idmap(pid_t ppid, uid_t uid, gid_t gid)
{
	/*
	 * Try to use `uidmap` suid binary to setup uid and gid map.
	 * If this function failed, we will use set_id_map() to setup the id map.
	 */
	struct ID_MAP id_map = get_idmap(uid, gid);
	// Failed to get /etc/subuid or /etc/subgid config for current user.
	if (id_map.gid_lower == 0 || id_map.uid_lower == 0) {
		return -1;
	}
	char ppid_text[128] = { '\0' };
	char uid_text[128] = { '\0' };
	char uid_lower[128] = { '\0' };
	char uid_count[128] = { '\0' };
	char gid_text[128] = { '\0' };
	char gid_lower[128] = { '\0' };
	char gid_count[128] = { '\0' };
	sprintf(ppid_text, "%d", ppid);
	sprintf(uid_text, "%d", id_map.uid);
	sprintf(uid_lower, "%d", id_map.uid_lower);
	sprintf(uid_count, "%d", id_map.uid_count);
	sprintf(gid_text, "%d", id_map.gid);
	sprintf(gid_lower, "%d", id_map.gid_lower);
	sprintf(gid_count, "%d", id_map.gid_count);
	// In fact I don't know how it works, but it works.
	// newuidmap pid 0 uid 1 1 uid_lower uid_count
	char *newuidmap[] = { "newuidmap", ppid_text, "0", uid_text, "1", "1", uid_lower, uid_count, NULL };
	if (try_execvp(newuidmap) != 0) {
		return -1;
	}
	// newgidmap pid 0 gid 1 1 gid_lower gid_count
	char *newgidmap[] = { "newgidmap", ppid_text, "0", gid_text, "1", "1", gid_lower, gid_count, NULL };
	if (try_execvp(newgidmap) != 0) {
		return -1;
	}
	return 0;
}
```
## 2.获取其他namespace的所有权：
为了使用mount(2),我们还需要CLONE_NEWNS来成为当前mount ns的所有者。
为了挂载procfs，我们需要CLONE_NEWPID，很怪，但实测没有这个不行。
```C
// We need to own mount namespace.
try_unshare(CLONE_NEWNS);
// Seems we need to own a new pid namespace for mount procfs.
try_unshare(CLONE_NEWPID);
```
记得fork()一次以让子进程加入创建的ns。
在这里，你也可以unshare来创建其他命名空间，如cgroup，ipc等。
注意，创建命名空间后fork()一次，这里得到的pid就是我们的ns_pid了。      
## 3.挂载目录：
sysfs从需要从宿主机挂载，由于我们无权mknod，/dev下设备需要从宿主机映射过去。
```C
static void init_rootless_container(struct CONTAINER *_Nonnull container)
{
	/*
	 * For rootless container, the way to create/mount runtime dir/files is different.
	 * as we don't have the permission to mount sysfs and mknod(2),
	 * we need to bind-mount some dirs/files from host.
	 */
	chdir(container->container_dir);
	mkdir("./sys", S_IRUSR | S_IWUSR | S_IROTH | S_IWOTH | S_IRGRP | S_IWGRP);
	mount("/sys", "./sys", NULL, MS_BIND | MS_REC, NULL);
	mkdir("./proc", S_IRUSR | S_IWUSR | S_IROTH | S_IWOTH | S_IRGRP | S_IWGRP);
	mount("proc", "./proc", "proc", MS_NOSUID | MS_NOEXEC | MS_NODEV, NULL);
	mkdir("./dev", S_IRUSR | S_IWUSR | S_IROTH | S_IWOTH | S_IRGRP | S_IWGRP);
	mount("tmpfs", "./dev", "tmpfs", MS_NOSUID, "size=65536k,mode=755");
	close(open("./dev/tty", O_WRONLY | O_CREAT | O_TRUNC | O_CLOEXEC, S_IRUSR | S_IWUSR | S_IROTH | S_IWOTH | S_IRGRP | S_IWGRP));
	mount("/dev/tty", "./dev/tty", NULL, MS_BIND, NULL);
	close(open("./dev/console", O_WRONLY | O_CREAT | O_TRUNC | O_CLOEXEC, S_IRUSR | S_IWUSR | S_IROTH | S_IWOTH | S_IRGRP | S_IWGRP));
	mount("/dev/console", "./dev/console", NULL, MS_BIND, NULL);
	close(open("./dev/null", O_WRONLY | O_CREAT | O_TRUNC | O_CLOEXEC, S_IRUSR | S_IWUSR | S_IROTH | S_IWOTH | S_IRGRP | S_IWGRP));
	mount("/dev/null", "./dev/null", NULL, MS_BIND, NULL);
	close(open("./dev/ptmx", O_WRONLY | O_CREAT | O_TRUNC | O_CLOEXEC, S_IRUSR | S_IWUSR | S_IROTH | S_IWOTH | S_IRGRP | S_IWGRP));
	mount("/dev/ptmx", "./dev/ptmx", NULL, MS_BIND, NULL);
	close(open("./dev/random", O_WRONLY | O_CREAT | O_TRUNC | O_CLOEXEC, S_IRUSR | S_IWUSR | S_IROTH | S_IWOTH | S_IRGRP | S_IWGRP));
	mount("/dev/random", "./dev/random", NULL, MS_BIND, NULL);
	close(open("./dev/urandom", O_WRONLY | O_CREAT | O_TRUNC | O_CLOEXEC, S_IRUSR | S_IWUSR | S_IROTH | S_IWOTH | S_IRGRP | S_IWGRP));
	mount("/dev/urandom", "./dev/urandom", NULL, MS_BIND, NULL);
	close(open("./dev/zero", O_WRONLY | O_CREAT | O_TRUNC | O_CLOEXEC, S_IRUSR | S_IWUSR | S_IROTH | S_IWOTH | S_IRGRP | S_IWGRP));
	mount("/dev/zero", "./dev/zero", NULL, MS_BIND, NULL);
	symlink("/proc/self/fd", "./dev/fd");
	symlink("/proc/self/fd/0", "./dev/stdin");
	symlink("/proc/self/fd/1", "./dev/stdout");
	symlink("/proc/self/fd/2", "./dev/stderr");
	symlink("/dev/null", "./dev/tty0");
	mkdir("./dev/pts", S_IRUSR | S_IWUSR | S_IROTH | S_IWOTH | S_IRGRP | S_IWGRP);
	mount("devpts", "./dev/pts", "devpts", 0, "gid=4,mode=620");
	mkdir("./dev/shm", S_IRUSR | S_IWUSR | S_IROTH | S_IWOTH | S_IRGRP | S_IWGRP);
	mount("tmpfs", "./dev/shm", "tmpfs", MS_NOSUID | MS_NOEXEC | MS_NODEV, "mode=1777");
	mount("./proc/bus", "./proc/bus", NULL, MS_BIND | MS_REC, NULL);
	mount("./proc/bus", "./proc/bus", NULL, MS_BIND | MS_RDONLY | MS_REMOUNT, NULL);
	mount("./proc/fs", "./proc/fs", NULL, MS_BIND | MS_REC, NULL);
	mount("./proc/fs", "./proc/fs", NULL, MS_BIND | MS_RDONLY | MS_REMOUNT, NULL);
	mount("./proc/irq", "./proc/irq", NULL, MS_BIND | MS_REC, NULL);
	mount("./proc/irq", "./proc/irq", NULL, MS_BIND | MS_RDONLY | MS_REMOUNT, NULL);
	mount("./proc/sys", "./proc/sys", NULL, MS_BIND | MS_REC, NULL);
	mount("./proc/sys", "./proc/sys", NULL, MS_BIND | MS_RDONLY | MS_REMOUNT, NULL);
	mount("./proc/sys-trigger", "./proc/sys-trigger", NULL, MS_BIND | MS_REC, NULL);
	mount("./proc/sys-trigger", "./proc/sys-trigger", NULL, MS_BIND | MS_RDONLY | MS_REMOUNT, NULL);
	mount("tmpfs", "./proc/asound", "tmpfs", MS_RDONLY, NULL);
	mount("tmpfs", "./proc/acpi", "tmpfs", MS_RDONLY, NULL);
	mount("/dev/null", "./proc/kcore", "", MS_BIND, NULL);
	mount("/dev/null", "./proc/keys", "", MS_BIND, NULL);
	mount("/dev/null", "./proc/latency_stats", "", MS_BIND, NULL);
	mount("/dev/null", "./proc/timer_list", "", MS_BIND, NULL);
	mount("/dev/null", "./proc/timer_stats", "", MS_BIND, NULL);
	mount("/dev/null", "./proc/sched_debug", "", MS_BIND, NULL);
	mount("tmpfs", "./proc/scsi", "tmpfs", MS_RDONLY, NULL);
	mount("tmpfs", "./sys/firmware", "tmpfs", MS_RDONLY, NULL);
	mount("tmpfs", "./sys/devices/virtual/powercap", "tmpfs", MS_RDONLY, NULL);
	mount("tmpfs", "./sys/block", "tmpfs", MS_RDONLY, NULL);
}
```
## 4.chroot并设置capabilitys：
```C
chdir(container->container_dir);
chroot(".");
chdir("/");
// Fix /etc/mtab.
remove("/etc/mtab");
unlink("/etc/mtab");
symlink("/proc/mounts", "/etc/mtab");
drop_caps(container);
usleep(2000);
```
```C
static void drop_caps(const struct CONTAINER *_Nonnull container)
{
	/*
	 * Drop CapBnd and CapAmb as the config in container->drop_caplist[].
	 * And clear CapInh.
	 * It will be called after chroot(2).
	 */
	for (int i = 0; i < CAP_LAST_CAP + 1; i++) {
		// INIT_VALUE is the end of drop_caplist[].
		if (container->drop_caplist[i] == INIT_VALUE) {
			break;
		}
		// Check if the capability is supported in the kernel,
		// so that we can avoid unnecessary warnings.
		if (CAP_IS_SUPPORTED(container->drop_caplist[i])) {
			// Drop CapBnd.
			if (cap_drop_bound(container->drop_caplist[i]) != 0 && !container->no_warnings) {
				warning("{yellow}Warning: Failed to drop cap `%s`\n", cap_to_name(container->drop_caplist[i]));
				warning("{yellow}error reason: %s{clear}\n", strerror(errno));
			}
			// Drop CapAmb.
			prctl(PR_CAP_AMBIENT, PR_CAP_AMBIENT_LOWER, container->drop_caplist[i], 0, 0);
		}
	}
	// Clear CapInh.
	// hrdp and datap are two pointers, so we malloc() to apply the memory for it first.
	cap_user_header_t hrdp = (cap_user_header_t)malloc(sizeof(typeof(*hrdp)));
	cap_user_data_t datap = (cap_user_data_t)malloc(sizeof(typeof(*datap)) * 10);
	hrdp->pid = getpid();
	hrdp->version = _LINUX_CAPABILITY_VERSION_3;
	capget(hrdp, datap);
	datap->inheritable = 0;
	capset(hrdp, datap);
	free(hrdp);
	free(datap);
}
```
## 5.运行命令:
```C
execvp(container->command[0], container->command);
```
于是我们完成了rootless容器的创建和运行。
# 统一ns：
在容器完成创建后，我们想要运行新的命令怎么办？
非常简单，setns()加入ns_pid的user ns，这时候uid和gid map已经设置好了无需再次设置，然后依次加入其他ns即可。 
# 问题修复：
在解压rootfs时如果遇到权限问题，可以使用`proot -0 tar -xxxxx`来解压。
如果在容器中遇到`Couldn't create temporary file /tmp/apt.conf.sIKx3J for passing config to apt-key`这样的错误，请`chmod 777 /tmp`