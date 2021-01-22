# 1、CFS的基本思路

在CFS算法引入之前，Linux使用过几种不同的调度算法，一开始的调度器是复杂度为O(n)的始调度算法(实际上每次会遍历所有任务，所以复杂度为O(n)), 这个算法的缺点是当内核中有很多任务时，调度器本身就会耗费不少时间，所以，从linux2.5开始引入赫赫有名的O(1)调度器，然而，linux是集全球很多程序员的聪明才智而发展起来的超级内核，没有最好，只有更好，在O(1)调度器风光了没几天就又被另一个更优秀的调度器取代了，它就是CFS调度器Completely Fair Scheduler. 这个也是在2.6内核中引入的，具体为2.6.23，即从此版本开始，内核使用CFS作为它的默认调度器，O(1)调度器被抛弃了。：

> - O(n)调度：内核调度算法理解起来简单：在每次进程切换时，内核依次扫描就绪队列上的每一个进程，计算每个进程的优先级，再选择出优先级最高的进程来运行；尽管这个算法理解简单，但是它花费在选择优先级最高进程上的时间却不容忽视。系统中可运行的进程越多，花费的时间就越大，时间复杂度为O(n)
> - O(1)调度：其基本思想是根据进程的优先级进行调度。进程有两个优先级,一个是静态优先级,一个是动态优先级.静态优先级是用来计算进程运行的时间片长度的,动态优先级是在调度器进行调度时用到的,调度器每次都选取动态优先级最高的进程运行.由于其数据结构设计上采用了一个优先级数组，这样在选择最优进程时时间复杂度为O(1)，所以被称为O(1)调度。

这两种调度算法，其基本思路都是通过一系列运行指标确定进程的优先级，然后根据进程的优先级确定调度哪个进程，而CFS则转换了一种思路，它不计算优先级，而是通过计算进程消耗的CPU时间（标准化以后的虚拟CPU时间）来确定谁来调度。从而到达所谓的公平性。

- 绝对公平性：
   cfs定义了一种新的模型，其基本思路很简单，他把CPU当做一种资源，并记录下每一个进程对该资源使用的情况，在调度时，调度器总是选择消耗资源最少的进程来运行。这就是所谓的“完全公平”。但这种绝对的公平有时也是一种不公平，因为有些进程的工作比其他进程更重要，我们希望能按照权重来分配CPU资源。
- 相对公平性：
   为了区别不同优先级的进程，就是会根据各个进程的权重分配运行时间(权重怎么来的后面再说)。进程的运行时间计算公式为:

> 分配给进程的运行时间 = 调度周期 * 进程权重 / 所有进程权重之和   (公式1)
>  调度周期很好理解，就是将所处于TASK_RUNNING态进程都调度一遍的时间。

举个例子来说明一下，比如系统中只两个进程A, B，权重分别为1和2，假设调度周期设为30ms，那么分配给A的CPU时间为:30ms * (1/(1+2)) = 10ms；而B的CPU时间为：30ms * (2/(1+2)) = 20ms。那么在这30ms中A将运行10ms，B将运行20ms。

# 2、实现原理

在实现层面，Linux通过引入virtual runtime(vruntime)来完成上面的设想，具体的,我们来看下从实际运行时间到vruntime的换算公式

> vruntime = 实际运行时间 * 1024 / 进程权重 。 (公式2)

实际上vruntime就是根据权重将实际运行时间标准化，标准化之后，各个进程对资源的消耗情况就可以直接通过比较vruntime来知道，比如某个进程的vruntime比较小，我们就可以知道这个进程消耗CPU资源比较少，反之消耗CPU资源就比较多。

有了vruntime的概念后，调度算法就非常简单了，谁的vruntime值较小就说明它以前占用cpu的时间较短，受到了“不公平”对待，因此下一个运行进程就是它。这样既能公平选择进程，又能保证高优先级进程获得较多的运行时间。这就是CFS的主要思想了。
 或者可以这么理解：CFS的思想就是让每个调度实体（没组调度的情形下就是进程，以后就说进程了)的vruntime互相追赶，而每个调度实体的vruntime增加速度不同，权重越大的增加的越慢，这样就能获得更多的cpu执行时间。

