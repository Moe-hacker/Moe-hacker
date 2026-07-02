---
title: C语言实现简单的pstree(子进程查询)功能
date: 2023-09-19 00:05:32
cover: /img/gnu.jpg
top_img: /img/gnu.jpg
tags:
  - Linux
  - C语言
---
# 前言：
最近开发ruri打算加个容器进程信息显示，由于ruri是C语言写的便决定还是用C实现。    
于是查半天。。。没查到一点相关内容。      
都欺负萌新是吧呜呜呜～     
然后就去看man proc了，有个特殊的文件`/proc/${pid}/task/${tid}/children`能记录子进程号，不过需要内核开启相关配置才行。。。。      
欺负萌新是吧呜呜呜～   
行吧，还是自己写唔喵。      
# 已知条件：
子进程pid永远大于父进程pid，即：
pid>ppid      
/proc/${pid}列表可以列出所有运行中进程的pid号。      
/proc/${pid}/stat(status)文件可以解析出每个进程的ppid。      
其中stat更适合C语言等程序解析，status更适合脚本/人类解析。      
具体内容可以看`man proc`。      
“这题我知道，选D，钝角”      
“寄！”      
（死亡微笑）      
# 开始编程：
我们可以先去看看内核是如何存储进程信息的：      
```C
struct task_struct {
#ifdef CONFIG_THREAD_INFO_IN_TASK
	/*
	 * For reasons of header soup (see current_thread_info()), this
	 * must be the first element of task_struct.
	 */
	struct thread_info		thread_info;
#endif
	unsigned int			__state;

#ifdef CONFIG_PREEMPT_RT
	/* saved state for "spinlock sleepers" */
	unsigned int			saved_state;
#endif
......此处省略114514行。
};
```
这怎么可能看懂啊喵！！！    
只是告诉大家我们要写的东西其实很简单而已嘛喵～      
这种对比。。。你是要进行NSA，啊不，BPF。。。IPC。。。啊对了是PUA啊喂！！！     


