---
layout: post
title: "Spark基础之SparkCore"
date: 2019-03-01
description: ""
tag: 大数据
---
### 1、创建RDD
```
	val rdd = sc.parallelize(Array(1,2,3,4))
	val rdd = sc.makeRDD(Array(1,2,3,4))
	rdd.collect()    # 显示结果
```

### 2、RDD的转换：Value类型

**map(func)**: 对原RDD中每个元素运用func函数，生成新的RDD;
```
val map_rdd = rdd.map(_*2)
//输出结果：2,4,6,8
```

**map(func)**: 与map类似，但函数单独在RDD的每个分区上运行;
```
val mapPara_rdd = rdd.mapPartitions(x=>x.map(_*2))
// 输出结果：2,4,6,8
```

**mapPartitionsWithIndex(func)**:与类似mapPartitions，但func带有一个整数参数表示分片的索引值
```
val mapParaWIn_rdd = rdd.mapPartitionsWithIndex((index,items)=>(items.map((index,_))))
// 输出结果：Array((0,1),(0,2),(1,3),(1,4))
```

**flatMap**: 每一个输入的item被映射成0个或多个输出的items,func返回序列;
```
val flatMap_rdd = rdd.flatMap(1 to _)
// 输出结果：Array(1,1,2,1,2,3,1,2,3,4,1,2,3,4,5)
```

**glom**: 将每个分区形成一个数组，形成新的RDD类型时RDD
```
val rdd = sc.parallelize(1 to 16,4)   # 创建4个分区的RDD
val glom_rdd = rdd.glom()
// 输出结果：Array(Array(1,2,3,4),Array(5,6,7,8),Array(9,10,11,12),Array(13,14,15,16))
```

**gropBy(func)**:按照传入函数的返回值进行分组，将相同的key对应的值放入迭代器
```
val gropBy_rdd = rdd.gropBy(_%2)
//输出结果：Array((0,CompactBuffer(2,4)),(1,CompactBuffer(1,3)))
```

**filter(func)**：经过func过滤返回
```
val rdd = sc.parallelize(Array("clintcheng","clintxiao","clintzhan","eve","sam"))
val filter_rdd = rdd.filter(_.contains("clint"))
// 输出结果：Array(clintcheng,clintxiao,clintzhan)
```

**sample(withReplacement,fraction,seed)**:
```
withReplacement：是否放回；fraction：抽出百分比数量；seed：指定随机种子
val rdd = sc.parallelize(1 to 10)
val sample_rdd = rdd.sample(true,0.4,2)
// 输出结果：Array(1,2,2,7,7,8,9)
```

**distinct([numTasks])**: 去重，默认有8个并行任务操作，但可传一个可选的numTasks参数改变
```
val rdd = sc.parallelize(list(1,3,5,7,4,3,1,9,9,8,3))
val distinct_rdd = rdd.distinct() # 未指定并行度
//输出结果：Array(4,8,1,9,5,3,7)
```

**coalesce(numPartitions)**: 缩减分区数, 用于大数据过滤后，提高小数据集的执行效率
```
val rdd = sc.parallelize(1 to 16, 4)
rdd.partitions.size # 查看分区数
val coalesce_rdd = rdd.coalesce(3)
coalesce_rdd.partitions.size
```

**repartitions(numPartitions)**: 根据分区数，重新通过网络随机洗牌所有数据

**sortBy(func,[ascending],[numTasks])**: func先对数据进行处理，按照处理后的数据比较结果排序，默认为正序

**pipe(command, [envVars])**: 管道，针对每个分区，都执行一个shell脚本，返回输出的RDD；脚本需要放在Worker节点可以访问到的位置
```shell
编写shell脚本
#!/bin/sh
echo "AA"
while read LINE; do
 echo ">>>"${LINE}
done
```
	# 创建只有一个分区的RDD
	val rdd = sc.parallelize(List("hi","Hello","how","are","you"),1)
	# 将脚本作用该RDD并打印
	rdd.pipe("/opt/module/spark/pipe.sh").collect()
	// 输出结果：Array(AA, >>>hi, >>>Hello, >>>how, >>>are, >>>you)

	# 创建有两个分区的RDD
	val rdd = sc.parallelize(List("hi","Hello","how","are","you"),2)
	# 将脚本作用该RDD并打印
	rdd.pipe("/opt/module/spark/pipe.sh").collect()
	// 输出结果：Array(AA, >>>hi, >>>Hello, AA, >>>how, >>>are, >>>you)

### 3、RDD的转换: 双Value类型
**union**: 对RDD1和RDD2求并集后返回新的RDD

**subtract**: 求第一个RDD与第二个RDD的差集

**intersection**: 求两个RDD的交集

**cartesian**: 求两个RDD的笛卡尔积

**zip**：组合两个RDD成Key/Value形式的RDD,默认两个RDD的partition数量以及元素数量都相同，否则抛出异常


