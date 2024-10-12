# MonitorLinux
CPU状态，系统负载，软中断，mem，net的监控
工厂模式，重写纯虚函数类实现多态，抽象出一个总的监控系统基类，这样可以规范实现类的实现
通过每三秒做差值，计算时间内的信息，便于看出增量，内存信息是只改变了数据单位

## 目的
在开发过程中会遇到某些性能的问题，这时候就需要我们去分析，所以需要进行性能监控，进而去实现性能优化，和Linux的top一样每三秒进行一次刷新  
Linux的运行信息都存储在一个伪文件系统  /proc 中  

## CPU负载
平均负载也就是单位时间内的进程数，存储在 /proc/loadavg/ 中，反应了系统的整体状态信息，有1、5、15分钟内的平均负载  
可以通过命令 cat /proc/cpuinfo/ 看本机有几个CPU，最理想的状态为平均负载等于CPU的数量  
**平均负载和CPU的使用率无直接关系**  
像在io等待，平均负载增高，但是CPU使用率较低；CPU密集型的情况下，都高。  

## CPU stat
提供了CPU和任务的统计信息，存储在 /proc/stat/中  
user：用户态cpu时间  
nice：优先级用户态cpu时间  
system：内核态cpu时间
idle：空闲时间cpu时间
irq：硬中断cpu时间
softirq：软中断cpu时间
steal：被其他虚拟机占用cpu时间
iowait：等待io的cpu时间

## 中断
异步事件的处理机制，提高系统的并发能力  
会打断其他正常程序，如果终端运行时间太长，可能会丢失其他中断
/proc/softirqs/ ： 记录自开机以来软中断积累次数  
/proc/interruts/ ： 记录自开机以来中断累计次数  
实现的时候需要进行横竖的数据转换，使用了二维数组的实现，因为记录的数据都比较大，所以使用了long long类型  
**Linux分为上、下半部分**  
上半部分：快速处理，处理硬件请求，网卡的数据存到内存中，更新寄存器，在中断模式下运行  
下半部分：延迟处理，由内核触发，在内核线程下运行，有个RCU锁（read、cpoy、update）  


## mem info
由free命令  
total：总内存大小  
used：已使用内存  
free：未使用内存  
shared：共享内存  
buffer/cache：缓存和缓冲区  可读可写  
available：可用内存，包括了缓存，所以比free大  
**buff是磁盘数据的缓存**  
**cache是文件的缓存**  
实现的时候具体数据不进行换算，但是把单位换了一下，kb转成了GB，但是总体位数还是没变

## net
网络流入流出信息  
bytes、packages、errs、drop  

## stress压测
压测工具   指令stress <options>  
模拟特别吃资源的环境  
stress -c n   跑满几个cpu，每个进程都反复不停的计算随机数平方根  

