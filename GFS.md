GFS阅读笔记
---

# 1 介绍

与之前系统的不同：

1. 组件失效是正常情况：

    * 持续监控
    * 错误检测
    * 故障容忍
    * 自动修复

2. 文件很大：block size变大
3. 文件写了以后就变成只读的：appending而不是overwrite
4. 文件api与应用交叉设计

# 2 设计概览

## 2.1 假设

1. 系统从由许多不贵的常常失效的组件构成
2. 系统存着一些大文件（小文件可以存在，但不必优化）
3. 工作量主要由两种读操作构成：
    
    * 大的流式地读
    * 小的随机地读

4. 系统对多个客户端并发地append一个文件的语义支持一定要好。
5. 高带宽比低延迟更重要。？

## 2.2 接口

实现了较为熟悉的文件系统接口，文件夹，文件什么的，和其他操作系统的文件系统都差不多。除了常用的操作，还添加了snapshot和record append。一个用来很低代价地复制文件，一个用来并发地append。

## 2.3 架构

* 一个master，许多chunkserver。
* 许多clients。
* 文件被划分成固定大小的chunks。每个chunk有个叫做chunk haddle的id。
* 每个chunk被备份到多个chunkserver
* master保存meta信息:
    1. 名字空间
    2. 权限
    3. files到chunks的映射
    4. chunks的位置
    
* master管理系统范围内的活动：
    1. chunk出租
    2. 孤儿chunk的垃圾回收
    3. chunkserver之间的chunk迁移

* master和chunk通过心跳事件周期性的沟通(发送指令，获取chunkserver状态)
* client向master获取元信息，然后直接和chunkserver沟通，读写数据
* client和chunkserver都不缓存数据（client因为缓存收益少，chunkserver因为linux本身已经提供了缓存）

## 2.4 单独的master

* 单一master简化了设计
* 单一master要求master尽可能少的参与到文件的读写中，而仅仅是提供元信息
* client会缓存一段时间的元信息

1. 一次简单地读过程
    * client使用chunk size,将文件名的byte range转化成文件的chunk index
    * client发送包含filename和chunk index的请求
    * master返回chunk handle还有备份的位置，client缓存（字典结构{(filename, chunkindex ):chunk handle 和备份位置}）
    * client就近向其中一个备份发送chunk handle 和byte range
    * client再次访问相同的chunk handle就不用继续问master，实际实现中client一次问好几个chunk,master也一次给好几个（附近的一些），不用重复通信。

## 2.5 Chunk size

* 一般64MB
* 不够了才会申请新的chunk
* 好处：
    
    1. client和master通信减少
    2. client和master之间tcp请求变少
    3. master存的meta信息变少

* 坏处：
   
   1. 小文件也要一个chunk，很多机器访问这样的文件形成热点
   2. 解决：多备份


## 2.6 Metadata

* 三大类metadata:
    
    1. file和chunk的名字空间
    2. files到chunks的映射
    3. chunks的备份的位置

* 所有metadata都在内存中，前两个数据还保存在master硬盘的操作日志中，并且备份到远端机器。
* chunks的位置master并不持久化存储，开机时master会询问chunkserver，还有chunkserver加入集群时master也会询问。


1. 内存内的数据结构
    
    * 元信息存在内存里，可以周期性扫描，为一些功能提供基础
    * 由于chunk size的原因，内存比较够用

2. chunk的位置

    * 因为chunkserver对它是不是有这个chunk有最终的话语权，所以存在chunkserver，而不是持久地存在master是一个明智的选择。
 
3. 操作日志

    * 操作日志很重要
    * 因为重要，所以再远端存储，因为在远端存储，减少同步开销，master会批量若干条记录。
    * master通过重做日志中的操作，修复文件系统，为了避免一下子执行太多，设立检查点，仅仅执行检查点之后的操作。检查点建立和新日志写入在不同线程中进行。我们只需要最新的检查点和检查点之后的日志，其他可以删除。

## 2.7 一致性模型

1. GFS提供的保证

    * 文件名字空间的改变是原子性的：namespace locking（4.1详细介绍）
    * file region在数据变更操作之后的状态 取决于变更的类型，变更是否成功，是否有并发的变更。
        1. file region是一致的：所有客户端无论从哪个备份读取，都看到一样的数据
        2. file region在数据变更之后是defined的：如果它是一致的，而且所有客户端都能看到什么改动发生过。当一个变动在没有并发变动的干涉下成功，则是这个file region是的defined，因为客户端都能看到这个变动。
        3. 并发的成功的变动使得这个file region一致但是undefined：所有客户端会看到一致的数据，但是并不是每个变动都被看到。
        4. 失败的变动让region不一致。
        5. 应用可以区分defined和undefined的region。
    * 一系列变动之后，gfs保证变动的file region是一致的。
        1. 所有chunkserver执行变动的顺序一致
        2. chunk version
    * 缓存的chunk index可能从失效的备份中读取数据，但是随着client缓存到期，这个就解决了。
    * 对于失效的组件，master通过和chunkserver握手来识别，并且会通过checksum来检验数据的有效性，发现问题就从有效地备份中拷贝。

2. 应用的含义
   
   * append 而不是write
   * 写自验证，包含id的记录
   

# 3 系统交互

