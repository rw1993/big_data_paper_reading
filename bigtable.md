BigTable
---

# 0 概述

关键词： 结构外数据，scale

# 1 介绍

关键词：类似数据库，动态控制，数据被视为不分割的字符串。row,column。

# 2 data model

* bigtable:稀疏，分布式，持久化，多维的map。

```
(row:string, column:string, time:int64) → string
```

* rows: 
    1. 一行的读写是原子的
    2. 数据按rowkey的字典序排列
    3. tablet: row range,动态划分的，是负载均衡和分布式的最小单位
    4. 读取一定row range是高效的（因为存在一个机器或者相近的机器）

* column families
    1. 是最小的access控制单位
    2. column families并需预先创建好，并且改动不多，并且不太多。
    3. family: qualifier这样的结构构成column key
    4. 权限控制以column family为最小单位进行

* 时间戳

    1. 时间戳用于确定版本
    2. 可以设定保留确定的版本或者保留一定时间之内的

# 3 API

1. set
2. delete
3. scan
4. 单行事务
5. Sawzall（一个读取bigtable的工具）
6. 可以与mapreduce合作

# 4 building blocks组件

1. gfs
2. sstable
3. **Chubby**:负责很多事情。。。。

# 5 实现

* 三个主要的构建：
   1. client都要有的library
   2. master server
   3. tablet servers：可以动态添加

* tablet 位置

    1. 使用三层类似B+树的结构存储tablet的位置，root tablet的位置在Chubby中
        * root tablet(meta tablet的第一个,从来不分裂）
        * metadata tablet：以table name + endrow作为key 
        * user tablet
    
    2. client会缓存tablet的位置
        * 如果没有缓存，问3次就知道tablet的位置（Chubby, root, meta）
        * 如果缓存失效，最多问六次就知道tablet的位置（user,meta,root,Chubby,root,meta,user）
        * 一次请求meta会返回多个user tablet的位置（和gfs中的策略相近）
    
    3. metadata table也存一些log
 
 * tablet 分配
    
    * Chubby被用来跟踪tablet server（觉得有点像zookeeper在hadoop生态中的作用）
    * tablet server开始，它便在Chubby创造一个唯一名字的文件，并且得到其排它锁。
    * master监控这些文件，以发行新的tablet server
    * 如果tablet 失去这个排它锁，它就不服务了，如果文件还在，它会重新申请这个排他锁，如果文件不见了，它会杀死自己。当某个tablet server被停止，它会尽力释放这个排它锁。
    * master会周期性地询问tablet server其排它锁的状态，如果联系不上或者排它锁丢了，master会申请这个tablet server相应文件上的排它锁，如果master申请到了，说明Chubby是好使的，tablet server出了问题，所以master就删除Chubby上的tablet server的文件。然后把这个tablet server的tablet都移动到未分配状态。
    * 当master联系不上Chubby，它就杀死自己，这不会影响之前tablet的分配。
    * master启动时，获取当前分配的步骤：
        1. 从Chubby获取唯一的master lock
        2. 浏览Chubby的server目录获取当前活着的tablet server
        3. 访问每个活着的tablet server，获取已经分配到tablet server的tablet
        4. 浏览metadata table,发现未分配的tablet,就分配
    * 3.5 4之前，tablet会问问root tablet有没有分配，如果没有的话，它会分配。

* tablet 服务

    




















