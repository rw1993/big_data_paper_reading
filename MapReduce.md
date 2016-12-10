Mapreduce论文阅读
---

# 0 概述

用户指定map程序和reduce程序，潜在系统自动将这些程序并发地在集群上执行。

# 1 介绍

* 为什么要有这个计算框架：很多任务都是先产生key/value，然后再对相同key的values进行reduce操作，map,reduce是从lisp和其他编程语言那里借鉴的。
* 重执行被作为主要的容错机制。
* 可以被用于集群，也可以是多核的单机（dpark，spark都可以local[4]）

# 2 编程模型

* 计算：
    * input: key/value pairs
    * output: key/value pairs

* 两种函数
    * Map
       1. 输入：key/value pair
       2. 输出：临时的key/value pair
       3. 有一样key的value会被收集在一起，传给reduce
    
    * Reduce:
       1. 输入：中间key/values
       2. 让这些values变小（往往零个或者一个）
       3. 通过实现迭代器，处理values过多的情况。（大概是python里面的file.xreadlines()）
      
## 2.1 例子

word count的例子,原文是伪代码，这里用Python写。

```python
def map(key, value):
    # key :文件名
    # value: 文件内容
    for w in value.split():
        EmitIntermediate(w, '1') #在spark里面好像是flatmap实现的
def combine(key, values):
    # key：中间key
    # values: 相应key的value的迭代器
    result = 0
    for v in values:
        result += int(v)
    EmitIntermediate(w, str(result))
       
def reduce(key, values):
    # key:中间key，也就是word
    # values： value的迭代器
    result = 0
    for value in values:
        result += int(value)
    Emit(str(result))
```

## 2.2 类型

```
map (k1, v1) ->  list(k2, v2)
reduce （k2, list(v2)） -> list(v2)
```

注：这里的k1,v1指的都是类型的范围。

# 3 实现

* 这里介绍的实现时基于google当时的硬件情况，和GFS。
* 用户向调度系统提交job，job包含一系列任务，调度系统将这些任务分配到集群内可用的机器上。


1. 执行概述

    * map请求被分配到多台机器，输入数据会被自动分成M块，这个分割过程可以并行。
    * 使用一个partition函数，中间key值分被分成R份。
    * eg: hash(key) mod R
    * 过程
        1. 用户程序调用MapReduce库中的函数
        2. 用户程序中的MapReduce函数库把输入文件分成M份（每份的大小用户可以定）。然后将程序拷贝到集群的机器中。
        3. 一台机器被选成master，其他成为被分配任务的worker。有M份map任务和R份reduce任务。master选择理想的worker，分配任务。
        4. 分配到map任务的worker读取自己相应部分的input文件，分析出key,value，然后传给用户定义的map函数，map函数的中间key/values被缓存在内存中。
        5. 周期性地，这些缓存被写入本地磁盘，用partition函数分成R个区域。最后，这些信息会被传给master，master再把这些信息传给分配了reduce任务的worker。
        6. reduce worker被master通知那些位置，然后使用远程调用从map worker的磁盘里读取它的那份partition。然后它会将这些按key排序。如果太大，会使用外部排序。
        7. reduce worker用这些数据生成若干个迭代器，依次传给用户的reduce函数，然后输出append到最后输出文件中。
        8. 所有map和reduce任务都弄完以后，返回用户代码。
 
     * 最后的输出是R个reduce的结果文件。

2. master的数据结构

    * task: 状态（idle, in-process, completed）
    * machine: id
    * master是中间文件区域在map worker和reduce worker间传输的管道。似乎是map worker边执行边向master反馈，然后master不断地向reduce worker转达。

3.错误容忍
    
   * worker失效的处理
       * master周期性的ping worker，一段时间ping不通，就认为这个worker不行了，这个worker完成的map工作就变成初始状态，再分配就是了。如果任务的状态是正在进行，不管是reduce任务还是map任务，都把这个任务变成初始状态，重新做。
       * 因为completed的reduce任务把文件存在gfs里面了，所以那个reduce worker就算ping不通了，这个task也不用被重新做。
       * 如果一个任务原来是worker A做，然后worker A不行了换成worker B做，那么所有的reduce worker会被提醒一遍，如果它们还没有从A读取需要的中间数据，那么它们就从B读取。
    
   * 有故障情况下的语义（疑问？什么是确定的函数？？？？？？？）
       * 当用户定义的map和reduce函数是确定的，则我们的过程在有无故障情况下，结果一致。
       * map task和reducer task输出的提交是原子性的。
           1. map worker 和 reduce worker在执行过程中都先把结果写在自己私有的文件里。
           2. 当map worker完成任务以后，它告诉master这些文件的位置（R个文件），如果这个任务已经完成了，那么master忽略，否则master将这些信息保存。
           3. reduce任务完成以后，它会把临时文件重命名为最后的输出文件，我们用GFS来保证最后有R个对应的输出文件。 

4. 位置

    * 调度系统会按远近给分配任务，减少对网络带宽的使用。

5. 任务粒度：M块，每块大小16MB到64MB,R一般是worker数量的几倍
6. 备份任务
    
    * 原因：有的worker因为各种原因执行被分配的任务很慢
    * 解决：任务快结束的时候master重新执行一些正在执行的任务。

7. 加强

    1. 用户定义的partition函数： 按用户想要的方式把中间key分配到不同的reduce worker里。
    2. order保证：传入reduce函数的中间key是字典序升序排列的
    3. combiner：先聚合
    4. 自定义输入输出
    5. 单机调试
























      
      
      
      
      

    