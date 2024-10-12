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

## QT
就当作了一个插件工具用的，因为要显示数据    
初始化qtapplication，管理GUI程序的控制和设置，要线程分离（detach），不能阻塞，不然老是报错，再用exec显示（内部其实就是个循环）  
QWidget：实现窗口类，构造窗口，调用app展示。先搞局部组合，再把局部组合成整体  
QTableView：表格控制器，和QstandardItermModel配用，形成表格加数据  
Qlabel：文本赋值  
Qgridlayout：进行布局  
Qstackedlayout：一次展示一个界面  
QpushButton：按钮（解决界面如何切换）  
QObject：用内部信号机制，编译的时候记得在CMake中添加AUTOMOC，不然编不了  
QAbstractTableModel：实现这个类的相关借口，用来做数据模型，具体是调用线程更新模型数据（先清空再添加），再将数据类型转换为QT的类型（枚举一个个实现的），在存储到二维数组中，执行endreset，就更新了数据  

## GRPC
用来进行远程通信的，远程方法像本地方法一样调用  
基于http2协议设计，是对tcp进行封装，不过优化了粘包的问题，通过protobuf传输  
**客户端**创建接连到远程服务器channel，构建使用该channel的Stub，通过Stub调用服务方法，执行RPC  
channel用于执行RPC请求的端点连接，基于负载和配置  
使用**constexpr**：将经过预处理的程序转化为特定的汇编代码，这样可以减少代码的消耗  

## Protobuf  
实现不同机器上的数据交互，序列化协议属于tcp/ip协议的应用层  
**底层原理：**  
1.IDL文件：约定通信的数据结构  
2.IDL compile：编译器，编译IDL文件，生成相应库和代码  
3.stub/skeleton lib：负责序列化和反序列化的代码，其中stub放在客户端，接受应用层参数，序列化后通过底层协议发送到服务端，skeleton相反  
4.client/server：应用层代码，IDL生成的类、对象这些  
5.底层协议栈和互联网：转换成数字信号发送  
**编码：**  
1.字段编号：每个字段的编号  
2.传输类型：每个字段都有对应的字段传输类型（可变长编码、32位、64位），其中可变长编码就是尽量用最小的字节去序列化一个整数  
3.field：一个字段的完整二进制描述 <<编号，传输类型>值>  
4.tag：由wire-type（后三位）和field组成  
负数有zigzag优化  
**可变长编码：**  
最高位是个标志位，为1表示后面还有其他字节，为0表示没有。每个字节的低7位用来存数值，采用的是小端序，高字节放在高地址  
**反射**
用于动态获取数据，获取了元数据（描述信息），放在.proto文件中  
可以通过元数据构建proto定义的一些描述，动态创建出proto对象，把数据赋值给对象  
接口有三种：**Field Descriptor、File Descriptor、Messgae Factory**  
Field Descriptor：proto文件中消息内的字段或扩展名的描述符  
File Descriptor：proto文件，包括了定义的所有内容  
Message Factory：创建消息对象  

## Docker
应用容器引擎，本质就是进程  
构建dockfile文件搭建环境（From、Run、Copy、Workdir）、dock build生成镜像、dock run启动容器、dock exec进入容器  
镜像文件就是把所需的环境和依赖打包  
ENV DEBIAN-FRONTED = nointeractive表示无交互，往下执行就行  
dirname：打印绝对路径  
Idconfig：更新共享库缓存  
xhost：为了qt显示
-t：镜像文件  
-f：目录文件  
-v：本机代码挂载到容器中  
-d：后台运行  
-f：目录文件  
-it：交互运行  

### 虚拟机和容器的区别
虚拟机：有硬件软件信息，有操作系统、内核  
容器：利用namespace进行资源隔离，利用cgroup对权限和cpu资源限制，容器之间互不影响，不影响宿主机  
