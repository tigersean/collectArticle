I have two sets of servers w/128G memory, distinguished by when they were provisioned, that are behaving very differently while running the exact same daemon (elasticsearch). I am using elasticsearch for full text search, not log storage, so this baically a read heavy operation with minimal writes (less than 1MB/s). This daemon mmap's the full dataset of ~350GB into its virtual memory and then accesses certain portions of it to serve requests. These servers have no swap space configured.

The problem is one set of servers behaves well, issues ~50 major faults per second and need on average 10MB/s of disk io to satisfy that demand. The poorly performing servers see 500 major faults per second and need on average ~200MB/s of disk io to satisfy that. The increase in disk io leads to poor p95 response latencies and occasional overloads as it hits the disk limits of ~550MB/s.

They all sit behind the same load balancer, and are part of the same cluster. I could see if one server was behaving badly it might be differences in load, but to have such a clear distinction with 16 servers behaving poorly and 20 servers behaving well, with them being purhased+provisioned at different times, that something at the kernel/configuration level must be causing the issues.

To get to the question, how can i make these poorly behaving servers act like the ones that are well behaved? Where should the debugging efforts be focused?

Below is some data I've collected to look at what the system is doing from the `sar` and `page-types` tools in each of the three states.

Software: - debian jessie - linux 4.9.25 - elasticsearch 5.3.2 - openjdk 1.8.0_141

First some page fault data from a well behaved server (from `sar -B`):

```
07:55:01 PM  pgpgin/s pgpgout/s   fault/s  majflt/s  pgfree/s pgscank/s pgscand/s pgsteal/s    %vmeff
08:05:01 PM   3105.89    811.60   2084.40     48.16   3385.30      0.00      0.00    810.35      0.00
08:15:01 PM   4317.65    665.28    910.29     57.03   3160.14      0.00      0.00   1047.14      0.00
08:25:01 PM   3132.84    648.48    868.41     50.00   2899.14      0.00      0.00    801.27      0.00
08:35:01 PM   2940.02   1026.47   2031.69     45.26   3418.65      0.00      0.00    764.05      0.00
08:45:01 PM   2985.72   1011.27    744.34     45.78   2796.54      0.00      0.00    817.23      0.00
08:55:01 PM   2970.14    588.34    858.08     47.65   2852.33      0.00      0.00    749.17      0.00
09:05:01 PM   3489.23   2364.78   2121.48     47.13   3945.93      0.00      0.00   1145.02      0.00
09:15:01 PM   2695.48    594.57    858.56     44.57   2616.84      0.00      0.00    624.13      0.00
```

And here is one of the poorly performing servers:

```
07:55:01 PM  pgpgin/s pgpgout/s   fault/s  majflt/s  pgfree/s pgscank/s pgscand/s pgsteal/s    %vmeff
08:05:01 PM 268547.64   1223.73   5266.73    503.12  71836.18      0.00      0.00  67170.50      0.00
08:15:01 PM 279865.31    944.04   3995.52    520.17  74231.90      0.00      0.00  70000.23      0.00
08:25:01 PM 265806.75    966.55   3747.43    499.45  70443.49      0.00      0.00  66407.62      0.00
08:35:01 PM 251820.93   1831.04   4689.62    475.43  67445.29      0.00      0.00  63056.35      0.00
08:45:01 PM 236055.04   1022.32   3498.37    449.37  62955.37      0.00      0.00  59042.16      0.00
08:55:01 PM 239478.40    971.98   3523.61    451.76  63856.04      0.00      0.00  59953.38      0.00
09:05:01 PM 232213.81   1058.04   4436.75    437.09  62194.61      0.00      0.00  58100.47      0.00
09:15:01 PM 216433.72    911.94   3192.28    413.23  57737.62      0.00      0.00  54126.78      0.00
```