### 4、RDD的转换: Key-Value类型
**partitionBy**: 对pairRDD进行分区操作，如果原有的partionRDD和现有的partionRDD是一致的话就不进行分区， 否则会生成ShuffleRDD,即会产生shuffle过程
```
# 创建一个4个分区的RDD，对其重新分区
val rdd = sc.parallelize(Array((1,"aaa"),(2,"bbb"),(3,"ccc"),(4,"ddd")),4)
rdd.partitions.size   # 4
var rdd2 = rdd.partitionBy(new org.apache.spark.HashPartitioner(2))
rdd2.partitions.size  # 2
```

**reduceByKey(func,[numTasks])**: 按照key值进行分组，并对分组后的数据执行归约操作;
**groupByKey**: groupByKey也是对每个key进行操作，但只生成一个seq;

两者区别：
>* 1、reduceByKey按照key进行聚合，在shuffle之前有combine（预聚合）操作，返回结果是RDD[k,v]。
>* 2、groupByKey按照key进行分组，直接进行shuffle。

aggregateByKey:
当调用（K，V）对的数据集时，返回（K，U）对的数据集，其中使用给定的组合函数和zeroValue聚合每个键的值。
与groupByKey类似，reduce任务的数量可通过第二个参数进行配置。

**foldByKey**:

**combineByKey[C]**:

**sortByKey([ascending],[numTasks])**

**mapValues**：

**join**: 在类型为(K,V)和(K,W)的RDD上调用，返回一个相同key对应的所有元素对在一起的(K,(V,W))的RDD;

**cogroup(RDD,[numTasks])**: 在类型为(K,V)和(K,W)的RDD上调用，返回一个(K,(Iterable<V>,Iterable<W>))类型的RDD;

### 5、Action常用算子
> **reduce[func]**: 通过func函数聚集RDD中的所有元素，先聚合分区内数据，再聚合分区间数据。

> **collect()**: 以数组的形式返回数据集的所有元素。

> **count()**: 返回RDD中元素的个数。

> **first()**: 返回RDD中的第一个元素。

> **take(n)**: 返回一个由RDD的前n个元素组成的数组。

> **takeOrdered(n)**: 返回RDD排序后的前n个元素组成的数组。

> **aggregate**:

	aggregate函数将每个分区里面的元素通过seqOp和初始值进行聚合，然后用combine函数将每个分区的结果和初始值(zeroValue)进行combine操作。这个函
	数最终返回的类型不需要和RDD中元素类型一致;

> **fold(num)(func)**: 折叠操作，aggregate的简化操作，seqop和combop一样;

> **saveAsTextFile(path)**:

	将数据集的元素以textfile的形式保存到HDFS文件系统对于每个元素，Spark将会调用toString方法，将它装换为文件中的文本;

> **saveAsSequenceFile(path)**:将数据集中的元素以Hadoop sequencefile的格式保存到指定的目录下，可以使HDFS或者其他Hadoop支持的文件系统;

> **saveAsObjectFile(path)**:用于将RDD中的元素序列化成对象，存储到文件中;

> **countByKey()**:针对(K,V)类型的RDD，返回一个(K,Int)的map，表示每一个key对应的元素个数;

> **foreach(func)**:在数据集的每一个元素上，运行函数func进行更新;

### 6、RDD依赖关系：
	窄依赖指的是每一个父RDD的Partition最多被子RDD的一个Partition使用；
	宽依赖指的是多个子RDD的Partition会依赖同一个父RDD的Partition，会引起shuffle；

### 7、RDD缓存：
	RDD通过persist方法或cache方法可以将前面的计算结果缓存，默认情况下persist()会把数据以序列化的形式缓存在JVM的堆空间中。
	但是并不是这两个方法被调用时立即缓存，而是触发后面的action时，该RDD将会被缓存在计算节点的内存中，并供后面重用。

### 8、RDD CheckPoint：
	Spark中对于数据的保存除了持久化操作之外，还提供了一种检查点的机制，检查点（本质是通过将RDD写入Disk做检查点）是为了通过lineage做容错的辅助，
	lineage过长会造成容错成本过高，这样就不如在中间阶段做检查点容错，如果之后有节点出现问题而丢失分区，从做检查点的RDD开始重做Lineage，就会减少
	开销。检查点通过将数据写入到HDFS文件系统实现了RDD的检查点功能。

	为当前 RDD 设置检查点。该函数将会创建一个二进制的文件，并存储到 checkpoint 目录中，该目录是用 SparkContext.setCheckpointDir()设置的。在
	checkpoint的过程中，该RDD 的所有依赖于父 RDD 中的信息将全部被移除。对 RDD 进行 checkpoint 操作并不会马上被执行，必须执行 Action 操作才能
	触发。

### 9、类型器
#### 9.1 自定义累加器
#### 9.2 广播变量（调优策略）
