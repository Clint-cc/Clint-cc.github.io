---
layout: post
title: "分布式计算框架MapReduce"
date: 2019-02-17
description: ""
tag: 大数据
---
### 1.1 概述
​		`MapReduce`他是一个分布式运算程序的编程框架，用户开发“基于`Hadoop`的并行批处理大规模的数据集分析应用”的核心框架。他的核心功能主要是**用户编写的业务逻辑代码**和**自带默认组件**整合成一个完成的分布式运算程序，并发布到集群上。

### 2.1 MR的优缺点

优点：

> * 易于编程，它简单的实现一些接口，就可以完成一个分布式。
> * 扩展性好，当计算资源不能满足需求，可以通过简单的增加机器来扩展计算能力。
> * 高容错性，比如其中一台机器挂了，它可以把上面的计算任务转移到另外一个节点上运行，不至于这个任务运行失败，而且这个过程不需要人工参与，而完全是由Hadoop内部完成的。
> * 适合PB级以上的海量数据的离线处理，因为它可以实现上千台服务器集群的并发工作。

缺点：

> * 不擅长实时计算，它不能像mysql一样，在毫秒内返回结果。
> * 不擅长流式计算，它的输入数据集是静态的，所以自身设计的特点决定它干不了这一类的活儿。
> * 不擅长DAG(有向图)计算，多个应用程序存在依赖关系，后一个应用程序的输入为前一个的输出。MR可以做到，但MR每个作业的输出结果都会写到磁盘，会造成大量的磁盘IO，导致性能非常低。

### 3.1 MR的编程过程（以词频统计为例）

> 第一步：通过input输入数据；
>
> 第二步：`splitting`将文本按照行进行拆分，此时得到K1为行数，V1为对应行的文本内容；
>
> 第三步：Mapper类(`mapping`),将每一行按照空格进行拆分，得到List(K2, V2),K2为每个单词，V2表示出现的次数。
>
> 第四步：由于`mapping`操作可能是在不同的机器上并行处理的，所以需要通过`shuffling`将相同key值的数据分发到同一个节点。
>
> 第五步：Reducer类(`reducing`),汇总各个key的个数并输出
>
> 第六步：Driver类，获取配置信息和job对象实例、指定jar包所在本地路径、关联M/R业务类、指定M/R输出的kv类型、指定最终输出数据的kv类型、指定job输入原始文件所在目录、指定job输出结果所在目录、提交作业。

#### 3.1.1 Combiner是什么，它的作用，实际应用
> &emsp;`Combiner`它是map运算后的可选操作，主要在map计算出中间文件后做一个简单的合并重复key的操作，如词频统计中map在遇到一个单词hello就会记录1，但是可能会出现多次，那么map输出的文件冗余就很多，因此在reduce之前做个合并操作，需要传输的数据量就会减小，传输效率就会提升。**但是**，使用它的前提是不影响reduce的最终输出结果；如，求总数、最大值、最小值都可以，平均值则不能使用。

#### 3.1.2 Partitioner分区
&emsp;满足按照条件输出到不同的文件中，如词频统计中将不同单词的统计结果输出到不同的文件中，还有时间统计产品的销量时，需要先将结果按照种类进行拆分，不过要实现该功能，得自定义`Partitioner`。
&emsp;默认的`Partitioner`是根据key的hashCode对numReduceTasks取余得到的，这样用户无法控制哪个key存储到哪个区。
默认分区代码如下：
```java
public class HashPartitioner<K, V> extends Partitioner<K, V> {
	public int getPartition(K key, V value,int numReduceTasks) {
    	return (key.hashCode() & Integer.MAX_VALUE) % numReduceTasks;
  }
}
```

如果要按照单词进行分类，那么我们继承`Partitioner`自定义分类：

```java
public class CustomPartitioner extends Partitioner<Text, IntWritable> {
    public int getPartitioner(Text text, IntWriteable intWritable, int numPartitions){
       	return WordCountDataUtils.WORD_LIST.indexOf(text.toString());
    }
}
```

构建job时候指定我们自己的分类规则，并设置相应数量的ReduceTask：

