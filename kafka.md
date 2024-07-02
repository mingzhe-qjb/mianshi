kafka的架构
<img width="1036" alt="image" src="https://github.com/mingzhe-qjb/mianshi/assets/174403724/836c84ae-f441-4345-a198-ca1f7a4cb7a0">

Kafka 架构分为以下几个部分：
  Producer：消息生产者，就是向 kafka broker 发消息的客户端。
  Consumer：消息消费者，向 kafka broker 取消息的客户端。
  Topic：可以理解为一个队列，一个 Topic 又分为一个或多个分区。
  Consumer Group：这是 kafka 用来实现一个 topic 消息的广播（发给所有的 consumer）和单播（发给任意一个 consumer）的手段。一个 topic 可以有多个 Consumer Group。
  Broker：一台 kafka 服务器就是一个 broker。一个集群由多个 broker 组成。一个 broker 可以容纳多个 topic。
  Partition：为了实现扩展性，一个非常大的 topic 可以分布到多个 broker上，每个 partition 是一个有序的队列。partition 中的每条消息都会被分配一个有序的id（offset）。将消息发给 consumer，kafka 只保证按一个 partition 中的消息的顺序，不保证一个 topic 的整体（多个 partition 间）的顺序。
  Offset：kafka 的存储文件都是按照 offset.kafka 来命名，用 offset 做名字的好处是方便查找。例如你想找位于 2049 的位置，只要找到 2048.kafka 的文件即可。当然 the first offset 就是 00000000000.kafka。


kafka中的ISR、AR又代表什么？ISR伸缩又是什么？
  分区中的所有副本统称为AR（Assigned Repllicas）。所有与leader副本保持一定程度同步的副本（包括Leader）组成ISR（In-Sync Replicas），ISR集合是AR集合中的一个子集。消息会先发送到leader副本，然后follower副本才能从leader副本中拉取消息进行同步，同步期间内follower副本相对于leader副本而言会有一定程度的滞后。前面所说的“一定程度”是指可以忍受的滞后范围，这个范围可以通过参数进行配置。与leader副本同步滞后过多的副本（不包括leader）副本，组成OSR(Out-Sync Relipcas),由此可见：AR=ISR+OSR。在正常情况下，所有的follower副本都应该与leader副本保持一定程度的同步，即AR=ISR,OSR集合为空。



