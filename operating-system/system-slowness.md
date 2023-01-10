## System Slowness

提问: Linux主机 console执行命令响应比较慢; 运行的Pod/服务 提供的REST API response 变慢; 如何排查？

以上是云平台运维中的常见问题, 通常大家都知道要排查: 内存、CPU、IO. 但具体怎么做？
- 需要检查哪些指标, 使用什么工具
- 根据每一步检查的结果, 如何分析定位原因

系统问题排查最常用的工具就是 top, 类似于看看系统的血常规, top结果里通常要看的就是常见三高
- load average
- io wait
- cpu utilization

问题是这三个指标, 多高算高(异常), 什么情况会导致system slowness, 下面我们来一起理解下这三个指标

### load average
- what is load average really means? 参考 [brendan gregg 的blog] [1]
  - 结论: load average 的值就是 R状态 + D状态的 task数量和 [Linux 中的负载高低和 CPU 开销并不完全对应](https://mp.weixin.qq.com/s/1Pl4tT_Nq-fEZrtRpILiig)
  - 过程: brendan 对这个问题也比较懵逼(mystery), 通过翻源码 [kernel/sched/loadavg.c] [2] + POC, 搞清楚了这个问题. 
  - 背景: In 1993, a Linux engineer found a nonintuitive case with load averages, and with a three-line patch changed them forever from "CPU load averages" to what one might call "system load averages."  

- 如何找到 R+D 的task
  - top -H 过滤task状态
    - load 统计的是最小调度单元 所以 -H 是必须的
    - press o - to open the case insensitive filter;
    - on the prompt, type S=R. this will filter by status, with running value [3]
  - ps [[4]] [[7]], D状态比较好用, R状态列不全task
    - ```for x in `seq 1 1 10`; do ps -eo state,pid,cmd | grep "^D"; echo "----"; sleep 5; done```
  - vmstat, 查看Procs的r和b两列, 但看不到Task的 cmd 和 pid
    - r: The number of runnable processes (running or waiting for run time).
    - b: The number of processes in uninterruptible sleep.
  
- 多高算高
  - [kernel/sched/loadavg.c] [2] 源码注释，感觉是作者的免责(耍流氓)声明
    > This file contains the magic bits required to compute the global loadavg figure. Its a silly number but people think its important. We go through great pains to make it work on big machines and tickless kernels.
  - load/cores 比值
    - 我自己的理解, cores应该是 sockets*cores*thread, 也就是lscpu看到的cpu数量, 也是top看到的cpu数量
    - 关于这个值, 没有公认的标准. [brendan gregg 的blog] [1]里提到他自己的邮件系统里 这个比值为2 跑起来也没啥问题
    - 我们自己的环境里, 基本都<0.5
    - [[4]] 有提到0.5-0.7之间 需要调查

### io wait
- what is io wait really means?
  - ```
    man top
    wa, IO-wait : time waiting for I/O completion 
    ```
  - 但是 "CPU never spends clock cycles waiting for an I/O operation to complete." 对呀, 如果Task要等IO, 内核应该做进程切换把CPU时间片分给其他进程, 那CPU io-wait 到底是啥意思呢?
  - [[5]] 里的POC解释的很清楚, io-wait的前提是 CPU是空闲的, CPU空闲的前提下如果有一个被调度到这个CPU的Task在等IO, 则产生io wait值. 下面是POC步骤，可以看到io-wait和 cpu idle的关系
  ```cgo
  taskset 1 dd if=/dev/vda of=/dev/null bs=1MB
  
  top - 20:03:33 up 168 days,  2:13,  3 users,  load average: 0.16, 0.11, 0.14
  Tasks: 102 total,   1 running, 101 sleeping,   0 stopped,   0 zombie
  %Cpu0  :  0.0 us,  1.0 sy,  0.0 ni, 99.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
  %Cpu1  :  0.0 us, 19.6 sy,  0.0 ni,  0.0 id, 80.4 wa,  0.0 hi,  0.0 si,  0.0 st
  %Cpu2  :  0.0 us,  1.0 sy,  0.0 ni, 99.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
  %Cpu3  :  0.0 us,  1.3 sy,  0.0 ni, 98.7 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
  
  taskset 2 sh -c "while true; do true; done"
  
  top - 20:04:12 up 168 days,  2:14,  3 users,  load average: 0.85, 0.29, 0.19
  Tasks: 103 total,   2 running, 101 sleeping,   0 stopped,   0 zombie
  %Cpu0  :  0.0 us,  0.0 sy,  0.0 ni,100.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
  %Cpu1  : 97.0 us,  3.0 sy,  0.0 ni,  0.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
  %Cpu2  :  0.0 us,  0.0 sy,  0.0 ni, 99.7 id,  0.0 wa,  0.0 hi,  0.0 si,  0.3 st
  %Cpu3  :  0.0 us,  0.7 sy,  0.0 ni, 99.3 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
  ```
  - 通过上面的信息, 相信大家都io wait有了更清晰的认知. 容易造成困惑的原因是 cpu不会等IO, 所说的等待IO其实都是针对Task而言, 但io wait又是 cpu utilization的一个指标. 
  - > For a given CPU, the I/O wait time is the time during which that CPU was idle (i.e. didn’t execute any tasks) and there was at least one outstanding disk I/O operation requested by a task scheduled on that CPU (at the time it generated that I/O request).
  - 综上，再强调下重点. CPU空闲是前提，这个前提下如果有一个被调度到这个CPU的Task在等IO完成，就会产生io wait值. 所以出现 io wait 能说明: 1. cpu idle; 2. 有 IO 没完成
  
- pidstat列出的%wait 和 top/mpstat 看到的 %io wait 是一回事吗
  - 这个问题很细节, top和mpstat两个名列列出的都是 %io wait, 也就是上面介绍的 iowait. 但pidstat 列出的是wait 是waiting to run
  ```cgo
  man pidstat
  
  %wait
  Percentage of CPU spent by the task while waiting to run.
  ```
  - pidstat的结果能看到, 有些Pods %wait有值, 但其实和IO是无关的. 可以理解为Runnable 的task等待分配CPU时间片
  ```cgo
  Average:      UID       PID    %usr %system  %guest   %wait    %CPU   CPU  Command
  Average:    42418      1505    1.75    0.00    0.00    0.88    1.75     -  heat-engine
  Average:        0     10498    1.75    0.00    0.00    0.88    1.75     -  hagent-api
  Average:        0     43350    0.00    0.88    0.00    0.88    0.88     -  vhost-43279
  Average:        0     43353    0.00    0.88    0.00    1.75    0.88     -  vhost-43279
  ```
- 如何找到那些进程导致io wait高
  - iotop, iotop显示的 IO> 这一列是Task等待IO的时间/Task整体执行时间的占比, 通过这一列能看出哪些Task在等待IO完成, 这一项是构成io wait的条件(cpu idle + IO operation)之一, 进而也能判断出是这些Task导致io wait高. 上面跑dd的例子, 同时跑iotop能看到dd的进程的 IO>
  ```cgo
  man iotop
  
  the percentage of time the thread/process spent while swapping in and while waiting on I/O.
  ```
  - iostat, 这个是看盘的io情况, 看不出进程, 但盘的IO是进程触发的, 所以能印证环境是是否有IO requests. 
    - iostat 主要关注的列是 %util, 从文档看 %util体现了device忙不忙(是否有IO requests), 但最后的如果是 modern SSDs... 那产品环境里看到的ssd缓存盘 %util>80 应该也没问题
    ```cgo
    man iostat
    
    %util
         Percentage of elapsed time during which I/O requests were issued to the device (bandwidth utilization for  the  device).  Device
         saturation  occurs  when this value is close to 100% for devices serving requests serially.  But for devices serving requests in
         parallel, such as RAID arrays and modern SSDs, this number does not reflect their performance limits.
    ```
    - iostat 能提供更详细的磁盘IO指标, 针对io wait这一项, iostat能列出 ```svctm r_await w_await```
  
- 多高算高
  - io wait升高 只能说明有Task 在做 IO operation
  - 通过iotop 能找到产生IO operation的 Task
  - 通过iostat 能看到磁盘处理 IO的情况
  - 如果以上几个指标都对应的上, 那只能说明当前环境确实有 IO密集的任务在跑, 例如数据ceph rebalance 或者 VM产生的大IO操作, 这种情况不能说环境异常 合理的方式是保证QoS或者限速 来防止共享存储对其他应用的影响
  - 如果进程的状态hang在D状态了, 那要看进程的IO操作对应的设备或者集群是不是出问题了; 如果iostat 看到的device 指标持续飙高那要看是不是 设备出现故障了 或者 设备到瓶颈了

### cpu utilization
- what is cpu utilization really means? [brendan 的 CPU Utilization is Wrong] [8], 然而并没有看懂...下面是 cpu time的分类, 常被忽视的是```ni hi si```
  ```cgo
   man top
  
   us, user    : time running un-niced user processes
   sy, system  : time running kernel processes
   ni, nice    : time running niced user processes
   id, idle    : time spent in the kernel idle handler
   wa, IO-wait : time waiting for I/O completion
   hi : time spent servicing hardware interrupts
   si : time spent servicing software interrupts
   st : time stolen from this vm by the hypervisor
  ```
- cpu utilization vs. cpu usage - No precise answer
- 如何找到那些进程导致cpu utilization高 - top很容易找到
- 多高算高
  - 如上面所说, 单机的cpu使用率问题通过top比较容易找到是哪个进程导致, 后面就是分析进程的原因
  - 这里介绍下数据中心的CPU使用率. 
    - 先八卦一下数据中心 低CPU使用率的 黑历史, [数据中心成本与利用率现状分析] [9] 介绍 Google 5000台在线应用服务器, 2006年平均CPU利用率在30%, 2013年依然是30%
    - [Alibaba 内部私有云 CPU使用率分析] [10], 2017 2018两年的数据分析, 也是30%左右
    > 这意味着，假设 100 万台服务器中有 50 万台利用率只有 30%，那么相当于 5 年 100 亿元人民币的运维成本中有 70 亿元被浪费掉了，只有30亿元真正产生了效益。
    > 30%低吗？而更多的企业达不到这个水平，比如麦肯锡估计整个业界服务器的平均利用率大约是 6%，而Gartner的估计稍乐观一些，认为大概是12%。
  - 为什么这么低
    - CPU运行速度 远大于 内存、外设?
    - CPU是弹性资源, 当前数据中心为了保障用户请求的服务质量，不得不采用过量提供资源的方式，哪怕是牺牲了资源利用率
  - 如何提升CPU使用率, 节约成本
    - CPU超卖
    - 弹性扩缩容  
    - 在/离线任务混合调度, [阿里混部-CPU利用率45%的运行之道] [11]
    > One of Borg’s primary goals is to make efficient use of Google’s fleet of machines, which represents a significant financial investment: increasing utilization by a few percentage points can save millions of dollars.

### 知道了这么多道理依然过不好这一生 -- mystery && ambiguous
- top、pidstat、iotop... 这些都是用户态的统计工具, 都是从/proc/<pid>/下面取值做聚合, 统计的值可能并不是你想的那个意思, 这些工具只能起辅助作用
- 深入了解了上面三个指标之后 会发现, 某个指标高其实说明不了什么问题, 最佳实践的建议都是要综合多个工具检测的结果做交叉验证, 进而定位问题
- 如果定位到某个应用出问题了, 需要perf上场 [[6]], 这里面有一系列使用了```perf```的案例分析, 也有```sar```分析SYN FLOOD 攻击, 推荐阅读

[1]: https://www.brendangregg.com/blog/2017-08-08/linux-load-averages.html
[2]: https://github.com/torvalds/linux/blob/master/kernel/sched/loadavg.c
[3]: https://stackoverflow.com/questions/58342540/how-to-filter-out-only-the-running-processes-using-top-command
[4]: https://cloud.tencent.com/developer/article/1027288
[5]: https://veithen.io/2013/11/18/iowait-linux.html
[6]: https://cloud.tencent.com/developer/article/1678330
[7]: https://tanelpoder.com/posts/high-system-load-low-cpu-utilization-on-linux/
[8]: https://www.brendangregg.com/blog/2017-05-09/cpu-utilization-is-wrong.html
[9]: https://docs.huihoo.com/data-center/%E6%95%B0%E6%8D%AE%E4%B8%AD%E5%BF%83%E6%88%90%E6%9C%AC%E4%B8%8E%E5%88%A9%E7%94%A8%E7%8E%87%E7%8E%B0%E7%8A%B6%E5%88%86%E6%9E%90.pdf
[10]: http://acs.ict.ac.cn/baoyg/downloads/IWQoS2019-AlibabaTraceAnalysis.pdf
[11]: https://developer.aliyun.com/article/651202