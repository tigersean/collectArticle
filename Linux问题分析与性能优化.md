# Linux问题分析与性能优化



## 排查顺序

整体情况：

1. `top/htop/atop`命令查看进程/线程、CPU、内存使用情况，[CPU使用情况](https://www.lijiaocn.com/soft/chapter1/cpu-usage-analysis.md)；
2. `dstat 2`查看CPU、磁盘IO、网络IO、换页、中断、切换，[系统I/O状态](https://www.lijiaocn.com/soft/chapter3/process_sys_io.md);
3. `vmstat 2`查看内存使用情况，[内存状态](https://www.lijiaocn.com/soft/chapter1/memory-stat.md)；
4. `iostat -d -x 2`查看所有磁盘的IO情况，[系统I/O状态](https://www.lijiaocn.com/soft/chapter3/process_sys_io.md)；
5. `iotop`查看IO靠前的进程，[系统的I/O状态](https://www.lijiaocn.com/soft/chapter3/process_sys_io.md)；
6. `perf top`查看占用CPU最多的函数，[CPU使用情况](https://www.lijiaocn.com/soft/chapter1/cpu-usage-analysis.md)；
7. `perf record -ag -- sleep 15;perf report`查看CPU事件占比，[CPU使用情况](https://www.lijiaocn.com/soft/chapter1/cpu-usage-analysis.md)；
8. `sar -n DEV 2`查看网卡的吞吐，[网卡状态](https://www.lijiaocn.com/soft/chapter1/network-nic-stat.md)；
9. `/usr/share/bcc/tools/filetop -C`查看每个文件的读写情况，[系统的I/O状态](https://www.lijiaocn.com/soft/chapter3/process_sys_io.md)；
10. `/usr/share/bcc/tools/opensnoop`显示正在被打开的文件，[系统的I/O状态](https://www.lijiaocn.com/soft/chapter3/process_sys_io.md)；

进程分析，[进程占用的资源](https://www.lijiaocn.com/soft/chapter1/process-resouce.md)：

1. `pidstat 2 -p 进程号`查看可疑进程CPU使用率变化情况；
2. `pidstat -w -p 进程号 2`查看可疑进程的上下文切换情况；
3. `pidstat -d -p 进程号 2`查看可疑进程的IO情况；
4. `lsof -p 进程号`查看进程打开的文件；
5. `strace -f -T -tt -p 进程号`显示进程发起的系统调用；

协议栈分析，[连接/协议栈状态](https://www.lijiaocn.com/soft/chapter1/network-stat.md)：

1. `netstat -nat|awk '{print awk $NF}'|sort|uniq -c|sort -n`查看连接状态分布；
2. `ss -ntp`或者`netstat -ntp`查看连接队列；

## 方法论

RED方法：监控服务的请求数（Rate）、错误数（Errors）、响应时间（Duration）。Weave Cloud在监控微服务性能时提出的思路。

USE方法：监控系统资源的使用率（Utilization）、饱和度（Saturation）、错误数（Errors）。

![USE方法常见指标分类](/home/ejungon/Documents/收集的文章/Linux问题分析与性能优化.assets/use-metrics.png)

## 性能分析工具

![性能分析工具](/home/ejungon/Documents/收集的文章/Linux问题分析与性能优化.assets/analyst-tool.png)

## CPU分析思路

![CPU指标](/home/ejungon/Documents/收集的文章/Linux问题分析与性能优化.assets/cpu-metrics.png)

![CPU性能分析](/home/ejungon/Documents/收集的文章/Linux问题分析与性能优化.assets/cpu-analyst.png)

![CPU分析工具](/home/ejungon/Documents/收集的文章/Linux问题分析与性能优化.assets/cpu-tools.png)

## 内存分析思路

![内存性能指标](/home/ejungon/Documents/收集的文章/Linux问题分析与性能优化.assets/memory-metrics.png)

![内存性能分析](/home/ejungon/Documents/收集的文章/Linux问题分析与性能优化.assets/cpu-analyst.png)

![内存分析工具](/home/ejungon/Documents/收集的文章/Linux问题分析与性能优化.assets/memory-tools.png)

## IO分析思路

![文件系统IO指标](/home/ejungon/Documents/收集的文章/Linux问题分析与性能优化.assets/file-io-metrics.png)

![IO性能分析](/home/ejungon/Documents/收集的文章/Linux问题分析与性能优化.assets/io-analyst.png)

![IO分析工具](/home/ejungon/Documents/收集的文章/Linux问题分析与性能优化.assets/file-io-tools.png)

## 网络分析思路

![网络性能指标](/home/ejungon/Documents/收集的文章/Linux问题分析与性能优化.assets/net-metrics.png)

![网络性能分析](/home/ejungon/Documents/收集的文章/Linux问题分析与性能优化.assets/net-analyst.png)

![网络性能工具](/home/ejungon/Documents/收集的文章/Linux问题分析与性能优化.assets/net-tools.png)

## 基准测试工具

![基准测试工具](/home/ejungon/Documents/收集的文章/Linux问题分析与性能优化.assets/benchmark-tool.png)