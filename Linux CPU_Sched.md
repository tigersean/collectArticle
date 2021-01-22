# Load Average 

> Load Average的定义见系统负载Load Averages的含义: load average并不是表示CPU的繁忙程度，而是度量系统整体负载。这个数值是是运行队列（状态R）和等待磁盘I/O（状态D）的任务数的分钟级平均值。
>
> 


## `ps aux`输出的`STAT`字段

在使用`ps aux`命令检查进程，有一列`STAT`字段，可以看到多种状态

- `D` 不可中断睡眠状态（通常是IO）
- `R` 运行中或者可运行的（在运行队列中）状态
- `S` 可中断的睡眠状态 (等待一个事件完成后唤醒)
- `T` 停止状态，或者通过一个任务控制信号或者它正在被跟踪
- `W` 分页中状态（在2.6.xx内核以后不再使用）
- `X` 死亡状态（以后不可见）
- `Z` 不工作（"僵尸"）进程，被终止的进程但是没有被父进程回收

对于BSD格式和其他使用的状态标记，附加字符含义：

- `<` 高优先级（对其他用户不好）
- `N` 低优先级（对其他用户好）
- `L` 在内存中有锁住的页面（针对实时和定制IO）
- `s` 是一个会话领先者
- `l` 是一个多线程（使用`CLONE_THREAD`，类似`NPTL` pthreads那样)
- `+` 在前台进程组

> 在线上维护服务器的时候，经常会遇到犹豫磁盘故障导致进程进入`D`状态
>
> 使用`man ps`可以看到在`PROCESS STATE CODES`段落下有上述进程状态解释

想要找出哪些在运行队列中的进程，可以采用

```
ps r -A
```

列出所有在运行队列中的进程

以下命令列出所有在运行状态的进程和线程

```
ps -o comm,pid,ppid,user,time,etime,start,pcpu,state --sort=comm aH | grep '^COMMAND\|R$'
```

- 按虚拟内存大小排序进程

```
ps awwlx --sort=vsz
```

显示输出

```
F   UID   PID  PPID PRI  NI    VSZ   RSS WCHAN  STAT TTY        TIME COMMAND
1     0     2     0  20   0      0     0 kthrea S    ?          0:00 [kthreadd]
1     0     3     2 -100  -      0     0 migrat S    ?          0:02 [migration/0]
1     0     4     2  20   0      0     0 ksofti S    ?          0:48 [ksoftirqd/0]
1     0     5     2 -100  -      0     0 cpu_st S    ?          0:00 [migration/0]
...
```

- 检查cpu的run queue方法

```
cat /proc/sched_debug | grep -A1 cpu#  | sed -n '/cpu/{N;s/\n/,/;p}' | sed -r 's/ +/ /g' | sort -k6,6nr |head -5 && cat /proc/loadavg
```

 CPU affinity for all running userspace threads

```
find -L /proc/[0-9]*/exe ! -type l | cut -d / -f3 | \
  xargs -l -i sh -c 'ps -p {} -o comm=; taskset -acp {}'
```