---
title: Memory Leak
date: 2017-07-20 10:14:32
tags:
---
# Rails 内存泄露问题定位及解决

> At some point in the life of every Rails developer you are bound to hit a memory leak. It may be tiny amount of constant memory growth, or a spurt of growth that hits you on the job queue when certain jobs run.[4]


## Problem Description
Unicorn + Rails 程序运行一段时间后，服务器内存（4核8G，仅运行Rails服务）发生泄露，其表现为Rails内存占用缓慢爬升，系统free内存逐渐减少，直至各种方法大量报错崩溃。

`Errno::ENOMEM: Cannot allocate memory`

## Problem Analysis
### 如何监控内存泄露

监控CPU是很容易的事情，仅需对production.log进行分析，统计出执行时间最长的请求。但要监控Rails进程的内存泄漏，却非常困难，甚至你找不到任何ruby API可以用来直接查询到进程使用的物理内存。实际上，要获取一个进程的物理内存是一个平台相关的操作，每个操作系统都会自己特定的API，并不通用。 [1]

- VmRSS process memory size
本方法的核心思想是通过监控 Linux系统 /proc/进程pid/status 文件中的VmRSS－－该进程使用的常驻物理内存（Resident Set Size） 来获取内存信息（如下所示）。可参考`pidstat`命令。

```
Name:	unicorn_rails
State:	S (sleeping)
Tgid:	6860
Ngid:	0
Pid:	6860
...
VmRSS:	  169232 kB ### <- Here it is.
VmData:	  745964 kB
VmStk:	    8196 kB
VmExe:	    2900 kB
VmLib:	   19748 kB
VmPTE:	     668 kB
VmSwap:	       0 kB
Threads:	10
...
```
注意第16行，VmRSS记录了该进程使用的常驻物理内存（Resident Set Size）。这就是该进程的实际内存占用了。Therefore，我们的任务转化为记录每次请求前后的VmRSS值变化，便可定位引起内存升高的具体API。这通过Rails的around_filter（其它框架同理）和几行简单的代码即可实现：
```
around_filter :record_memory

def record_memory  
    process_status = File.open("/proc/#{Process.pid}/status")  
    16.times { process_status.gets }   
    rss_before_action = process_status.gets.split[1].to_i  
    process_status.close  
    yield  
    process_status = File.open("/proc/#{Process.pid}/status")  
    16.times { process_status.gets }   
    rss_after_action = process_status.gets.split[1].to_i  
    process_status.close  
    filename  = File.join(Rails.root, 'log', "performance.log")
    logger = Logger.new(filename)
    logger.info("CONSUME MEMORY: #{rss_after_action - rss_before_action}KB\tNow: #{rss_after_action} KB\t#{request.url}")  
  end
```

其中日志的格式把需要统计的数值（消耗的常驻物理内存、当前常驻物理内存）放在前面，方便后续的统计。在这里我通过Logger [3] 把这个日志信息单独记录在"performance.log”这个文件中，并使用logrotate工具按天进行轮询，方便后续分析。
```
/my_rails_app/log/performance.log {                                                                                                                                                                                                                                       
        daily
        missingok
        dateext
        copytruncate
        rotate 365 
        notifempty
}
```
轮询后就是这样：
```
...
-rw-r--r--  1 root root  54M Jul 27 06:25 performance.log-20170727
-rw-r--r--  1 root root  53M Jul 28 06:25 performance.log-20170728
-rw-r--r--  1 root root  44M Jul 29 06:25 performance.log-20170729
-rw-r--r--  1 root root  35M Jul 30 06:25 performance.log-20170730
-rw-r--r--  1 root root  29M Jul 31 06:25 performance.log-20170731
...
```