好吧我们定义一个结构体pstree：
```C
struct PSTREE
{
    pid_t pid;
    // Child process.
    struct PSTREE *child;
    // The next node, it has the same parent process with current node.
    struct PSTREE *next;
};
```
其中child用于存储pid的子进程信息，next是下一个节点。      
然后我们就可以呼呼大睡啦嘿嘿嘿。。。      
然后我们来创建二叉树(不知不觉就把二叉树学了，之前只用单链表的说)：      
```C
struct PSTREE *add_pid(struct PSTREE *pstree, pid_t pid)
{
  /*
   * If current pstree struct is NULL, add pid info here.
   * Or we try the ->next node.
   */
  if (pstree == NULL)
  {
    pstree = (struct PSTREE *)malloc(sizeof(struct PSTREE));
    pstree->pid = pid;
    pstree->next = NULL;
    pstree->child = NULL;
    return pstree;
  }
  pstree->next = add_pid(pstree->next, pid);
  return pstree;
}
struct PSTREE *add_child(struct PSTREE *pstree, pid_t ppid, pid_t pid)
{
  /*
   * If we find ppid in pstree nodes, add pid to the child struct of its ppid.
   * Or this function will do nothing.
   * So we can always run add_child(pstree,ppid,pid) to check && add a pid.
   */
  if (pstree == NULL)
  {
    return NULL;
  }
  if (pstree->pid == ppid)
  {
    pstree->child = add_pid(pstree->child, pid);
    return pstree;
  }
  pstree->next = add_child(pstree->next, ppid, pid);
  pstree->child = add_child(pstree->child, ppid, pid);
  return pstree;
}
```
其中`add_pid()`函数用于添加同级pstree节点，`add_child()`用于查找包含的pid值等于参数中的ppid的节点并将参数中的pid添加到其ppid的child链表中。      
应该。。。不难理解。。。吧。（对自己的水平表示怀疑）      
下面我们来编写进程信息获取相关的函数，三个函数长得很像，因为都是无脑解析/proc/${pid}/stat      
```C
pid_t get_ppid(pid_t pid)
{
  /*
   * Just a simple function that reads /proc/pid/stat and return the ppid.
   */
  pid_t ppid = 0;
  char path[PATH_MAX];
  sprintf(path, "%s%d%s", "/proc/", pid, "/stat");
  char buf[8192];
  char ppid_buf[256];
  int fd = open(path, O_RDONLY | O_CLOEXEC);
  read(fd, buf, sizeof(buf));
  int j = 0;
  for (unsigned long i = 0; i < sizeof(buf); i++)
  {
    if (j == 3)
    {
      for (int k = 0; buf[k + i] != ' '; k++)
      {
        ppid_buf[k] = buf[k + i];
      }
      break;
    }
    if (buf[i] == ' ')
    {
      j++;
    }
  }
  ppid = atoi(ppid_buf);
  return ppid;
}
char *getpid_name(pid_t pid)
{
  /*
   * Just like above.
   */
  char path[PATH_MAX];
  sprintf(path, "%s%d%s", "/proc/", pid, "/stat");
  char buf[8192];
  char name_buf[PATH_MAX];
  int fd = open(path, O_RDONLY | O_CLOEXEC);
  read(fd, buf, sizeof(buf));
  int j = 0;
  for (unsigned long i = 0; i < sizeof(buf); i++)
  {
    if (j == 1)
    {
      for (int k = 0; buf[k + i + 1] != ')'; k++)
      {
        name_buf[k] = buf[k + i + 1];
        name_buf[k + 1] = '\0';
      }
      break;
    }
    if (buf[i] == ' ')
    {
      j++;
    }
  }
  char *name = strdup(name_buf);
  return name;
}
char *getpid_stat(pid_t pid)
{
  /*
   * Just like above.
   */
  char path[PATH_MAX];
  sprintf(path, "%s%d%s", "/proc/", pid, "/stat");
  char buf[8192];
  char stat_buf[PATH_MAX];
  int fd = open(path, O_RDONLY | O_CLOEXEC);
  read(fd, buf, sizeof(buf));
  int j = 0;
  for (unsigned long i = 0; i < sizeof(buf); i++)
  {
    if (j == 2)
    {
      for (int k = 0; buf[k + i] != ' '; k++)
      {
        stat_buf[k] = buf[k + i];
        stat_buf[k + 1] = '\0';
      }
      break;
    }
    if (buf[i] == ' ')
    {
      j++;
    }
  }
  char *stat = strdup(stat_buf);
  return stat;
}
```
应该不难看出`get_ppid()`返回指定pid的ppid，其他两个函数的话仅用于显示进程信息。     
下面是信息的输出：      
```C
void print_tree(struct PSTREE *pstree, int depth)
{
  /*
   * How this function works:
   * Print info of current pid.
   * Print info of the child tree of current pid.
   * Print info of ->next node.
   × So that it will walk all the nodes.
   */
  if (pstree == NULL)
  {
    return;
  }
  char *color[] = {"\033[1;38;2;0;0;255m", "\033[1;38;2;0;255;0m", "\033[1;38;2;255;0;0m"};
  if (depth > 0)
  {
    for (int i = 0; i < depth + 1; i++)
    {
      printf("%s: ", color[i % 3]);
    }
    printf("\n");
    for (int i = 0; i < depth; i++)
    {
      printf("%s: ", color[i % 3]);
    }
    printf("%s:·> ", color[depth % 3]);
  }
  char *stat = getpid_stat(pstree->pid);
  char *name = getpid_name(pstree->pid);
  printf("\033[1;38;2;137;180;250m%d\033[1;38;2;245;194;231m (%s) \033[1;38;2;254;228;208m%s\n", pstree->pid, stat, name);
  free(name);
  free(stat);
  print_tree(pstree->child, depth + 1);
  print_tree(pstree->next, depth);
}
```
其中depth用于缩进，我还加了彩色显示。     
其实真正有用的是最后三行啦喵～      
最后我们写一个`pstree()`函数就完成了喵～      
```C
void pstree(pid_t parent)
{
  /*
   * This function gets the pid that is bigger than parent,
   * Try to add them to pstree struct if they are child of parent that is given.
   * then call print_tree to print pid tree info.
   */
  DIR *proc_dir = opendir("/proc");
  struct dirent *file = NULL;
  int len = 0;
  while ((file = readdir(proc_dir)) != NULL)
  {
    if (file->d_type == DT_DIR)
    {
      if (atoi(file->d_name) > parent)
      {
        len++;
      }
    }
  }
  seekdir(proc_dir, 0);
  int pids[len + 1];
  // For passing clang-tidy.
  memset(pids, 0, sizeof(pids));
  int i = 0;
  while ((file = readdir(proc_dir)) != NULL)
  {
    if (file->d_type == DT_DIR)
    {
      if (atoi(file->d_name) > parent)
      {
        pids[i] = atoi(file->d_name);
        i++;
      }
    }
  }
  closedir(proc_dir);
  struct PSTREE *pstree = NULL;
  pstree = add_pid(pstree, parent);
  for (int j = 0; j < len; j++)
  {
    pstree = add_child(pstree, get_ppid(pids[j]), pids[j]);
  }
  print_tree(pstree, 0);
  free(pstree);
}
```
其中parent是父进程id，这些函数加起来就可以完成一个简单的pstree了喵～      
貌似也没啥技术含量。。。