具体实现上，Linux采用了一颗红黑树（对于多核调度，实际上每一个核有一个自己的红黑树），记录下每一个进程的vruntime，需要调度时，从红黑树中选取一个vruntime最小的进程出来运行。

# 3、更多的细节

## 3.1、权重如何决定

权重由nice值确定，具体的，权重跟进程nice值之间有一一对应的关系，可以通过全局数组prio_to_weight来转换，nice值越大，权重越低。



```cpp
nice值共有40个，与权重之间，每一个nice值相差10%左右。
static const int prio_to_weight[40] = {
    /* -20 */ 88761, 71755, 56483, 46273, 36291,
    /* -15 */ 29154, 23254, 18705, 14949, 11916,
    /* -10 */ 9548, 7620, 6100, 4904, 3906,
    /* -5 */ 3121, 2501, 1991, 1586, 1277,
    /* 0 */ 1024, 820, 655, 526, 423,
    /* 5 */ 335, 272, 215, 172, 137,
    /* 10 */ 110, 87, 70, 56, 45,
    /* 15 */ 36, 29, 23, 18, 15,
};
```

## 3.2、新创建进程的vruntime是多少？

假如新进程的vruntime初值为0的话，比老进程的值小很多，那么它在相当长的时间内都会保持抢占CPU的优势，老进程就要饿死了，这显然是不公平的。**CFS是这样做的：**每个CPU的运行队列cfs_rq都维护一个min_vruntime字段，记录该运行队列中所有进程的vruntime最小值，新进程的初始vruntime值就以它所在运行队列的min_vruntime为基础来设置，与老进程保持在合理的差距范围内。

> 新进程的vruntime初值的设置与两个参数有关：
>  sched_child_runs_first：规定fork之后让子进程先于父进程运行;
>  sched_features的START_DEBIT位：规定新进程的第一次运行要有延迟。

> *注：sched_features是控制调度器特性的开关，每个bit表示调度器的一个特性。在sched_features.h文件中记录了全部的特性。START_DEBIT是其中之一，如果打开这个特性，表示给新进程的vruntime初始值要设置得比默认值更大一些，这样会推迟它的运行时间，以防进程通过不停的fork来获得cpu时间片。*

> 如果参数 sched_child_runs_first打开，意味着创建子进程后，保证子进程会在父进程之前运行。

创建子进程的具体流程如下：

1. 子进程在创建时，vruntime初值首先被设置为min_vruntime；
2. 然后，如果sched_features中设置了START_DEBIT位，vruntime会在min_vruntime的基础上再增大一些。
3. 设置完子进程的vruntime之后，检查sched_child_runs_first参数，如果为1的话，就比较父进程和子进程的vruntime，若是父进程的vruntime更小，就对换父、子进程的vruntime，这样就保证了子进程会在父进程之前运行。

## 3.3、休眠进程的vruntime一直保持不变吗？

如果休眠进程的 vruntime 保持不变，而其他运行进程的 vruntime 一直在推进，那么等到休眠进程终于唤醒的时候，它的vruntime比别人小很多，会使它获得长时间抢占CPU的优势，其他进程就要饿死了。这显然是另一种形式的不公平。**CFS是这样做的：**在休眠进程被唤醒时重新设置vruntime值，以min_vruntime值为基础，给予一定的补偿，但不能补偿太多。



```rust
static void
place_entity(struct cfs_rq *cfs_rq, struct sched_entity *se, int initial)
{
        u64 vruntime = cfs_rq->min_vruntime;

        /*
         * The 'current' period is already promised to the current tasks,
         * however the extra weight of the new task will slow them down a
         * little, place the new task so that it fits in the slot that
         * stays open at the end.
         */
        if (initial && sched_feat(START_DEBIT)) /* initial表示新进程 */
                vruntime += sched_vslice(cfs_rq, se);

        /* sleeps up to a single latency don't count. */
        if (!initial) { /* 休眠进程 */
                unsigned long thresh = sysctl_sched_latency; /*一个调度周期 */

                /*
                 * Halve their sleep time's effect, to allow
                 * for a gentler effect of sleepers:
                 */
                if (sched_feat(GENTLE_FAIR_SLEEPERS)) /* 若设了GENTLE_FAIR_SLEEPERS */
                        thresh >>= 1; /* 补偿减为调度周期的一半 */

                vruntime -= thresh;
        }

        /* ensure we never gain time by being placed backwards. */
        vruntime = max_vruntime(se->vruntime, vruntime);

        se->vruntime = vruntime;
}
```

