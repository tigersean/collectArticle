一*.* 进程*Context* 定义

​    当一个进程在执行时*, CPU*的所有寄存器中的值、进程的状态以及堆栈中的内容*,*比如各个变量和数据，包括所有的寄存器变量、进程打开的文件、内存信息等。这些信息被称为该进程的上下文*(Context)**.*

 

​    一个进程的*Context*可以分为三个部分*:*用户级上下文、寄存器上下文以及系统级上下文*:*

​    （*1*）用户级上下文*:* 正文、数据、用户堆栈以及共享存储区；*
​    （*2*）寄存器上下文*:* 通用寄存器、程序寄存器*(IP)*、处理器状态寄存器*(EFLAGS)*、栈指针*(ESP)*；*
​    （*3*）系统级上下文*:* 进程控制块*task_struct*、内存管理信息*(mm_struct*、*vm_area_struct*、*pgd*、*pte)*、内核栈。

 

​    当内核需要切换到另一个进程时，它需要保存当前进程的所有状态，即保存当前进程的上下文，以便在再次执行该进程时，能够必得到切换时的状态执行下去。在*LINUX*中，当前进程上下文均保存在进程的任务数据结构中。在发生中断时*,*内核就在被中断进程的上下文中，在内核态下执行中断服务例程。但同时会保留所有需要用到的资源，以便中继服务结束时能恢复被中断进程 的执行。

 

​    在*Linux* 的*Vmstat* 命令中的*system*下，有*interrupts* 和 *context switching* 选项。

 

*[root@db1 ~]# vmstat 2*

*procs -----------memory---------- ---swap-- -----io---- --system-- -----cpu------*

 *r b swpd free buff cache si  so bi  bo in  cs us sy id wa st*

 *0 0 23524 13660 62952 651628 0  0  6  14  45 34 0 1 99 0 0*

 *1 0 23524 13660 62952 651656 0  0  0  0  93  62 0 0 100 0 0*

 

其中的*in* 表示*interrupts*， *cs* 表示*contxt switching*。

*in: The number of interrupts per second, including the clock.*

*cs: The number of context switches per second.*

 

​    *Unix vmstat* 命令

​    *http://www.cndba.cn/Dave/article/685*

 

 

二*. Interrupts and Context Switching*

​    *To drastically simplify how computers work, you could say that computers do nothing more that shuffle bits (i.e. 1s and 0s) around. All computer data is based on these binary digits, which are represented in computers as voltages (5 V for a 1 and 0 V for a 0), and these voltages are physically manipulated through transistors, circuits, etc... When you get into the guts of a computer and start looking at how they work, it seems amazing how many operations it takes to do something simple, like addition or multiplication. Of course, computers have gotten a lot smaller and thus a lot faster, to the point where they can perform millions of these operations per second, so it still feels fast. The processor is performing these operations in a serial fashion - basically a single-file line of operations.*

 

*1.1 what is Context switching*