I suspect this is due to poor performance of the LRU portion of page reclamation. If i run `while true; do echo 1 > /proc/sys/vm/drop_caches; sleep 30; done`, which drops all non-mmaped pages, there will be an initial spike of disk io, but after about 30 minutes it will settle down . I ran this for ~48 hours on all servers to verify it would show the same reduction in IO both under peak daily load and the low points. It did. Sar now reports the following:

```
12:55:01 PM  pgpgin/s pgpgout/s   fault/s  majflt/s  pgfree/s pgscank/s pgscand/s pgsteal/s    %vmeff
01:05:01 PM 121327.14   1482.09   2277.40    140.25  32781.26      0.00      0.00   1764.61      0.00
01:15:01 PM 109911.39   1334.51   1057.51    130.53  31095.68      0.00      0.00   1121.39      0.00
01:25:01 PM 126500.69   1652.51   1216.76    143.07  35830.38      0.00      0.00   2801.84      0.00
01:35:01 PM 132669.45   1857.62   2309.86    148.47  36735.79      0.00      0.00   3181.19      0.00
01:45:01 PM 126872.04   1451.94   1062.94    145.68  34678.26      0.00      0.00    992.60      0.00
01:55:01 PM 121002.21   1818.32   1142.16    139.40  34168.53      0.00      0.00   1640.18      0.00
02:05:01 PM 121824.18   1260.22   2319.56    142.80  33254.67      0.00      0.00   1738.25      0.00
02:15:02 PM 120768.12   1100.87   1143.36    140.20  34195.15      0.00      0.00   1702.83      0.00
```

Major page faults have been cut to 1/3 the prior value. Pages brought in from disk have cut in half. This reduces disk IO from ~200MB/s to ~100MB/s, but the well behaved server is still outperforming them all significantly with only 50 major faults/s, and only needing to do ~10MB/s of disk io.

To get a look at what the LRU algorithm has to work I use the page-types tool from the kernel. This is a well behaved server (from `page-types | awk '$3 > 1000 { print $0 }' | sort -nk3`):

```
             flags      page-count       MB  symbolic-flags                     long-symbolic-flags
0x0000000000000828          257715     1006  ___U_l_____M______________________________ uptodate,lru,mmap
0x0000000000000080          259789     1014  _______S__________________________________ slab
0x000000000000006c          279344     1091  __RU_lA___________________________________ referenced,uptodate,lru,active
0x0000000000000268          305744     1194  ___U_lA__I________________________________ uptodate,lru,active,reclaim
0x0000000000100000          524288     2048  ____________________n_____________________ nopage
0x000000000000082c          632704     2471  __RU_l_____M______________________________ referenced,uptodate,lru,mmap
0x0000000000000000          763312     2981  __________________________________________
0x0000000000000068         2108618     8236  ___U_lA___________________________________ uptodate,lru,active
0x000000000000086c         6987343    27294  __RU_lA____M______________________________ referenced,uptodate,lru,active,mmap
0x0000000000005868         8589411    33552  ___U_lA____Ma_b___________________________ uptodate,lru,active,mmap,anonymous,swapbacked
0x0000000000000868        12513737    48881  ___U_lA____M______________________________ uptodate,lru,active,mmap
             total        34078720   133120
```

This is a poorly performing server:

```
             flags      page-count       MB  symbolic-flags                     long-symbolic-flags
0x0000000000100000          262144     1024  ____________________n_____________________ nopage
0x0000000000000828          340276     1329  ___U_l_____M______________________________ uptodate,lru,mmap
0x000000000000086c          515691     2014  __RU_lA____M______________________________ referenced,uptodate,lru,active,mmap
0x0000000000000028          687263     2684  ___U_l____________________________________ uptodate,lru
0x0000000000000000          785662     3068  __________________________________________
0x0000000000000868         7946840    31042  ___U_lA____M______________________________ uptodate,lru,active,mmap
0x0000000000005868         8588629    33549  ___U_lA____Ma_b___________________________ uptodate,lru,active,mmap,anonymous,swapbacked
0x0000000000000068        14133541    55209  ___U_lA___________________________________ uptodate,lru,active
             total        33816576   132096
```