```java
// 设置自定义分区规则
job.setPartitionerClass(CustomPartitioner.class);
// 设置 reduce 个数
job.setNumReduceTasks(WordCountDataUtils.WORD_LIST.size());
```

**分区总结：**

> - 如果`ReduceTask`的数量 > `getPartitioner` 的结果数，则会多产生几个空的输出文件part-r-000xx；
> - 如果1<`ReduceTask`的数量 < `getPartitioner` 的结果数，则有一部分分区数据无法安放，抛出Exception；
> - 如果`ReduceTask`的数量 =1，则不管MapTask端输出多少个分区文件，最终结果都交给这一个ReduceTask,最终也就只产生一个结果文件part-r-00000;
> - 分区必须从零开始，逐一累加

案例：如果自定义分区数为5，则：

> 1、job.setNumReduceTasks(1)    正常运行，只不过会产生一个输出文件
>
> 2、job.setNumReduceTasks(2)    报错
>
> 3、job.setNumReduceTasks(6)     正常运行，但会产生空文件

####  3.1.3 Shuffle机制

&emsp;`Shuffle`机制在MapReduce中是比较完整的一个过程，Map方法之后，Reduce方法之前的数据处理过程称之为Shuffle，shuffle是连接Map和Reduce之间的桥梁，Map的输出要用到Reduce中必须经过shuffle这个环节，shuffle的性能高低直接影响了整个程序的性能和吞吐量。因为在分布式情况下，reduce task需要跨节点去拉取其它节点上的map task结果。这一过程将会产生网络资源消耗和内存，磁盘IO的消耗。通常shuffle分为两部分：**Map阶段的数据准备**和**Reduce阶段的数据拷贝处理**。一般将在map端的Shuffle称之为Shuffle Write，在Reduce端的Shuffle称之为Shuffle Read其中包括：分区、排序、归并等等。

**Spark Shuffle：**
&emsp;在Spark的中，负责shuffle过程的执行、计算和处理的组件随着Spark的发展有两种实现方式，分别为`HashShuffleManager`和`SortShuffleManager`，因此spark的Shuffle有Hash Shuffle和Sort Shuffle两种。
&emsp;`HashShuffleManager`的运行机制主要分成两种，一种是普通运行机制，另一种是合并的运行机制。合并机制主要是通过复用buffer来优化Shuffle过程中产生的小文件的数量。Hash shuffle是不具有排序的Shuffle。
&emsp;`SortShuffleManager`的运行机制主要分成两种，一种是普通运行机制，另一种是bypass运行机制。当shuffle read task的数量小于等于spark.shuffle.sort.bypassMergeThreshold参数的值时(默认为200)，就会启用bypass机制。

**Spark shuffle和MapReduce shuffle的区别：**
> 1、整体功能上，两者没有太大的区别；都是将mapper(Spark里是ShuffleMapTask)的输出进行partition，不同的partition送到不同的reducer(Spark里reducer)可能是下一个stage里的ShuffleMapTask，也可能是 ResultTask。Reducer以内存作缓冲区，边shuffle边aggregate数据，等到数据 aggregate 好以后进行 reduce() (Spark 里可能是后续的一系列操作)。

> 2、流程上，两者差别不大;MapReduce是sort-based，进入combine()和reduce()的records必须先sort。这样的好处在于combine/reduce()可以处理大规模的数据，因为其输入数据可以通过外排得到（mapper对每段数据先做排序，reducer的shuffle 对排好序的每段数据做归并)。以前Spark默认选择的是hash-based，通常使用HashMap来对shuffle来的数据进行aggregate，不会对数据进行提前排序。如果用户需要经过排序的数据，那么需要自己调用类似sortByKey()的操作。

> 3、.从流程实现角度来看，两者有不少差别。MapReduce将处理流程划分出明显的几个阶段：map()、spill、merge、shuffle、sort、reduce()等。每个阶段各司其职，可以按照过程式的编程思想来逐一实现每个阶段的功能。Spark中，没有这样功能明确的阶段，只有不同的stage和一系列的transformation()，所以spill、merge、aggregate等操作需要蕴含在transformation()中。
