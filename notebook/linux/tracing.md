---
layout: post
title: Tracing
---


## TODO
- tracepoints
- perf
- SystemTap
- KernelShark



## ftrace
### Check if ftrace is enabled
```
$ cat /proc/sys/kernel/ftrace_enabled 
1

$ sudo sysctl -a 2>&1 | grep ftrace_enabled
kernel.ftrace_enabled = 1
```


### tracefs
- The API interface with ftrace is located in the tracefs file systems.
- As a result, ftrace requires no specialized userspace uitilities to operate.
- Typically, that is mounted at `/sys/kernel/debug/tracing`.
- You can check if the tracefs is mounted as follows:
```
# mount | grep tracefs
tracefs on /sys/kernel/debug/tracing type tracefs (rw,relatime)
```
- If trace has not been mounted yet, you can mount it as follows:
```
# mount -t tracefs nodev /sys/kernel/debug/tracing
```
- tracefs directory
	- `current_tracer`: sets or displays the current trace that is configured.
	- `available_tracer`: displays a list of tracer names that can be configured by echoing their name into `current_tracer`.
	- `tracing_on`: sets or displays whether writing to the trace ring buffer is enabled. (0: disabled, 1: enabled)
	- `trace`: holds the ouput of the trace in a human readable format.
	- `trace_pipe`: is same as the `trace` file, but this is meant to be streamed with live tracing.
	- `trace_marker`: is useful for synchronizing user space with events happending in the kernel. Writing strings into this file will be written into the ftrace buffer.
	- `available_filter_functions`: lists the functions that ftrace can trace.
	- `set_ftrace_filter`: filters functions to be tracked.
	- `buffer_size_kb`: the number of kilobytes each CPU buffer holds.
	- `buffer_total_size_kb`: the total combined size of all the trace buffers.
	- `stack_trace`: the stack back trace of the largest stack that was encountered since the stack tracer is activated.
	- `stack_max_size`: the maximum stack size that was encountered since the stack tracer is activated.


### Tracers
#### Start and Stop Tracing
To stop tracing
```
[~]# cd /sys/kernel/debug/tracing
[tracing]# echo 0 > tracing_on
```

To start tracing
```
[~]# cd /sys/kernel/debug/tracing
[tracing]# echo 1 > tracing_on
```

Common pattern
```
[~]# cd /sys/kernel/debug/tracing
[tracing]# echo 0 > tracing_on
[tracing]# echo function_graph > current_tracer
[tracing]# echo 1 > tracing_on; run_test; echo 0 > tracing_on
```