## 3.1 租约和改动顺序


 * master给一个备份授予lease，它成为主备份（primary）
 * lease有timeout，但是primary可以通过心跳事件向master申请延时
 * master决定大顺序，primary决定小顺序
 * 写的一个例子：
    1. client问master哪个chunkserver有相应的chunk的lease,如果没有备份有lease，master授予其中一个备份lease，这个备份成为primary。
    2. master回复primary的id和其他备份的位置。client缓存。
    3. client把数据传给所有备份，任意顺序。chunkserver缓存这些数据直到这些数据被使用，或者超时。
    4. 所有备份都承认收到数据，client向primary发送写请求，primary将这些改变排序，并且在自己那份数据上执行。
    5. primary把写请求转发给其他备份，其他备份按primary排的顺序执行这些变动。
    6. 其他备份向primary报告它们做完这些变动了。    
    7. primary向master报告。如果出现错误，client重新写几次。


## 3.2 数据流

1. 为了充分利用带宽，线性的传输数据
2. eg: client把数据给chunkserver s1 到 s4，那么client先把数据传给离自己最近的一个，然后那一个再把数据传给离它最近的一个，以此类推。
3. 一旦接收到一些数据，就开始向前传输。

## 3.3 原子性的record append

1. client给定数据，gfs保证将这个数据加入文件至少一次，并返回加入的offset。
2. record append的过程和3.1的差不多。但是如果record append在任意一个备份失效，client就会重新进行一次record append，所以有的备份里面可能会出现多次加入这个记录。。。
3. 这样的重复靠application赖解决。。。

## 3.4 snapshot

1. snapshot:瞬间复制
2. copy-on-write
3. 过程：
    1. 回收所有lease，这样所有写操作会经过master。
    2. 记录到硬盘，然后在内存中新建，copy这个时候指向原始的数据（并没有真正copy）
    3. snapshot后，如果有改变snapshot中某个chunk的请求，则master发现指向这个chunk的引用多于一个，则master推迟客户端请求，然后在该chunk所在的chunkserver上新建一个chunk的复制，然后每个备份也新建一个一样的chunk，然后给其中一个lease，客户端以为自己在原来的chunk上，其实自己写的是新的chunk，snapshot指向原来的chunk。这里肯定要改变元数据里面files到chunks的映射。但是仅仅是内存操作了。


# 4 内存操作

## 4.1 名字空间的管理和锁

* 为什么要锁：因为有些master操作很发时间，比如snapshot中回收lease。
* namespace Tree
* /d1/d2/.../dn/leaf上的master操作，需要每一集目录的read锁，和/d1/d2/.../dn/leaf上的write锁和read锁。
* 解释/home/user/foo为什么不会被建立，当/home/user正在被快照为/save/user.
    1. snapshot需要/home和/save的读锁，需要/home/user和/save/user的写锁，create操作需要/home和/home/user的读锁和/home/user/foo的写锁，因为他们要的锁冲突了，所以不会一起执行，肯定有一个先执行。
    2. 这里和linux有点不一样，create不需要父目录的写锁。这样一个目录下可以并发地生成名字不一样的许多文件。

## 4.2 备份放在那儿

* 两个目的
    1. 最大化数据可用性和可靠性
    2. 最小化网络利用

* 需要在两个目的之间折衷。

## 4.3 creation, re-replication, rebalance

* 如题的三种情况导致备份的产生

1. create chunks：
    
    * 当master新建chunk的时候，它决定把初始的空的chunk放在哪里。
    * 考虑的因素：
        1. 硬盘利用率低的chunkserver
        2. 近期放过新chunk的chunkserver不适合再放
        3. 不同机架要有一个（为了可靠性）

2. re-replication
   
   * 备份不够的时候就会重新备份。
   * 备份的优先级：
       1. 丢了两份的比丢了一份的先备份
       2. 活着的比已经删除的先备份
       3. client要用的先备份
   * 为了不影响client，限制了活跃的克隆数量，对于整个集群和单个chunkserver都做了限制。

3. rebalance：不平衡的时候还是会出现，这个时候根据策略让集群重新平衡。


## 4.4 垃圾回收

一个文件刚被删除，gfs不会马上说这块物理存储空间没人用了，这个会在垃圾回收阶段进行。
    

1. 机制

    1. 应用删除文件时，master记录这个事件，并且把文件重命名为一个包含删除时间的隐藏的名字。
    2. 在master日常扫描整个名字区间的时候，它删除超过一定时常的的这样的文件。当这样的文件从名字空间被删除以后，它的元信息也被删除。
    3. 在类似的队chunk名字空间的扫描中，master标记孤儿chunk，删除这些chunk的元信息，在心跳事件中，chunkserver告知master它有那些chunk，master告诉chunkserver哪些chunk的源信息已经被删除了，chunkserver有权删除这些chunk。

## 4.5 坏备份的发现

对于每一个chunk，master保存chunk version number信息，以此检测是否有坏备份。

* chunk version number在master给某一个备份赋予lease的时候生成。
* master通过日常垃圾回收删除坏备份
* master也会把这个version number给client作为另一个安全保障。


# 容错与检测


## 5.1 高可用性

1. 快速恢复
2. chunk备份
3. master备份

## 5.2 数据完整性

checksum，chunkserver自己负责自己的chunk。









