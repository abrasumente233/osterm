# 基于xv6的进程查看器

由 @aprill4 翻译🥰

[English](README.md)

## 简介

哪个程序员不想挂着一个很酷的 `top(1)` ，假装自己是个厉害的黑客呢（😎）？

`top` 提供了实时的进程活动，并提供了一个可以操作进程的交互式接口。如今有许多top的变体，强烈推荐大家试试  [htop](https://github.com/htop-dev/htop) 和 [btop](https://github.com/aristocratos/btop)。

![btop running](images/btop.png)

更酷的是我们将在自己的操作系统上从零实现一个我们自己的 `top` 程序。而在Linux 上实现 `top` 和在我们自己的操作系统上实现 `top` 的最大区别在于我们没有现成的 API，所以我们需要自己动手实现这些 API，这无疑让事情变得更酷了。

> 那么什么是 API 呢？🤔

API 即为 [Application Programming Interface](https://en.wikipedia.org/wiki/API)，通过 API，不同模块可以互相通信。在操作系统中，API 使我们能够与内核通信。拿 `top` 来说，我们需要通过系统调用 [`getpid()`](https://man7.org/linux/man-pages/man2/getpid.2.html)来向内核请求当前正在运行的进程 pid。

幸运的是，我么将在  [xv6](https://pdos.csail.mit.edu/6.828/2018/xv6/book-rev11.pdf)（一个基于教学的操作系统）上来完成我们的工作，在这个操作系统中已经实现了 `getpid()` ，以及一些其他的系统调用。

> 但是我如果想从零实现一个操作系统，并想要从中获得最大的乐趣呢？

当然了！如果你有足够的时间，你完全可以[实现一个](https://rcore-os.github.io/rCore-Tutorial-Book-v3/index.html)[自己的](https://nankai.gitbook.io/ucore-os-on-risc-v64/)[操作系统](https://xiayingp.gitbook.io/build_a_os/)（这里提供一些教学文档）。但为了节省时间，我们还是将基于 xv6 实现，但仍然有许多 API 需要添加。

更好玩的是，我们将不仅仅在虚拟器上运行我们的 `top` 程序，而且会在真正的硬件 — [K210开发板](https://canaan.io/product/kendryteai)（基于 [RISC-V架构](https://en.wikipedia.org/wiki/RISC-V)）上运行。

> 那 RISC-V 又是什么呢？

就好比 [ARM](https://en.wikipedia.org/wiki/ARM_architecture_family) 是 x86 的简易版本，而 RISC-V 又是 ARM 的简易版本。最重要的是，每个人都喜欢 RISC-V。

> 但我并不喜欢……

但那没关系，因为在我们的任务中，并不需要了解 RISC-V。 `xv6-k210` 项目的[贡献者](https://github.com/HUST-OS/xv6-k210/graphs/contributors)已经完成了那些困难的工作。接下来介绍 `xv6-k210` ，这个项目将 xv6 移植到了 K210 开发板上。

我们要实现的是一个 `top` 程序，他是一个交互式的程序，但我们只需要专注于操作系统相关的部分，实现一个简易版的 `top` - `ps` ，只需要以表格的形式打印出进程状态然后退出，而不需要去设计很酷的 UI。

在 Linux 上运行 ps 将会输出以下信息。

```bash
$ ps
    PID TTY          TIME CMD
   5303 pts/0    00:00:00 zsh
   5380 pts/0    00:00:00 ps
```

这里 `ps` 打印了两个进程的相关信息，`zsh` shell 和 `ps` 本身。

> 但现在只能同时运行两个进程，那太简单啦！

没错！但事情当然没有那么简单，在实现 `ps` 之前，让 xv6 shell 支持多进程是必要的，需要利用 `&` 使得能够进程在后台运行。

> 好消息是 xv6 已经可以支持进程在[后台运行](https://github.com/HUST-OS/xv6-k210/blob/main/xv6-user/sh.c#L392-L402)。

```bash
# in the xv6-k210 directory
$ make run platform=qemu
...
-> / $ sleep 100 &       # finishes immediately
-> / $ echo do something else while sleeping
do something else while sleeping
-> / $ 
```
所以我们不必再完成支持进程后台运行这一任务，但我们将会提高难度，具体任务将会在后文详细说明。现在继续回到多进程。

假设我们已经为 xv6 添加了多进程的支持，下一步就是从内核获取进程信息。我们将参考 Linux 系统调用的实现，并利用文件系统在内核和我们的 `ps` 之间通信。

这里提到的文件系统是 [proc filesystem](https://man7.org/linux/man-pages/man5/proc.5.html)， 这个“虚拟”文件系统为用户提供接口去访问内核数据结构，其通常被挂载在 `/proc` ，提供进程信息。例如：

```bash
$ sleep 1337 &
[1] 5690
$ ps
    PID TTY          TIME CMD
   5303 pts/0    00:00:01 zsh
   5690 pts/0    00:00:00 sleep
   5832 pts/0    00:00:00 ps
$ echo The pid is 5569, Let\\\\'s check it out in /proc
The pid is 5569, Let's check it out in /proc
$ ls -l /proc | grep 5690
dr-xr-xr-x  9 ubuntu           ubuntu                         0 Jun 21 10:13 5690
$ cat /proc/5690/stat
5690 (sleep) S 5303 5690 5303 34816 5769 1077936128 143 0 0 0 0 0 0 0 25 5 1 0 12929348 6823936 116 18446744073709551615 187650721452032 187650721479328 281474017591088 0 0 0 0 0 0 1 0 0 17 0 0 0 0 0 0 187650721544968 187650721546472 187650724663296 281474017594807 281474017594818 281474017594818 28147
```

下面对其进行详细说明：

1. 创建一个 `sleep` 进程，并使其在后台运行；
2. 利用 `ps` 可以观察到 `sleep` 进程确实在运行，其 pid 为5690；
3. 在 `/proc` 目录中，有一目录名为 5690，其中包含了进程 `sleep` 的相关信息；
4.  `/proc/5690/stat` 文件中存储着进程相关的状态信息；
5. 查询 [man page](https://man7.org/linux/man-pages/man5/proc.5.html)，可以知道 `/proc/[pid]/stat` 中各项的含义。第一个数字为进程 id，在这里为 5690；
6. 接下来的 (xxxx) ，其中 xxxx 为可执行文件名；
7. 紧接着一个字母，标志进程状态，S 表示正在睡眠；
8. 第四项为进程的父进程 id，在这里是 zsh shell ，其 pid 为5303；
9. 其余各项可在 man ps 手册的 STANDARD FORMAT SPECIFIERS 中查看。

那么现在，我们可以利用 `/proc` 来实现 `ps`。

> 可是 /proc 太难啦！

嗯……那我们将首先从系统调用开始，并逐步实现 `/proc` 。

## 任务0：前导

1. 首先在 GitHub fork [xv6-k210](https://github.com/abrasumente233/xv6-k210) 项目；
2. 实验环境为Linux，可以使用[WSL2](https://docs.microsoft.com/en-us/windows/wsl/install)（注意：WSL1不支持 `mount` 和 usb 设备），VirtualBox 或者是一台 Linux 机器；
3. 打开终端；
4. 如果你使用的是Ubuntu，运行 `sudo apt update && sudo apt install gcc-riscv64-unknown-elf` 安装64位 RISC-V 的编译器。对于其他的 Linux 版本，你可以搜索相应的包管理软件或者[自己编译](https://github.com/riscv-collab/riscv-gnu-toolchain)。
5. `sudo apt install qemu-system-misc` ，安装 RISC-V 的 QEMU模拟器（注意：xv6-k210 不支持在6.2.0或以上版本运行，那将会报错（“Some ROM regions are overlapping"）。所以尝试安装 4.2.1 以下版本或者使用更低版本的 Ubuntu， 如果有更好的解决方法，我们会及时更新文档）；
6. `sudo apt install python3` ，安装python3，因为我们的测试脚本是用 Python 写的；
7. `sudo apt install make git` ，安装一些实验相关软件；
8. `sudo apt install dosfstools` ，安装 mkfs.vfat 工具；
9. `git clone https://github.com/[your_github_username]/xv6-k210.git` ，方括号以及其中内容为你自己的GitHub用户名；
10. `cd xv6-k210`
11. `make fs` 生成一个 FAT32 的文件系统镜像，并将它保存在 `fs.img` ;
12. `make run platform=qemu` ，在 QEMU 上运行 xv6-k210;
13. 经过几分钟漫长的编译之后，xv6 的 shell 将会出现，现在可以开玩！
![xv6 running](images/xv6-running.png)
14. 按下 `Ctrl+A 和 X` 终止QEMU的运行。

xv6-k210 在 K210 开发板上的运行，请查看其[详细的文档](https://github.com/HUST-OS/xv6-k210#run-on-k210-board)。

## 任务1：实现进程相关的系统调用

`ps` 需要我们从内核获取一些进程信息提供给它。所以在这项任务中，我们将利用系统调用为其提供这些信息。具体地，需要完成以下系统调用。

### 系统调用 #1：getppid

`getppid()` 返回当前进程的父进程 id。

```c
// 返回值为父进程PID
pid_t ret = getppid();
```

#### 提示：

1. xv6-k210 中对[如何添加系统调用](https://github.com/HUST-OS/xv6-k210/blob/main/doc/%E7%94%A8%E6%88%B7%E4%BD%BF%E7%94%A8-%E7%B3%BB%E7%BB%9F%E8%B0%83%E7%94%A8.md))有详细说明；
2. 注意 `struct proc` 中有一个 `parent` 成员；
3. 可参考已有的 `getpid()` 实现。

### 系统调用 #2：times

`times()` 获取进程的用户和系统时间，将运行时间相关信息储存在参数 `tms *` 中，并返回程序从运行开始以来的滴答数。

```c
struct tms {
	uint64 utime;		// 用户时间
	uint64 stime;		// 系统时间
	uint64 cutime;		// 子进程的用户时间
	uint64 cstime;		// 子进程的系统时间
}

struct tms t;
clock_t ticks = times(&t);
```

`utime` 为调用进程执行指令的 CPU 时间，`stime` 为内核为发起调用的进程执行任务所用的 CPU 时间。

`cutime` 为 `utime` 和所有等待被终止的子进程的 `cutime` 之和。`cstime` 为 `stime` 和 所有等待被终止的子进程的 `cstime` 之和。

*终止子进程（和其子进程，即子孙进程）的时间在* *`wait(2)` 返回进程 ID 时添加。特别要注意的是，子进程不会等待的子孙进程的时间将不会计算。*

所有时间用滴答数表示。

#### 提示：

1. RISC-V 中有一计时器，可以利用 `riscv.h` 中的函数 `r_time()` 来读取其中数值；
2. 在这个系统调用中，没有必要将滴答数转化为秒，但之后当你需要程序自开始运行以来过去的时间，得到滴答数毫无意义，这时候转化为秒是更推荐的做法。这里给出 QEMU 和 K210 的时钟频率，QEMU 为 12500000 赫兹，K210 为 6500000 赫兹。

### 系统调用 #3：getmem

`getmem()` 以 KiB（1024字节）为单位返回进程的虚拟内存大小。

### 【加分项】系统调用 #4：clone[^1]

`clone` 需要对 xv6 添加多线程支持。线程共享虚拟地址空间，又被称为是轻量的进程，能够并发执行并共享代码、全局变量以及堆。但每一个线程执行时都拥有独立的栈和寄存器。

#### 提示：

1. 可参考 xv6 中[内核线程的实现](https://github.com/kishanpatel22/xv6-kernel-threads#xv6-kernel-threads)；
2. 后期将提供测试样例作为基准。

[^1]: https://github.com/kishanpatel22/xv6-kernel-threads/blob/master/doc/xv6_kernel_thread_project.md

## 任务2：添加信号[^2][^3]

信号也就是软件中断，能够在某一事件发生时通知进程，其可能有以下来源：

1. 内核
2. 进程本身
3. 其他进程

当接收到一个信号时可能有不同的行为：

1. 执行默认操作，如接收到 `SIGINT` 时终止进程;
2. 忽略此信号；
3. 执行用户自定义的信号处理函数。

同样地，有许多发出信号的方法：

1. 进程可通过调用 [`int raise(int sig)`](https://man7.org/linux/man-pages/man3/raise.3.html) 发出信号；
2. 通过 [`int kill(pid_t pid, int sig)`](https://man7.org/linux/man-pages/man2/kill.2.html)向其他进程（包括进程本身）发出信号；
3. 通过[`unsigned int alarm(unsigned int seconds)`](https://man7.org/linux/man-pages/man2/alarm.2.html) 给未来 n 秒后的自己发送一个 `SIGALARM` 信号。

#### 系统调用：`alarm`

在这部分我们将实现一个新的系统调用 `unsigned int alarm(unsigned int seconds)` ，它将在指定的时间之后向调用进程发送 `SIGALARM` 信号，这里无需实现信号接收部分，也就是说发出  `SIGALARM` 信号仅仅 `kill()` 掉进程。

程序`alarmtest.c` 调用 `alarm()` ，然后进入死循环。对于 `alarm`，正确的实现是在指定的秒数之后进程被终止。

```c
// alarmtest.c
#include "xv6-user/user.h"

int main() {
  int pid;
  printf("Alarm testing!\\\\n");

  alarm(5);         // send SIGALARM to calling process after 5 seconds, which means terminating it
  while(1);			// process suspended, waiting for signals to wake up
  printf("unreachable!");

  exit(0);
}
```

#### 系统调用：`signal` 第一步

在上述所说的 `alarm` 中，当我们接收到一个 `SIGALARM` 信号，就将进程终止。而 `signal()` 具有更多功能，我们可以利用其定义接收信号时如何处理。

```c
void (*signal(int sig, void (*func)(int)))(int);
```

这个复杂的函数原型表示 `signal()` 接受两个参数：*sig* 和 *func* ，*func* 指定接收信号 *sig* 时的处理函数，这个函数必须以一个 **int** 作为参数并且其返回类型为 **void**。

`signal()` 函数返回一个同类型的函数，这是 *sig* 信号的旧处理函数，或者以下两个特殊值中的其中一个：

1. `SIG_IGN` 忽略此信号；
2. `SIG_DFL` 为默认操作，即终止进程。

在这一步中，只需要支持 `SIG_IGN` (=1, 忽略) 和 `SIG_DFL` （=0，默认操作，即终止进程）。

利用以下 `alarmtest2.c` 程序测试 `signal()`：

```c
// alarmtest2.c
#include "xv6-user/user.h"

int main()
{
  int pid;

  printf("Alarm testing!\\\\n");

  alarm (5);

  printf("Waiting for alarm to go off\\\\n");
  (void) signal ( SIGALARM, SIG_IGN );

  while(1);			//process suspended, waiting for signals to wake up
  printf("now reachable!\\\\n");

  exit(0);
}
```

#### 系统调用：`signal` 第二步

如果第二个参数为一函数（非 `SIG_DFL` 或 `SIG_ING` ），那么收到信号时就需要利用这个函数来处理，请参考 [Linux 对此的实现](https://www.linuxjournal.com/article/3985)。

**重要**：信号处理函数只能运行在用户态！

`alarmtest3.c` 程序对其进行测试：

```c
// alarmtest3.c
#include "xv6-user/user.h"

//simulates an alarm clock
void ding ( int sig )
{
  printf("Alarm has gone off\\\\n");
}

int main()
{
  int pid;

  printf("Alarm testing!\\\\n");

  alarm (5);

  printf("Waiting for alarm to go off\\\\n");
  (void) signal ( SIGALARM, ding );

  while(1);			//process suspended, waiting for signals to wake up
  printf("Done!\\\\n");

  exit(0);
}
```

#### 提示：

当信号处理函数返回时，进程需要能够从它被中断的地方重新开始。

#### 系统调用：kill

到目前为止，我们只能通过 `alarm` 发出信号，但这一系统调用只能用于在进程内发送与接收信号。

我们已经提到过利用 `kill` 可以在不同进程间发信号，但 xv6 中已有的 `kill` 只能终止其他进程，而不能像我们希望的那样，向其他进程发出信号。所以在这一部分，我们需要将 `kill(int pid)` 扩展为 `kill(int pid, int sig)` ，使得 `kill()`能够：

1. 向由 pid 指定的进程发送信号；
2. 发送任意信号，例如 `SIGINT` 和 `SIGALARM` 。

##### 提示：

`kill(int pid)` 在 xv6 中已经存在，为修改 `kill` 函数，也必须修改对 `kill` 的调用，并且不破坏原本代码。

#### 按下CTRL-C 向前台进程发送 SIGINT 信号

修改 `console.c` 文件中的 `consoleintr()` 函数，使得不管何时按下 `CTRL-C` ，内核将向 `sh` 发送信号 `SIGINT` 。接着 `sh` 将执行其信号处理函数，具体为检查正在前台运行的进程并向其发送 `SIGINT` 。如果无任何进程在前台运行，那么只需要重新打印 shell 的命令提示符。

##### 提示：

使用 `sleep 100` 测试 `Ctrl+C` 。

[^2]: http://cse.csusb.edu/tongyu/courses/cs460/labs/lab3.php

[^3]: https://cs385.class.uic.edu/homeworks/7-implementing-signals/

## 任务3：实现虚拟文件系统 `/proc`[^4]

在 Linux 中，`/proc` 目录下有一系列不需要存储在磁盘上的文件，这些 ”虚拟“ 文件描述了系统的状态、配置以及进程等等。事实上，`/proc` 目录也确实不在磁盘存储，而是在每次被读取时动态生成其中内容。

在这项任务中，我们将基于 xv6 实现 `/proc` 文件系统。

### 虚拟目录列表

利用  `mkdir` 创建目录 `/proc` ，当执行 `ls /proc` 时，将会出现一系列虚拟目录（并不真正存储在磁盘上）。每一个正在运行的进程都拥有自己的目录。

在 xv6 中，一个文件即为一个 `inode` ，`inode` 中含有读写文件内容以及构成 `inode` 数据的函数。这些 `inode` 函数将会调用硬件接口，但事实上`/proc` 在磁盘上并不存在。

所以我们需要修改文件系统相关的代码。更具体地说，现在 `struct inode`  中包含一个叫做 `struct inode_functions` 的指针，其中包含指向读写 `inode` 的函数指针。这里将提供一个 `/proc` inode 的实现。

```c
struct inode_functions {
  void (*ipopulate)(struct inode*);                   // fill in inode details
  void (*iupdate)(struct inode*);                     // write inode details back to disk
  int (*readi)(struct inode*, char*, uint, uint);     // read from file contents
  int (*writei)(struct inode*, char*, uint, uint);    // write to file contents
};

struct inode {
  /// ... other fields ...
  struct inode_functions *i_func; // NEW!
};
```

首先修改 inode 中 的 `i_func` 指针，以便读取 `/proc` 时函数能够被调用。从 `ls.c` 可以观察到如何获取目录列表，然后便可以写一个 `procfs_readi` 函数。`namei` 函数根据给定的文件路径（利用参数进行传递）返回对应的 `struct inode*` 。

需要确保 `ls /proc` 显示正确的文件类型和大小。因此需要实现 `procfs_ipopulate` ，注意此处的`iget()` ：对于 “proc” 文件使用不同的设备号来避免重复获取错误的 inode。

#### 提示：

在向 `struct inode` 添加 `inode_functions` 之后，应该用 `inode->i_func->readi` 来替换原本文件系统中的 `readi` ，其他 `i_func` 中的函数也需被替换。

### 进程目录和文件

对于每个正在运行的进程，对应的目录（`/proc/[pid]`）应该包含一个 `stat` 文件。当执行 `cat /proc/[pid]/stat` 时，应该用空格分隔各项输出。如下所示：

```
pid (command) state ppid utime stime cutime cstime vsz
```

其中：

1. pid: 进程 PID；
2. command: 命令名（即可执行文件名），被包含在括号中；
3. ppid: 父进程的 PID；
4. state: 进程状态。（R - 正在运行或可运行，S - 睡眠，Z - 已经被终止但还没被父进程回收的僵死进程）；
5. utime, stime, cutime, cstime 与 `tms` 结构体中含义相同；
6. vsz: 虚拟内存大小，以 KiB 为单位，可由系统调用 `getmem()` 返回。

### 【加分项】自动创建 /proc 目录并挂载 proc

测试时手动创建 proc 目录是可行的 —— 重启然后利用 `main()` 函数执行挂载。还可以在生成文件系统时创建 `/proc`, 但最好是 `proc` 不存在时，由 xv6 自动创建。

所以，在`init.c` 中利用 `mkdir()` 函数创建 `/proc` 目录，但 `main()` 函数第一次运行时，还是没有 `/proc` 目录。因此，最好创建系统调用 `sys_mount` 并在 `init.c` 中调用它。

[^4]: https://cs385.class.uic.edu/homeworks/8-proc-file-system/

## 任务4：实现 `ps` 命令

在这项任务中，我们将利用以上完成的各项任务来实现 `ps` 。其中包含7项，以下为 Linux 中 `ps` 的输出，供作参考。

```bash
$ ps -o pid,ppid,comm,state,time,etime,vsz
    PID    PPID COMMAND         S     TIME     ELAPSED    VSZ
   5303    5302 zsh             S 00:00:06    08:27:26  14880
   7900    5303 ps              R 00:00:00       00:00  10048
```

其中：

1. PID：进程 PID；
2. PPID：父进程 PID；
3. COMMAND：命令名（可执行文件名）
4. S：进程状态；（R - 正在运行或可运行，S - 睡眠，Z - 已经被终止但还未被父进程回收的僵死进程）
5. TIME：累积的CPU时间
6. ELAPSED：从进程开始运行以来的时间，格式：`[[DD-]hh:]mm:ss`；
7. VSZ: 虚拟内存大小，以 KiB 为单位。

### 提示：

1. `ps` 利用 `/proc` 目录，遍历每一个子目录，从子目录名解析 pid, 并从 `/proc/[pid]/stat` 读取进程信息。
2. 可参考[StackOverflow的回答](https://stackoverflow.com/questions/23607980/implementing-my-own-ps-command)  或者 [busybox中`ps` 的实现](https://github.com/mirror/busybox/blob/master/procps/ps.c) 。

## 参考资源

1. xv6 手册: https://pdos.csail.mit.edu/6.828/2018/xv6/book-rev11.pdf，以及其中文译本： https://th0ar.gitbooks.io/xv6-chinese/content/
2. build a OS（关于 xv6 的笔记）：https://xiayingp.gitbook.io/build_a_os/