And here's what it looks like when looping the drop_caches command:

```
             flags      page-count       MB  symbolic-flags                     long-symbolic-flags
0x0000000000100000          262144     1024  ____________________n_____________________ nopage
0x0000000000000400          394790     1542  __________B_______________________________ buddy
0x0000000000000000          761557     2974  __________________________________________
0x0000000000000868         1451890     5671  ___U_lA____M______________________________ uptodate,lru,active,mmap
0x000000000000082c         3123142    12199  __RU_l_____M______________________________ referenced,uptodate,lru,mmap
0x0000000000000828         5278755    20620  ___U_l_____M______________________________ uptodate,lru,mmap
0x0000000000005868         8622864    33683  ___U_lA____Ma_b___________________________ uptodate,lru,active,mmap,anonymous,swapbacked
0x000000000000086c        13630124    53242  __RU_lA____M______________________________ referenced,uptodate,lru,active,mmap
             total        33816576   132096
```

Things tried:

- increase /proc/sys/vm/vfs_cache_pressure to various values between 150 and 10000. This makes no difference in IO or the data reported by `page-types`, which makes sense because this balances kernel structures vs user pages, and my problem is with different user pages
- increase /proc/sys/vm/swappiness. Didn't expect this to do anything, and it doesn't, but didn't hurt to check.
- Disable mmap (instead using java's nio which is based on epoll). This immediatly making the server's IO usage look just like the well behaved ones in terms of IO usage. The downside here is that system cpu % is tied to how much IO is happening, with 10MB/s taking ~1.5% and the occasional load up to ~100MB/s of disk io saw system cpu % of 5 to 10%. On a 32 core server that's 1.5 to 3 cpus used entirely to handle epoll. Latency was also worse with nio (vs well behaved mmap'd servers). This is a plausible solution but its a cop-out, without understanding what actually is going wrong.
- countless hours with the `perf` tool poking around at stack traces, flame graphs, and looking for differences in kernel behaviour. Little if any insight gained.
- Checked disk read-ahead settings are the same across servers. raid0 on poorly performing servers defaulted to 2048 blocks, well performing servers raid0 defaulted to 256 block. Updating poorly performing servers to 256 with `blockdev --setra` shows no effect on IO behavior.
- Start the jvm with `numactl --interleave=all` to ensure the problem isn't related to balancing between the two numa nodes. Made no difference.
- verified with `vmtouch`, which basically uses `mincore(2)` to ask the kernel if files are cached, that 99%+ of the buffered memory is being used for the filesystem elasticsearch stores it's data on. This is true in all 3 cases from above.
- verified with `fuser -m` that elasticsearch is the only process using files on the filesystem elasticsearch stores it's data on.

To test soon:

- I'll be re-provisioning one of the misbehaving servers next week, but I'm not optimistic this will have much effect. During this provisioning i'm also updating the raid array to put LVM in front of it. Not expecting anything different from LVM but removing a variable seems worthwhile.



After 2 months

This ended up being an overly aggressive disk read-ahead. The well performing servers had a read-ahead of 128 kB. The poorly performing servers had a read-ahead of 2 mB. Wasn't able to track down why exactly the read-ahead was different, but the most likely cause was that the new machines had LVM in front of the software raid and the older machines were talking to the software raid directly.

While my question indicates I did originally check the read-ahead, notice the difference, and update it on one of the servers, the interaction between mmap and read-ahead is not that straight forward. Specifically when a file is mmap'd the linux kernel will copy the read ahead settings from the disk to a structure about that mmap'd file. Updating the read-ahead settings will not update that structure. Because of this the updated read-ahead does not take effect until after the daemon holding the files open has been restarted.

Reducing the read-ahead and restarting the daemons on the poorly performing servers immediately brought the disk read inline with the well performing ones and reduced our IO wait and tail latency immediately.