Kafka 高效文件存储设计特点？
![image](https://github.com/mingzhe-qjb/mianshi/assets/174403724/61fe5be8-6f91-4728-b37b-aa0aa804f2db)

1）Kafka把topic中一个parition大文件分成多个小文件段，通过多个小文件段，就容易定期清除或删除已经消费完文件，减少磁盘占用。
  2）通过索引信息可以快速定位message和确定response的最大大小。
  3）通过index元数据全部映射到memory，可以避免segment file的IO磁盘操作。
  4）通过索引文件稀疏存储，可以大幅降低index文件元数据占用空间大小。

kafka的稳定性保障，从哪几个方面做到的
1. 分区
2. 一个topic下有多个不同partition，每个partition为一个目录，partiton命名规则为topic名称+有序序号，第一个partiton序号从0开始，序号最大值为partitions数量减1
3. ![image](https://github.com/mingzhe-qjb/mianshi/assets/174403724/d826cc27-6142-4446-9746-b6a62cd0ed7a)
partition文件存储

1.每个partion(目录)相当于一个巨型文件被平均分配到多个大小相等segment(段)数据文件中。但每个段segment file消息数量不一定相等，这种特性方便old segment file快速被删除。
2.每个partiton只需要支持顺序读写就行了，segment文件生命周期由服务端配置参数决定
![image](https://github.com/mingzhe-qjb/mianshi/assets/174403724/193fd054-ae5f-4038-a441-e068b7fc9f2b)

segment文件存储

1.segment file组成：由2大部分组成，分别为index file和data file，此2个文件一一对应，成对出现，分别表示为segment索引文件、数据文件.
![image](https://github.com/mingzhe-qjb/mianshi/assets/174403724/ab20362a-e75e-486c-9cf2-e7fb68fc0740)
2.segment文件命名规则：partion全局的第一个segment从0开始，后续每个segment文件名为上一个segment文件最后一条消息的offset值。数值最大为64位long大小，19位数字字符长度，没有数字用0填充。
![image](https://github.com/mingzhe-qjb/mianshi/assets/174403724/bd00ac92-5ad3-4afb-97c0-989f07fd885a)
segment index file采取稀疏索引存储方式，它减少索引文件大小，通过mmap可以直接内存操作，稀疏索引为数据文件的每个对应message设置一个元数据指针,它比稠密索引节省了更多的存储空间，但查找起来需要消耗更多的时间


kafka的吞吐为什么这么大
1. 利用 Partition 实现并行处理
Topic 只是一个逻辑的概念。每个 Topic 都包含一个或多个 Partition，不同 Partition 可位于不同节点。

一方面，由于不同 Partition 可位于不同机器，因此可以充分利用集群优势，实现机器间的并行处理。另一方面，由于 Partition 在物理上对应一个文件夹，即使多个 Partition 位于同一个节点，也可通过配置让同一节点上的不同 Partition 置于不同的磁盘上，从而实现磁盘间的并行处理，充分发挥多磁盘的优势。
2.  顺序读写
很久很久以前就有人做过基准测试：《每秒写入2百万（在三台廉价机器上）》http://ifeve.com/benchmarking-apache-kafka-2-million-writes-second-three-cheap-machines/
3. 分partition，定期清理（由于磁盘有限，不可能保存所有数据，实际上作为消息系统 Kafka 也没必要保存所有数据，需要删除旧的数据。又由于顺序写入的原因，所以 Kafka 采用各种删除策略删除数据的时候，并非通过使用“读 - 写”模式去修改文件，而是将 Partition 分为多个 Segment，每个 Segment 对应一个物理文件，通过删除整个文件的方式去删除 Partition 内的数据。这种方式清除旧数据的方式，也避免了对文件的随机写操作。）
4. 充分利用 Page Cache
引入 Cache 层的目的是为了提高 Linux 操作系统对磁盘访问的性能。Cache 层在内存中缓存了磁盘上的部分数据。当数据的请求到达时，如果在 Cache 中存在该数据且是最新的，则直接将数据传递给用户程序，免除了对底层磁盘的操作，提高了性能。Cache 层也正是磁盘 IOPS 为什么能突破 200 的主要原因之一。

在 Linux 的实现中，文件 Cache 分为两个层面，一是 Page Cache，另一个 Buffer Cache，每一个 Page Cache 包含若干 Buffer Cache。Page Cache 主要用来作为文件系统上的文件数据的缓存来用，尤其是针对当进程对文件有 read/write 操作的时候。Buffer Cache 则主要是设计用来在系统对块设备进行读写的时候，对块进行数据缓存的系统来使用

使用 Page Cache 的好处：

I/O Scheduler 会将连续的小块写组装成大块的物理写从而提高性能

I/O Scheduler 会尝试将一些写操作重新按顺序排好，从而减少磁盘头的移动时间

充分利用所有空闲内存（非 JVM 内存）。如果使用应用层 Cache（即 JVM 堆内存），会增加 GC 负担

读操作可直接在 Page Cache 内进行。如果消费和生产速度相当，甚至不需要通过物理磁盘（直接通过 Page Cache）交换数据

如果进程重启，JVM 内的 Cache 会失效，但 Page Cache 仍然可用

Broker 收到数据后，写磁盘时只是将数据写入 Page Cache，并不保证数据一定完全写入磁盘。从这一点看，可能会造成机器宕机时，Page Cache 内的数据未写入磁盘从而造成数据丢失。但是这种丢失只发生在机器断电等造成操作系统不工作的场景，而这种场景完全可以由 Kafka 层面的 Replication 机制去解决。如果为了保证这种情况下数据不丢失而强制将 Page Cache 中的数据 Flush 到磁盘，反而会降低性能。也正因如此，Kafka 虽然提供了 flush.messages 和 flush.ms 两个参数将 Page Cache 中的数据强制 Flush 到磁盘，但是 Kafka 并不建议使用。

5. 多副本
6. 0拷贝技术
传统模式下，数据从网络传输到文件需要 4 次数据拷贝、4 次上下文切换和两次系统调用。
这一过程实际上发生了四次数据拷贝：

首先通过 DMA copy 将网络数据拷贝到内核态 Socket Buffer

然后应用程序将内核态 Buffer 数据读入用户态（CPU copy）

接着用户程序将用户态 Buffer 再拷贝到内核态（CPU copy）

最后通过 DMA copy 将数据拷贝到磁盘文件
![image](https://github.com/mingzhe-qjb/mianshi/assets/174403724/8d3f4ca4-4d88-4dc0-9234-bbceb4ffa254)

零拷贝（Zero-copy）技术指在计算机执行操作时，CPU 不需要先将数据从一个内存区域复制到另一个内存区域，从而可以减少上下文切换以及 CPU 的拷贝时间。

它的作用是在数据报从网络设备到用户程序空间传递的过程中，减少数据拷贝次数，减少系统调用，实现 CPU 的零参与，彻底消除 CPU 在这方面的负载。

目前零拷贝技术主要有三种类型[3]：

直接I/O：数据直接跨过内核，在用户地址空间与I/O设备之间传递，内核只是进行必要的虚拟存储配置等辅助工作；

避免内核和用户空间之间的数据拷贝：当应用程序不需要对数据进行访问时，则可以避免将数据从内核空间拷贝到用户空间

mmap

sendfile

splice && tee

sockmap

copy on write：写时拷贝技术，数据不需要提前拷贝，而是当需要修改的时候再进行部分拷贝。
Kafka 在这里采用的方案是通过 NIO 的 transferTo/transferFrom 调用操作系统的 sendfile 实现零拷贝。总共发生 2 次内核数据拷贝、2 次上下文切换和一次系统调用，消除了 CPU 数据拷贝
5. 批处理
在很多情况下，系统的瓶颈不是 CPU 或磁盘，而是网络IO。

因此，除了操作系统提供的低级批处理之外，Kafka 的客户端和 broker 还会在通过网络发送数据之前，在一个批处理中累积多条记录 (包括读和写)。记录的批处理分摊了网络往返的开销，使用了更大的数据包从而提高了带宽利用率。

6. 数据压缩
Producer 可将数据压缩后发送给 broker，从而减少网络传输代价，目前支持的压缩算法有：Snappy、Gzip、LZ4。数据压缩一般都是和批处理配套使用来作为优化手段的。


小总结 | 下次面试官问我 kafka 为什么快，我就这么说
partition 并行处理

顺序写磁盘，充分利用磁盘特性

利用了现代操作系统分页存储 Page Cache 来利用内存提高 I/O 效率

采用了零拷贝技术

Producer 生产的数据持久化到 broker，采用 mmap 文件映射，实现顺序的快速写入

Customer 从 broker 读取数据，采用 sendfile，将磁盘文件读到 OS 内核缓冲区后，转到 NIO buffer进行网络发送，减少 CPU 消耗
 