#### Function Tracer
- The function tracer uses the [`-pg` option of `gcc`](https://sourceware.org/binutils/docs/gprof/Implementation.html) to have every function in the kernel call a special function `mcount()`.
- `CONFIG_DYNAMIC_FTRACE`
	- When `CONFIG_DYNAMIC_FTRACE` is configured the call is converted to a NOP at boot time to keep the system running at 100% performance.
		- During compilation the `mcount()` call-sites are recorded.
		- This list is used at boot time to convert those sites no NOPs.
		- The list is saved to convert the call-sites back into trace calls when the function (or function graph) tracer is eanbled.
		- It is highly recommended to enable `CONFIG_DYNAMIC_FTRACE` because of this performance enhancement.
	- `CONFIG_DYNAMIC_FTRACE` gives the ability to filter which function should be traced.

```
[~]# cd /sys/kernel/debug/tracing

[tracing]# echo function > current_tracer

[tracing]# head -n 20 trace
# tracer: function
#
# entries-in-buffer/entries-written: 102586/278326   #P:2
#
#                                _-----=> irqs-off
#                               / _----=> need-resched
#                              | / _---=> hardirq/softirq
#                              || / _--=> preempt-depth
#                              ||| /     delay
#           TASK-PID     CPU#  ||||   TIMESTAMP  FUNCTION
#              | |         |   ||||      |         |
          <idle>-0       [001] d... 112465.334367: nr_iowait_cpu <-menu_select
          <idle>-0       [001] d... 112465.334367: get_typical_interval <-menu_select
          <idle>-0       [001] d... 112465.334367: tick_nohz_tick_stopped <-menu_select
          <idle>-0       [001] d... 112465.334367: tick_nohz_tick_stopped <-menu_select
          <idle>-0       [001] d... 112465.334367: tick_nohz_tick_stopped <-cpuidle_idle_call
          <idle>-0       [001] d... 112465.334367: tick_nohz_idle_retain_tick <-cpuidle_idle_call
          <idle>-0       [001] d... 112465.334368: timer_clear_idle <-cpuidle_idle_call
          <idle>-0       [001] d... 112465.334368: cpuidle_enter <-cpuidle_idle_call
          <idle>-0       [001] d... 112465.334368: tick_nohz_get_next_hrtimer <-cpuidle_enter

[tracing]# echo nop > current_tracer
```

#### Function Graph
- The function tracer shows the flow of functions nicely.
- It traces both the entry and exit of a function, which gives the tracer the ability to know the depth of functions that are called.
- It can make following the flow of execution within the kernel much easier to follow with the human eye.
	- It gives the start and end of a function denoted with the C like annotation of `{` to start a function and `}` at the end.
	- Leaf functions, which do not call other functions, simply end with a `;`.
	- The DURATION column shows the time spent in the corresponding function.
		- These numbers only appear with the leaf functions and the `}` symbol.
		- Note that this time also includes the overhead of all functions within a nested function as well as the overhead of the function graph tracer itself.
			- The function graph tracer hijacks the return address of the function in order to insert a trace callback for the function exit.
			- This breaks the CPU's branch prediction and cause a bit more overhead than the function tracer.
		- `+` is shown when the duration is greater than 10 ms. `!` is shown when the duration is greater than 100 ms.
```
[~]# cd /sys/kernel/debug/tracing
[tracing]# echo function_graph > current_tracer

[tracing]# head -n 20 trace
# tracer: function_graph
#
# CPU  DURATION                  FUNCTION CALLS
# |     |   |                     |   |   |   |
 0)               |                                              sch_direct_xmit() {
 0)               |                                                validate_xmit_skb_list() {
 0)               |                                                  validate_xmit_skb() {
 0)               |                                                    netif_skb_features() {
 0)               |                                                      harmonize_features() {
 0)   0.161 us    |                                                        skb_network_protocol();
 0)   0.495 us    |                                                      }
 0)   0.885 us    |                                                    }
 0)   0.157 us    |                                                    skb_csum_hwoffload_help();
 0)   0.157 us    |                                                    validate_xmit_xfrm();
 0)   2.010 us    |                                                  }
 0)   2.345 us    |                                                }
 0)   0.150 us    |                                                _raw_spin_lock();
 0)               |                                                dev_hard_start_xmit() {
 0)               |                                                  xmit_one.constprop.0() {
 0)               |                                                    dev_queue_xmit_nit() {

[tracing]# echo nop > current_tracer
```

#### Stack Tracer
```
[~]# cd /sys/kernel/debug/tracing/

[tracing]# echo 1 > /proc/sys/kernel/stack_tracer_enabled

[tracing]# cat stack_trace
        Depth    Size   Location    (36 entries)
        -----    ----   --------
  0)     3688      48   _raw_spin_lock_irqsave+0x5/0x30
  1)     3640      24   down_trylock+0xf/0x30
  2)     3616      24   xfs_buf_trylock+0x17/0xa0
  3)     3592      96   xfs_buf_find+0x161/0x360
  4)     3496      80   xfs_buf_get_map+0x44/0x270
  5)     3416      96   xfs_buf_read_map+0x54/0x280
  6)     3320       8   xfs_trans_read_buf_map+0x12d/0x2c0
  7)     3312     184   xfs_btree_read_buf_block.constprop.0+0x95/0xd0
  8)     3128      72   xfs_btree_lookup_get_block+0x96/0x170
  9)     3056     152   xfs_btree_lookup+0xd2/0x4b0
 10)     2904      24   xfs_alloc_lookup_eq+0x1d/0x30
 11)     2880     104   xfs_alloc_fixup_trees+0x1d3/0x3a0
 12)     2776      40   xfs_alloc_cur_finish+0x2b/0x90
 13)     2736     144   xfs_alloc_ag_vextent_near+0x2ed/0x3b0
 14)     2592      16   xfs_alloc_ag_vextent+0x134/0x140
 15)     2576      64   xfs_alloc_vextent+0x325/0x460
 16)     2512     232   xfs_bmap_btalloc+0x45b/0x830
 17)     2280      72   xfs_bmapi_allocate+0xd2/0x2d0
 18)     2208     280   xfs_bmapi_convert_delalloc+0x230/0x470
 19)     1928     168   xfs_map_blocks+0x1b0/0x400
 20)     1760     112   iomap_writepage_map+0x173/0x440
 21)     1648     240   write_cache_pages+0x186/0x3d0
 22)     1408      16   iomap_writepages+0x1c/0x40
 23)     1392     144   xfs_vm_writepages+0x64/0x90
 24)     1248      80   do_writepages+0x34/0xc0
 25)     1168      56   __writeback_single_inode+0x39/0x200
 26)     1112     208   writeback_sb_inodes+0x200/0x470
 27)      904      64   __writeback_inodes_wb+0x4c/0xe0
 28)      840     120   wb_writeback+0x1d8/0x290
 29)      720      88   wb_check_old_data_flush+0xb2/0xc0
 30)      632     120   wb_do_writeback+0xc1/0x170
 31)      512     152   wb_workfn+0x6e/0x240
 32)      360      64   process_one_work+0x1b0/0x350
 33)      296      56   worker_thread+0x49/0x300
 34)      240      64   kthread+0x11b/0x140
 35)      176     176   ret_from_fork+0x22/0x30
```

#### Filter Functions
To find functions to trace
```
[~]# cd /sys/kernel/debug/tracing/

[tracing]# grep kmalloc available_filter_functions
debug_kmalloc
mempool_kmalloc
__traceiter_kmalloc
__traceiter_kmalloc_node
kmalloc_slab
kmalloc_order
kmalloc_order_trace
kmalloc_fix_flags
kmalloc_large_node
__kmalloc
__kmalloc_node_track_caller
__kmalloc_node
__kmalloc_track_caller
devm_kmalloc_match
devm_kmalloc_release
devm_kmalloc
sock_kmalloc
```

To trace specific functions only
```
[~]# cd /sys/kernel/debug/tracing/

[tracing]# echo "xfs_*" > set_ftrace_filter

[tracing]# echo function_graph > current_tracer

[tracing]# head -n 20 trace
# tracer: function_graph
#
# CPU  DURATION                  FUNCTION CALLS
# |     |   |                     |   |   |   |
 0)               |  xfs_trans_ail_cursor_first() {
 0)   0.365 us    |    xfs_trans_ail_cursor_init();
 0)   3.436 us    |  }
 0)               |  xfs_buf_delwri_submit_nowait() {
 0)   0.926 us    |    xfs_buf_delwri_submit_buffers();
 0)   1.390 us    |  }
 0)               |  xfs_trans_ail_cursor_first() {
 0)   0.651 us    |    xfs_trans_ail_cursor_init();
 0)   4.006 us    |  }
 0)               |  xfs_buf_delwri_submit_nowait() {
 0)   0.609 us    |    xfs_buf_delwri_submit_buffers();
 0)   1.075 us    |  }
 0)               |  xfs_trans_ail_cursor_first() {
 0)   0.207 us    |    xfs_trans_ail_cursor_init();
 0)   2.879 us    |  }
 0)               |  xfs_buf_delwri_submit_nowait() {

[tracing]# echo nop > current_tracer
```

#### Trace Specific Process
To trace specific PID
```
[~]# cd /sys/kernel/debug/tracing/

[tracing]# echo $pid > set_ftrace_pid
```


### `trace_printk()`
- Why not `printk()`?
	- `printk()` might lead to a problem when you are debugging a high volume area such as the timer interrupt, the scheduler, or the network.
	- It is also quite common to see a bug "disappear" when adding a few `printk()`s. This is due to the sheer overhead that `printk()` introduces.
- `trace_printk()`
	- `trace_printk()` can be used just like `printk()` and can also be used in any context (interrupt code, NMI code, and scheduler code).
	- `trace_printk()` doesn't output to the console. Instead it writes to the ftrace ring buffer.
		- Writing into the ring buffer with `trace_printk()` only takes around a tenth of a microsecond or so.
		- Using `printk()`, especially when writing to the serial console, may take several milliseconds per write.


### `ftrace_dump_on_oops`
- `ftrace_dump_on_oops` enables ftrace to dump to the onsole the entire trace buffer in ASCII format on oops or panic.
- To enable temporarily
```
# echo 1 > /proc/sys/kernel/ftrace_dump_on_oops
# sysctl kernel.ftrace_dump_on_oops=1
```
- To enable permanently
```
# echo kernel.ftrace_dump_on_oops=1 >> /etc/sysctl.conf
```


### Links
- [ftrace - Wikipedia](https://en.wikipedia.org/wiki/Ftrace)
- [https://www.kernel.org/doc/Documentation/trace/ftrace.txt](https://www.kernel.org/doc/Documentation/trace/ftrace.txt)
- [Debugging the kernel using Ftrace - part 1 [LWN.net]](https://lwn.net/Articles/365835/)
- [Debugging the kernel using Ftrace - part 2 [LWN.net]](https://lwn.net/Articles/366796/)




## trace-cmd
- trace-cmd is a utility that interacts with ftrace.


### Usage
#### trace-cmd list
To list available events, tracers and options
```
# trace-cmd list
events:
...

tracers:
blk function_graph wakeup_dl wakeup_rt wakeup function nop

options:
...
```

To list subsystems where events are available.
```
# trace-cmd list -e | cut -d ':' -f 1 | sort | uniq
alarmtimer
...
xen
xfs
```

To list available events for the specific subsystem.
```
# trace-cmd list -e net
sock:inet_sock_set_state
...
net:net_dev_xmit
net:net_dev_start_xmit
```

To list available functions to filter on.
```
# trace-cmd list -f
...
rpc_unregister_sysctl [sunrpc]
```


### trace-cmd record
To record a live trace and write a trace.dat file.
```
# trace-cmd record -p function_graph
  plugin 'function_graph'
Hit Ctrl^C to stop recording
^CCPU 0: 809024 events lost
CPU 1: 4203724 events lost
CPU 2: 58042 events lost
CPU 3: 3128402 events lost
CPU0 data recorded at offset=0x4d1000
    159522816 bytes in size
CPU1 data recorded at offset=0x9cf3000
    167211008 bytes in size
CPU2 data recorded at offset=0x13c6a000
    160903168 bytes in size
CPU3 data recorded at offset=0x1d5dd000
    167301120 bytes in size
```

To record for the specific command.
```
# trace-cmd record -p function_graph -F <command>
```

To record for the specific process.
```
# trace-cmd record -p function_graph -P <pid>
```

To recrod for the specific function.
```
# trace-cmd record -p function_graph -l <function>
```


### trace-cmd start
To start the tracing without recording to a trace.dat file.
```
# trace-cmd start -p function_graph
  plugin 'function_graph'
```


### trace-cmd stop
To stop tracing
```
# trace-cmd stop
```


### trace-cmd extract
To extract the data from the kernel buffer and create a trace.dat file.
```
# trace-cmd extract
```


### trace-cmd report
To read a trace.dat file and convert the binary data to a ASCII text readable format.
```
# trace-cmd report
```


### trace-cmd stat
To check tracing status
```
# trace-cmd stat

Events:
 All disabled

Buffer size in kilobytes (per cpu):
   1408

Buffer total size in kilobytes:
   5632

Tracing is disabled
```


### trace-cmd reset
To disable all tracing and give back the system performance (clear all data from the kernel buffer).
```
# trace-cmd reset
```



### Links
- [trace-cmd(1) - Linux manual page](https://man7.org/linux/man-pages/man1/trace-cmd.1.html)