​    *This single-file line could be quite inefficent and there are times when you want a computer to be processing many different things at once, rather than one thing at a time. For example, most computers rely on peripherals for input, but those peripherals are often much slower than the processor itself. For instance, when a program needs some data, it may have to read that data from the hard drive first. This may only take a few milliseconds, but the CPU would be idle during that time - quite inefficient. To improve efficiency, computers use multitasking. A CPU can still only be running one process at a time, but multitasking gets around that by scheduling which tasks will be running at any given time. The act of switching from one task to another is called [Context Switching](http://en.wikipedia.org/wiki/Context_switching).*

​    *--CPU* 一次只能运行一个进程，在*multitasking* 下， 当*CPU* 处理的进行需要从硬盘读取一些数据时，虽然这个时间也是按*miliseconds*来计算，但是这个时间还是要比*CPU* 运算的时间长，所以在*CPU* 的这个等待的时间里，可以将当前处理的*task* 切换出去，让*CPU* 去处理其他的*task*。 这个过程就叫作*Context switch*。

 

​    *Ironically, the act of context switching adds a fair amount of overhead to the computing process. To ensure that the original running program does not lose all its progress, the computer must first save the current state of the CPU in memory before switching to the new program. Later, when switching back to the original, the computer must load the state of the CPU from memory. Fortunately, this overhead is often offset by the efficiency gained with frequent context switches.*

​    *--*为了保证之前的程序没有丢失它的进程， 电脑必须先在内存中保存当前*CPU*的状态，然后才能进行*task switch*。

​    当要切换到之前的*task*时， 将之前保存在内存中的*cpu*状态装载到*CPU*就可以了。

 

*1.2 Context switch: steps*

​    *In a switch, the state of the first process must be saved somehow, so that, when the scheduler gets back to the execution of the first process, it can restore this state and continue.*

​    *The state of the process includes* *all the registers that the process may be using, especially the [program counter](http://en.wikipedia.org/wiki/Program_counter), plus any other operating system specific data that may be necessary. This data is usually stored in a data structure called a [process control block](http://en.wikipedia.org/wiki/Process_control_block) (PCB), or [switchframe](http://en.wikipedia.org/wiki/Switchframe).*

​    *-- process* 中的信息保存在一个叫*PCB*的数据结构中。

​    *In order to switch processes, the PCB for the first process must be created and saved. The PCBs are sometimes stored upon a per-process [stack](http://en.wikipedia.org/wiki/Stack_(computing)) in kernel memory (as opposed to the user-mode stack), or there may be some specific operating system defined data structure for this information.*

​    *--*在进行*switch processes*时，必须先创建当前运行*process*的*PCB* 并将该*PCB* 保存到*kernel memory*中。

 

​    *Since the operating system has effectively suspended the execution of the first process, it can now load the PCB and context of the second process. In doing so, the program counter from the PCB is loaded, and thus execution can continue in the new process. New processes are chosen from a queue or queues. Process and thread priority can influence which process continues execution, with processes of the highest priority checked first for ready threads to execute.*

​    *--* 将根据进程的优先级从队列中选择装载哪个进程去运行。

 

 

*1.3 What is Interrupt?*

​    *If you can do context switches frequently enough, the computer appears to be doing many things at once (even though the CPU is only processing a single task at any given time). Signaling the CPU to do a context switch is often accomplished with the use of a command called an [Interrupt](http://en.wikipedia.org/wiki/Interrupt). For the most part, the computers we're all using are Interrupt driven, meaning that running processes are often interrupted by higher-priority requests, forcing context switches.*

​    *--* 虽然*CPU* 一次只能处理一个进程，但只要*context switch* 足够频繁，我们还是可以感觉*CPU* 同时在处理多个*tasks.* 而*CPU* 根据指令去进行*context switch* 的过程就叫做*Interrupt.*

​    *Interrupt* 可以是有高优先级的进程申请，或者是强制的*context switches*。

 

*From*：*http://kaedrin.com/weblog/archive/001657.html*

 

 

三*. Interrupt Context* 与 *Process Context*

 

处理器总处于以下三种状态之一：

​    *1.*内核态，运行于*Process context*，内核代表进程运行于内核空间；*
​    2.*内核态，运行于*Interrupt Context*，内核代表硬件运行于内核空间；*
​    3.*用户态，运行于用户空间。

 

​    用户空间的应用程序，通过系统调用，进入内核空间。这个时候用户空间的进程要传递很多变量、参数的值给内核，内核态运行的时候也要保存用户进程的一些寄存 器值、变量等。

​    进程上下文*:* 可以看作是用户进程传递给内核的这些参数以及内核要保存的那一整套的变量和寄存器值和当时的环境等。

*
\*    硬件通过触发信号，导致内核调用中断处理程序，进入内核空间。这个过程中，硬件的一些变量和参数也要传递给内核，内核通过这些参数进行中断处理。   中断上下文*:* 可以看作就是硬件传递过来的这些参数和内核需要保存的一些其他环境（主要是当前被打断执行的进程环境）。

 

 

*3.1 Interrupt Context*

​    *When executing an interrupt handler or bottom half, the kernel is in interrupt context. Recall that process context is the mode of operation the kernel is in while it is executing on behalf of a process -- for example, executing a system call or running a kernel thread. In process context, the current macro points to the associated task. Furthermore, because a process is coupled to the kernel in process context(*因为进程是以进程上文的形式连接到内核中的*), process context can sleep or otherwise invoke the scheduler.*

 

​    *Interrupt context, on the other hand, is not associated with a process. The current macro is not relevant (although it points to the interrupted process). Without a backing process(*由于没有进程的背景*),* *interrupt context cannot sleep -- how would it ever reschedule?(*否则怎么再对它重新调度*?) Therefore, you cannot call certain functions from interrupt context. If a function sleeps, you cannot use it from your interrupt handler -- this limits the functions that one can call from an interrupt handler.(*这是对什么样的函数可以在中断处理程序中使用的限制*)*

 

​    *Interrupt context is time critical because the interrupt handler interrupts other code. Code should be quick and simple. Busy looping is discouraged. This is a very important point; always keep in mind that your interrupt handler has interrupted other code(possibly even another interrupt handler on a different line!). Because of this asynchronous nature, it is imperative(*必须*) that all interrupt handlers be as quick and as simple as possible. As much as possible, work should be pushed out from the interrupt handler and performed in a bottom half, which runs at a more convenient time.*

 

​    *The setup of an interrupt handler's stacks is a configuration option. Historically, interrupt handlers did not receive their own stacks. Instead, they would share the stack of the process that they interrupted[1]. The kernel stack is two pages in size; typically, that is 8KB on 32-bit architectures and 16KB on 64-bit architectures. Because in this setup interrupt handlers share the stack, they must be exceptionally frugal(*必须非常节省*) with what data they allocate there. Of course, the kernel stack is limited to begin with, so all kernel code should be cautious.*

​    *[1] A process is always running. When nothing else is schedulable, the idle task runs.*

 

​    *Early in the 2.6 kernel process, an option was added to reduce the stack size from two pages down to one, providing only a 4KB stack on 32-bit systems. This reduced memory pressure because every process on the system previously needed two pages of nonswappable kernel memory. To cope with(*应对*) the reduced stack size, i**nterrupt handlers were given their own stack, one stack per processor, one page in size. This stack is referred to as the interrupt stack(*这个栈就程为中断栈*). Although the total size of the interrupt stack is half that of the original shared stack, the average stack space available is greater because interrupt handlers get the full page of memory to themselves.*

​    *Your interrupt handler should not care what stack setup is in use or what the size of the kernel stack is. Always use an absolute minimum amount of stack space.*

 

*3.2 Process Context*

​    *One of the most important parts of a process is the executing program code. This code is read in from an executable file and executed within the program's address space. Normal program execution occurs in user-space. When a program executes a system call or triggers an exception, it enters kernel-space. At this point, the kernel is said to be "executing on behalf of the process" and is in process context. When in process context, the current macro is valid[7]. Upon exiting the kernel, the process resumes execution in user-space, unless a higher-priority process has become runnable in the interim(*过渡期*), in which case the scheduler is invoked to select the higher priority process.*

 

​    *[7] Other than process context there is interrupt context, In interrupt context, the system is not running on behalf of a process, but is executing an interrupt handler. There is no process tied to interrupt handlers and consequently no process context.*

​    *System calls and exception handlers are well-defined interfaces into the kernel. A process can begin executing in kernel-space only through one of these interfaces -- all access to the kernel is through these interfaces.*

 

*3.3 Process Context* 和 *Interrupt Context* 区别

​    *1. The main difference between process context and interrupt context is that process context can block, while interrupt context can’t.*

​    *2.Interrupt context mean that there will not be scheduling taking place until and unless that task ends while in the process context every task gets the CPU time*

 

 