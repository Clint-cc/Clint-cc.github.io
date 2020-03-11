---
layout: post
title: "MapReduce中常见问题"
date: 2019-02-18
description: ""
tag: 大数据
---

### 1、数据倾斜的产生和解决办法
#### 1.1 什么是数据倾斜？
简单的说就是数据的key分化严重不均，造成部分数据过多，部分数据很少的局面，举例说明：词频统计中，map阶段形成("hello", 1),然后在reduce阶段进行value相加，得出“hello”的次数， 如果输入文本中有100G，其中有90G的文本是“hello”， 剩下10G是其他的单词，那就会形成90G的数据量交给一个reduce相加，其余的10G根据key不同分散到不同reduce上进行相加的情况，这就造成数据倾斜，导致reduce跑到99%，然后一直等待那个reduce知道执行完。

#### 1.2 为什么说数据倾斜和业务逻辑和数据量有关？

从另外角度看数据倾斜，其本质还是在单台节点在执行那一部分数据reduce任务的时候，由于数据量大，跑不动，造成任务卡住。若是这台节点机器内存够大，CPU、网络等资源充足，跑90G左右的数据量和跑10M数据量所耗时间不是很大差距，那么问题也不大。所以机器配置和数据量存在一个合理的比例，一旦数据量远超机器的极限，那么不管每个key的数据如何分布，总会有一个key的数据量超出机器的能力，造成 reduce缓慢甚至卡顿。

日常使用过程中，容易造成数据倾斜的业务逻辑有很多，可以归纳以下几点：

> - 分组：group by   优于 distinct
> - 去重：distinct   count(distinct xx)
> - 连接：join       其中一个表较小，但是key几种； 大表和小表，但是分桶的判断字段0值或空值过多

#### 1.3 如何处理数据倾斜？

> 1、调优参数
>
> ```java
> set hive.map.aggr = true
> // 在map中会做部分聚焦操作，效率更高但需要更多内存
>     
> set hive.groupby.skewindata = true
> // 数据倾斜时负载均衡，当选项设定为true，生成的查询计划会有两个MRJob。第一个MRJob中，Map的输出结果集合会随机分布到Reduce中，每个Reduce做部分聚合操作，并输出结果，这样处理的结果是相同的GroupBy Key有可能被分发到不同的Reduce中，从而达到负载均衡的目的；第二个MRJob再根据预处理的数据结果按照GroupBy Key分布到Reduce中（这个过程可以保证相同的GroupBy Key被分布到同一个Reduce中），最后完成最终的聚合操作。
> ```
>
> 可以看到第二个参数更重要一些，是计算变成两个mapreduce，先在第一个中在shuffle过程partition时随机给key打标记，使每个key随机均匀分配到各个reduce上计算，但这只能完成部分计算，因为相同key没有分配到相同reduce上，所以需要第二次的mapreduce,这次就回归正常 shuffle,但是数据分布不均匀的问题在第一次mapreduce已经有了很大的改善，因此基本解决数据倾斜。
>
> 2、可以在map阶段将造成数据倾斜的key先分成多组，map时在后面加上1,2,3,4之一，把key分成四组，先运算，之后再恢复key进行最终运算。
>
> 3、能先进行group操作的时候先进行group操作，把key进行一次reduce，再进行count或distinct count。
>
> 4、join操作，使用map join在map端先进行join，以免在reduce时卡住。
>
> 5、其他常用的优化手段：
>
> ​	减少job数、设置合理的task数、数据量大也要慎用count(distinct)、对小文件进行合并

### 2、MR实现二次排序

#### 2.1 二次排序需求说明

在MapReduce操作时，shuffle阶段会多次根据key值排序。但是在shuffle分组后，相同key值的calues序列的顺序是不确定的,想要对values值也是排序的，这就是二次排序。

#### 2,2 实现思路

1、直接在reduce端对分组后的values进行排序

 ```java
// reduce端关键代码
@Override
public void reduce(Text key, Iterable<IntWritable> values, Context context)
    throws IOException, InterruptedException {

    List<Integer> valuesList = new ArrayList<Integer>();

    // 取出value
    for(IntWritable value : values) {
        valuesList.add(value.get());
    }
    // 进行排序
    Collections.sort(valuesList);

    for(Integer value : valuesList) {
        context.write(key, new IntWritable(value));
    }
}
 ```


**注意：** 把排序工作都放在reduce端完成，如果相同key的values序列非常长的话，会对CPU和内存造成极大的负载。

2、将map端输出的<key,value>中的key和value组合成一个新的key（称为newKey），value值不变。这里就变成<(key,value),value>，在针对newKey排序的时候，如果key相同，就再对value进行排序。

> - 自定义数据类型实现组合key：继承WritableComparable。
> - 自定义partioner，形成newKey后保持分区规则任然按照key进行。保证不打乱原来的分区：继承partitioner。
> - 自动以分组，保持分组规则任然按照key进行。不打乱原来的分组：继承RawComparator

自定义数据类型关键代码：

```java
 public class PairWritable implements WritableComparable<PairWritable> {
    // 组合key
      private String first;
      private int second;

    public PairWritable() {
    }

    public PairWritable(String first, int second) {
        this.set(first, second);
    }

    /**
     * 方便设置字段
     */
    public void set(String first, int second) {
        this.first = first;
        this.second = second;
    }

    /**
     * 反序列化
     */
    @Override
    public void readFields(DataInput arg0) throws IOException {
        this.first = arg0.readUTF();
        this.second = arg0.readInt();
    }

    /**
     * 序列化
     */
    @Override
    public void write(DataOutput arg0) throws IOException {
        arg0.writeUTF(first);
        arg0.writeInt(second);
    }

    /*
     * 重写比较器
     */
    public int compareTo(PairWritable o) {
        int comp = this.first.compareTo(o.first);

        if(comp != 0) {
            return comp;
        } else { // 若第一个字段相等，则比较第二个字段
            return Integer.valueOf(this.second).compareTo(
                    Integer.valueOf(o.getSecond()));
        }
    }

    public int getSecond() {
        return second;
    }
    public void setSecond(int second) {
        this.second = second;
    }
    public String getFirst() {
        return first;
    }
    public void setFirst(String first) {
        this.first = first;
    }
```
自定义分区规则：
```java
public class SecondPartitioner extends Partitioner<PairWritable, IntWritable> {
    @Override
    public int getPartition(PairWritable key, IntWritable value, int numPartitions) {
        /*
         * 默认的实现 (key.hashCode() & Integer.MAX_VALUE) % numPartitions
         * 让key中first字段作为分区依据
         */
        return (key.getFirst().hashCode() & Integer.MAX_VALUE) % numPartitions;
    }
}
```
自定义分组比较器:
```java
public class SecondGroupComparator implements RawComparator<PairWritable> {
    /*
     * 对象比较
     */
    public int compare(PairWritable o1, PairWritable o2) {
        return o1.getFirst().compareTo(o2.getFirst());
    }
    /*
     * 字节比较
     * arg0,arg3为要比较的两个字节数组
     * arg1,arg2表示第一个字节数组要进行比较的收尾位置，arg4,arg5表示第二个
     * 从第一个字节比到组合key中second的前一个字节，因为second为int型，所以长度为4
     */
    public int compare(byte[] arg0, int arg1, int arg2, byte[] arg3, int arg4, int arg5) 	{
        return WritableComparator.compareBytes(arg0, 0, arg2-4, arg3, 0, arg5-4);
    }
}
```
### 3、待更新...
