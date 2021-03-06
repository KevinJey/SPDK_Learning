# SPDK入门

SPDK Storage Performance Development Kit 存储性能开发套件 —— 针对于支持nvme协议的SSD设备。
SPDK是一种高性能的解决方案。
## 什么是SPDK

首先要明确spdk是一个框架，而不是一个分布式系统，spdk的基石（官网用了bedrock 这个词）是用户态（user space）、轮询（polled-mode）、异步（asynchronous）、无锁（lockless）的NVMe驱动，其提供了零拷贝、高并发直接从用户态访问ssd的特性。其最初的目的是为了优化块存储落盘。但随着spdk的持续演进，大家发现spdk可以优化存储软件栈的各个方面。

很多分布式存储系统都在思考如何吸纳spdk框架，或是采用spdk代表的高性能存储技术，来优化整条IO链路。

## 名词解释

* iSCSI

Internet小型计算机系统接口，又称为IP-SAN，是一种基于因特网及SCSI-3协议下的存储技术，由IETF提出，并于2003年2月11日成为正式的标准

* 用户态和内核态

操作系统所需要的两种内核状态分别是用户态和内核态。**内核态运行操作系统程序，操作硬件；用户态运行用户程序**

* CPU状态之间的转换

用户态转换到内核态：唯一途径是通过中断、异常、陷入机制（访管指令）
内核态转换到用户态：设置程序状态字psw

* 用户态和内核态的区别
  
类似于Linux的普通用户和root用户

内核态和用户态是操作系统的两种运行级别，当程序运行在三级特权上的时候，就可以称之为运行在用户态，这是最低特权级。大部分用户都运行在用户态。

当程序运行在0级特权级上时，就可以称之为运行在内核态。
运行在用户态下的程序不能直接访问操作系统内部的数据结构以及程序。当我们在系统中执行一个程序的时候，大部分时间是运行在用户态之下的，当他需要操作系统来帮助完成所需要完成的工作的时候就会切换到内核态。

处于用户态执行时，进程所能访问的内存空间和对象受到限制，其所处于占有的处理器是可被抢占的；
处于内核态执行时，则能访问所有的内存空间和对象，且所占有的处理器是不允许被抢占的。
总结，用户态执行时，访问受到限制，资源会被抢占；内核态执行时，访问不受限制，不允许被处理器所抢占。

* 块设备 

对于内存这种容量较小的存储介质通常采用以一个byte字节为单位的访问；对于U盘，软盘硬盘等存储介质，通常用一个块block为单位进行读写，这类设备称之为块设备 


## SPDK

机械硬盘的处理方式：采用中断的方式进行DMA将数据从内核态拷贝到用户态，再交给用户程序处理，**这就是机械硬盘时代的io处理方式**。这样的方式在使用SSD设备时很浪费存储空间。

### SPDK的设计理念

Intel开发了一套基于nvme-ssd的开发套件，SPDK ，存储性能开发套件。使用了两项关键技术UIO以及polling,

spdk主要通过引入以下技术，实现其高性能方案。

1. 将存储用到的驱动转移到用户态，从而避免系统调用带来的性能损耗，顺便可以直接使用用态内存落盘实现零拷贝
2. 使用polling模式
    1. 轮询硬件队列，而不像之前那样使用中断模式，中断模式带来了不稳定的性能和延时的提升
    2. 任何业务都可以在spdk的线程中将轮询函数注册为poller，注册之后该函数会在spdk中周期性的执行，避免了epoll等事件通知机制造成的overhead。
3. 避免在IO链路上使用锁。使用无锁队列传递消息/IO
    1. spdk设计的主要目标之一就随着使用硬件（e.g. SSD，NIC，CPU）的增多而获得性能的线性提升，为了达到这目的，spdk的设计者就必须消除使用更多的系统资源带来的overhead，如：更多的线程、进程间通信，访问更多的存储硬件、网卡带来的性能损耗。
    2. 为了降低这种性能开销，spdk引入了无锁队列，使用lock-free编程，从而避免锁带来的性能损耗。
    3. spdk的无锁队列主要依赖的dpdk的实现，其本质是使用cas（compare and swap）实现了多生产者多消费者FIFO队列。

spdk编程最基本的准则，就是避免在spdk核上出现进程上下文切换。其会打破spdk高性能框架，造成性能降低甚至不能工作。进程上下文切换会因为很多原因导致，大致列举如下：

* cpu时间片耗尽
* 进程在系统资源不足（比如内存不足）时，要等到资源满足后才可以运行，这个时候进程也会被挂起，并由系统调度其他进程运行。
* 进程主动调用sleep等函数让出cpu使用权。
* 当有优先级更高的进程运行时，为了保证高优先级进程的运行，当前进程会被挂起，由高优先级进程来运行。
* 硬件中断会导致CPU上的进程被挂起，转而执行内核的中断服务程序。