## 3.4、休眠进程在唤醒时会立刻抢占CPU吗？

这是由CFS的唤醒抢占 特性决定的，即sched_features的WAKEUP_PREEMPT位。

由于休眠进程在唤醒时会获得vruntime的补偿，所以它在醒来的时候有能力抢占CPU是大概率事件，这也是CFS调度算法的本意，即保证交互式进程的响应速度，因为交互式进程等待用户输入会频繁休眠。除了交互式进程以外，主动休眠的进程同样也会在唤醒时获得补偿，例如通过调用sleep()、nanosleep()的方式，定时醒来完成特定任务，这类进程往往并不要求快速响应，但是CFS不会把它们与交互式进程区分开来，它们同样也会在每次唤醒时获得vruntime补偿，这有可能会导致其它更重要的应用进程被抢占，有损整体性能。

> 我曾经处理过的一个案例：服务器上有两类应用进程，A进程定时循环检查有没有新任务，如果有的话就简单预处理后通知B进程，然后调用nanosleep()主动休眠，醒来后再重复下一个循环；B进程负责数据运算，是CPU消耗型的；B进程的运行时间很长，而A进程每次运行时间都很短，但睡眠/唤醒却十分频繁，每次唤醒就会抢占B，导致B的运行频繁被打断，大量的进程切换带来很大的开销，整体性能下降很厉害。那有什么办法吗？有，CFS可以禁止唤醒抢占 特性：
>  \# echo NO_WAKEUP_PREEMPT > /sys/kernel/debug/sched_features 1
>  \# echo NO_WAKEUP_PREEMPT > /sys/kernel/debug/sched_features

禁用唤醒抢占 特性之后，刚唤醒的进程不会立即抢占运行中的进程，而是要等到运行进程用完时间片之后。在以上案例中，经过这样的调整之后B进程被抢占的频率大大降低了，整体性能得到了改善。
 如果禁止唤醒抢占特性对你的系统来说太过激进的话，你还可以选择调大以下参数：

> sched_wakeup_granularity_ns
>  这个参数限定了一个唤醒进程要抢占当前进程之前必须满足的条件：只有当该唤醒进程的vruntime比当前进程的vruntime小、并且两者差距(vdiff)大于sched_wakeup_granularity_ns的情况下，才可以抢占，否则不可以。这个参数越大，发生唤醒抢占就越不容易。

## 3.5、进程占用的CPU时间片可以无穷小吗？

假设有两个进程，它们的vruntime初值都是一样的，第一个进程只要一运行，它的vruntime马上就比第二个进程更大了，那么它的CPU会立即被第二个进程抢占吗？**CFS是这样做的：**为了避免过于短暂的进程切换造成太大的消耗，CFS设定了进程占用CPU的最小时间值，sched_min_granularity_ns，正在CPU上运行的进程如果不足这个时间是不可以被调离CPU的。

sched_min_granularity_ns发挥作用的另一个场景是，本文开门见山就讲过，CFS把调度周期sched_latency按照进程的数量平分，给每个进程平均分配CPU时间片（当然要按照nice值加权，为简化起见不再强调），但是如果进程数量太多的话，就会造成CPU时间片太小，如果小于sched_min_granularity_ns的话就以sched_min_granularity_ns为准；而调度周期也随之不再遵守sched_latency_ns，而是以 (sched_min_granularity_ns * 进程数量) 的乘积为准。

## 3.6、进程从一个CPU迁移到另一个CPU上的时候vruntime会不会变？

在多CPU的系统上，不同的CPU的负载不一样，有的CPU更忙一些，而每个CPU都有自己的运行队列，每个队列中的进程的vruntime也走得有快有慢，比如我们对比每个运行队列的min_vruntime值，都会有不同：