而统计这个日志变得非常简单，只需要一行shell（如果文件太大请注意系统资源消耗，使用tail、分片抑或是在其它非production environment机器进行）：
`grep "CONSUME MEMORY" performance.log | grep -v "CONSUME MEMORY: 0" | awk '{print $9 "\t"  $11 $12 "\t" $13}' | sort -r -n |head -n 100`
这里使用了简单的`awk`进行格式化输出，可根据自己的日志格式进行调整。再通过`sort`命令统计出最占用内存的前100个请求（-r, —reverse: reverse the result of comparisons; -n,  --numeric-sort: compare according to string numerical value）。结果示例如下：
```
### 增加数 当前内存占用    请求URL 
16528KB	211704KB	https://your_url1
15044KB	221160KB	https://your_url2
14168KB	229456KB	https://your_url3
11868KB	196868KB	https://your_url4
11340KB	199348KB	https://your_url5
...
```
第一行表示请求”https://your_url1” 完成后，VmRSS升高了16528KB，请求完成后当前pid进程占用内存211704KB。（我的Unicorn Rails App是多进程启动，在这里不讨论多进程的问题）。

你会发现Top 100 中有许多相同的请求，这些往往涉及大量数据操作。例如，我将一个Top 5内存占用的请求进行了缓存，使这个请求后续内存占用大幅降低。
优化前：
`CONSUME MEMORY: 79560KB	Now: 758640KB	https://my_target_api`
缓存后：
`CONSUME MEMORY: 0KB	Now: 210952 KB	https://my_target_api`
感谢Robbin[1]分享的这个方法，行之有效。在一定程度上缓解了内存压力。

## 罪魁祸首－－ GC.disable
后来有一天，在研究Unicorn Rails 的重启机制时发现了当前Unicorn 的配置文件，其中`after_fork`中有一行`GC.disable`立马引起了我的关注：
```
before_fork do |server, worker|
  old_pid = "#{Rails.root}/tmp/pids/unicorn.pid.oldbin"
  if File.exists?(old_pid) && server.pid != old_pid
    begin
      Process.kill("QUIT", File.read(old_pid).to_i)
    rescue Errno::ENOENT, Errno::ESRCH
      puts "Send 'QUIT' signal to unicorn error!"
    end 
  end 
end

after_fork do |server, worker|
  # 禁止GC，配合后续的OOB，来减少请求的执行时间
  GC.disable ### <- Disabled??? WTF?
  # the following is *required* for Rails + "preload_app true",
  defined?(ActiveRecord::Base) and 
    ActiveRecord::Base.establish_connection
end
```

这是公司另一个sb不知从哪儿copy的一段，

> You can't run ruby without GC in production. It will *rapidly run out of memory and crash*. You can, however, apply optimization, produce less garbage and get your code be even faster than original benchmark with GC turned off! [2]

> Unicorn uses forked processes to achieve concurrency. Since forked processes are essentially copies of each other, this means that the Rails application need not be thread safe.[6]

- How many memory does one process take? (safely budget the amount of workers)
- send it a QUIT signal. This tells the worker to die after finishing the current request. 
---
## References
[1] [监视Rails进程内存泄漏的技巧 - robbin的自言自语 - ITeye博客](http://robbin.iteye.com/blog/307271) 
[2] [Garbage Collection is Why Ruby on Rails is Slow: Patches to Improve Performance 5x; Memory Profiling - Aconite Blog](http://www.acunote.com/blog/2008/01/garbage-collection-is-why-ruby-is-slow.html) 
[3] [Logger详解](http://starzhou.com/blogs/logger_in_rails) 
[4] [Debugging memory leaks in Ruby](https://samsaffron.com/archive/2015/03/31/debugging-memory-leaks-in-ruby)
[5] [Ruby memory leak detection · Ruby China](https://ruby-china.org/topics/9737)
[6] [How To Optimise Unicorn Workers in a Ruby on Rails App | DigitalOcean](https://www.digitalocean.com/community/tutorials/how-to-optimize-unicorn-workers-in-a-ruby-on-rails-app) 