## spdk bdev

spdk在上述加速访问NVMe存储的基础上，提供了块设备（bdev）的软件栈，这个块设备并不是linux系统中的块设备，spdk中的块设备只是软件抽象出的接口层。

spdk已经提供了各种bdev，满足不同的后端存储方式、测试需求。如NVMe （NVMe bdev既有NVMe物理盘，也包括NVMeof）、内存（malloc bdev）、不落盘直接返回（null bdev）等等。用户也可以自定义自己的bdev，一个很常见的使用spdk的方式是，用户定义自己的bdev，用以访问自己的分布式存储集群。

**spdk通过bdev接口层，统一了块设备的调用方法**，使用者只要调用不同的rpc将不同的块设备加到spdk进程中，就可以使用各种bdev，而不用修改代码。并且用户增加自己的bdev也很简单，这极大的拓展了spdk的适用场景。

讲到这里，各位同学应该明白了，**spdk目前的的应用场景主要是针对块存储，可以说块存储的整个存储的基石**，再其之上我们又构建了各种文件存储、对象存储、表格存储、数据库等等，我们可以如各种云原生数据库一样将上层的分布式系统直接构建在分布式块存储、对象存储之上，也可以将其他存储需要管理的元数据、索引下推到块层，直接用spdk优化上层存储，比如目前的块存储使用lba作为索引粒度管控，我们可以将索引变为文件/对象，在其之上构建文件/对象存储系统。

## 架构

![SPDK architecture.jpg](./images/SPDK%20architecture.jpg)
SFDK的存储架构分为四层：**驱动层、块设备层、存储服务层、存储协议层**

### 驱动层

NVMe driver：SPDK的基础组件，高度优化且无锁的驱动提供了前所未有的高扩展性，高效性和高性能
IOAT:基于Xeon处理器平台上的copy offload引擎。通过提供用户空间访问，减少了DMA数据移动的阈值，允许对小尺寸I/O或NTB(非透明桥)做更好地利用。
### 块设备层：

NVMe over Fabrics（NVMe-oF）initiator：本地SPDK NVMe驱动和NVMe-oF initiator共享一套公共的API。本地远程API调用及其简便。
RBD:将rbd设备作为spdk的后端存储。
Blobstore Block Device：基于SPDK技术设计的Blobstore块设备，应用于虚机或数据库场景。由于spdk的无锁机制，将享有更好的性能。
Linux AIO：spdk与内核设备（如机械硬盘）交互。
### 存储服务层：

bdev：通用的块设备抽象。连接到各种不同设备驱动和块设备的存储协议，类似于文件系统的VFS。在块层提供灵活的API用于额外的用户功能（磁盘阵列，压缩，去冗等）。
Blobstore：SPDK实现一个高精简的类文件的语义（非POSIX）。这可为数据库，容器，虚拟机或其他不依赖于大部分POSIX文件系统功能集（比如用户访问控制）的工作负载提供高性能支撑。

### 存储协议层

iSCSI target：现在大多使用原生iscsi
NVMe-oF target：实现了新的NVMe-oF规范。虽然这取决于RDMA硬件，NVMe-oF target可以为每个CPU核提供高达40Gbps的流量。
vhost-scsi target：KVM/QEMU的功能利用了SPDK NVMe驱动，使得访客虚拟机访问存储设备时延更低，使得I/O密集型工作负载的整体CPU负载有所下降。

从流程上来看，spdk有数个子构件组成，包括网络前端、处理框架和存储后端。

### 总结

从流程上来看，spdk子构件组成，包括网络前端、处理框架、存储后端组成。
网络前端接受数据，处理框架把iSCSI转换成SCSI块级命令、后端对设备进行读写。
#### 实现细节
前端由DPDK、网卡驱动、用户态网络服务构件组成。DPDK给网卡提供一个高性能的包处理框架；网卡驱动提供一个从网卡到用户态空间的数据快速通道；用户态网络服务则破解TCP/IP包并生成iSCSI命令。

处理框架得到包的内容，并将iSCSI命令翻译为SCSI块级命令。不过，在将这些命令送给后端驱动之前，SPDK提供一个API框架以加入用户指定的功能，即spcial sauce（上图绿框中）。例如缓存，去冗，数据压缩，加密，RAID和纠删码计算等，诸如这些功能都包含在SPDK中。不过这些功能仅仅是为了帮助我们模拟应用场景，需要经过严格的测试优化才可使用。

数据到达后端驱动，在这一层中与物理块设备发生交互，即读与写操作。SPDK包括了几种存储介质的用户态轮询模式驱动：
NVMe设备；
inux异步IO设备如传统磁盘；
基于块地址的内存应用的内存驱动（如RAMDISKS）
可以使用Intel I/O加速技术设备。



接口定义语言

RPD是一种进程之间通信的方式
remote procedure Call 远程过程调用 一个节点请求另一个节点