```bash
# grep min_vruntime /proc/sched_debug
.min_vruntime : 12403175.972743
.min_vruntime : 14422108.528121

# grep min_vruntime /proc/sched_debug
.min_vruntime : 12403175.972743
.min_vruntime : 14422108.528121
```

如果一个进程从min_vruntime更小的CPU (A) 上迁移到min_vruntime更大的CPU (B) 上，可能就会占便宜了，因为CPU (B) 的运行队列中进程的vruntime普遍比较大，迁移过来的进程就会获得更多的CPU时间片。这显然不太公平。
 **CFS是这样做的：**

> - 当进程从一个CPU的运行队列中出来 (dequeue_entity) 的时候， 它的vruntime要减去队列的min_vruntime值；
> - 而当进程加入另一个CPU的运行队列 ( enqueue_entiry) 时，它的vruntime要加上该队列的min_vruntime值。

这样，进程从一个CPU迁移到另一个CPU之后，vruntime保持相对公平。具体代码如下：



```rust
static void
dequeue_entity(struct cfs_rq *cfs_rq, struct sched_entity *se, int flags)
{
...
        /*
         * Normalize the entity after updating the min_vruntime because the
         * update can refer to the ->curr item and we need to reflect this
         * movement in our normalized position.
         */
        if (!(flags & DEQUEUE_SLEEP))
                se->vruntime -= cfs_rq->min_vruntime;
...
}

static void
enqueue_entity(struct cfs_rq *cfs_rq, struct sched_entity *se, int flags)
{
        /*
         * Update the normalized vruntime before updating min_vruntime
         * through callig update_curr().
         */
        if (!(flags & ENQUEUE_WAKEUP) || (flags & ENQUEUE_WAKING))
                se->vruntime += cfs_rq->min_vruntime;
...
}


static void
dequeue_entity(struct cfs_rq *cfs_rq, struct sched_entity *se, int flags)
{
...
        /*
         * Normalize the entity after updating the min_vruntime because the
         * update can refer to the ->curr item and we need to reflect this
         * movement in our normalized position.
         */
        if (!(flags & DEQUEUE_SLEEP))
                se->vruntime -= cfs_rq->min_vruntime;
...
}
 
static void
enqueue_entity(struct cfs_rq *cfs_rq, struct sched_entity *se, int flags)
{
        /*
         * Update the normalized vruntime before updating min_vruntime
         * through callig update_curr().
         */
        if (!(flags & ENQUEUE_WAKEUP) || (flags & ENQUEUE_WAKING))
                se->vruntime += cfs_rq->min_vruntime;
...
}
```

## 3.7、Vruntime溢出问题

之前说过红黑树中实际的作为key的不是vruntime而是`vruntime - min_vruntime`。min_vruntime是当前红黑树中最小的key。这是为什么呢，我们先看看vruntime 的类型，是usigned long类型的，再看看key的类型，是signed long类型的，因为进程的虚拟时间是一个递增的正值，因此它不会是负 数，但是它有它的上限，就是unsigned long所能表示的最大值，如果溢出了，那么它就会从0开始回滚，如果这样的话，结果会怎样？结果很严重 啊，就是说会本末倒置的，比如以下例子，以unsigned char说明问题：



```cpp
unsigned char a = 251，b = 254;
b += 5;//到此判断a和b的大小看看上面的例子，b回滚了，导致a远远大于b，其实真正的结果应该是b比a大8，怎么做到真正的结果呢？改为以下：
unsigned char a = 251，b = 254;
b += 5;
signed char c = a - 250,d = b - 250;//到此判断c和d的大小
```

结果正确了，要的就是这个效果，可是进程的vruntime怎么用unsigned long类型而不处理溢出问题呢？因为这个vruntime的作用就是 推进虚拟时钟，并没有别的用处，它可以不在乎，然而在计算红黑树的key的时候就不能不在乎了，于是减去一个最小的vruntime将所有进程的key围 绕在最小vruntime的周围，这样更加容易追踪。运行队列的min_vruntime的作用就是处理溢出问题的。



作者：kummerwu
链接：https://www.jianshu.com/p/673c9e4817a8
来源：简书
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。