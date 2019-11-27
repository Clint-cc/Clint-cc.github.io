---
layout: post
title: "MapReduce之WordCount词频统计实操"
date: 2019-02-18
description: ""
tag: 大数据
---
## 本地测试(Windows10下)
### 一、windows上搭建hadoop环境
#### 1-1、首先到官方下载官网的hadoop2.7.7
链接：[https://mirrors.tuna.tsinghua.edu.cn/apache/hadoop/common/](https://mirrors.tuna.tsinghua.edu.cn/apache/hadoop/common/)

#### 1-2、在网盘中下载hadooponwindows-master.zip
网盘链接：[https://pan.baidu.com/s/1vxtBxJyu7HNmOhsdjLZkYw](https://pan.baidu.com/s/1vxtBxJyu7HNmOhsdjLZkYw) 提取码：y9a4

#### 1-3、解压hadoop-2.7.7.tar.gz文件，注意不要解压到有空格或中文的路径中，然后解压hadooponwindows-master的bin和etc替换hadoop2.7.7的bin和etc

#### 1-4、添加系统变量
>* 变量名：HADOOP_HOME，变量值：E:\A-software-jar\hadoop-2.7.7（这里填自己解压的路径）
>* path中添加%HADOOP_HOME%\bin

#### 1-5、修改配置文件（E:\A-software-jar\hadoop-2.7.7\etc\hadoop目录中）
hadoop-env.cmd 中修改：
```xml
set JAVA_HOME=E:\JDK\jdk1.8.0_171（自己的jdk安装路径）
```

hdfs-site.xml 中修改：
```xml
<configuration>
    <property>
        <name>dfs.replication</name>
        <value>1</value>
    </property>
    <property>
        <name>dfs.namenode.name.dir</name>
        <value>file:/E:/A-software-jar/hadoop-2.7.7/data/namenode</value>
    </property>
    <property>
        <name>dfs.datanode.data.dir</name>
        <value>file:/E:/A-software-jar/hadoop-2.7.7/data/datanode</value>
    </property>
</configuration>
```

#### 1-6、把hadoop.dll从E:\A-software-jar\hadoop-2.7.7\bin拷贝到 C:\Windows\System32中

### 二、在IDEA中创建maven

#### 在pom.xml中添加如下依赖：
```xml
<dependencies>
    <dependency>
        <groupId>junit</groupId>
        <artifactId>junit</artifactId>
        <version>RELEASE</version>
    </dependency>
    <dependency>
        <groupId>org.apache.logging.log4j</groupId>
        <artifactId>log4j-core</artifactId>
        <version>2.8.2</version>
    </dependency>
    <dependency>
        <groupId>org.apache.hadoop</groupId>
        <artifactId>hadoop-common</artifactId>
        <version>2.7.7</version>
    </dependency>
    <dependency>
        <groupId>org.apache.hadoop</groupId>
        <artifactId>hadoop-client</artifactId>
        <version>2.7.7</version>
    </dependency>
    <dependency>
        <groupId>org.apache.hadoop</groupId>
        <artifactId>hadoop-hdfs</artifactId>
        <version>2.7.7</version>
    </dependency>
</dependencies>
```

#### 在项目的src/main/resources目录下，新建log4j.properties文件，文件中添加如下
```
log4j.rootLogger=INFO, stdout
log4j.appender.stdout=org.apache.log4j.ConsoleAppender
log4j.appender.stdout.layout=org.apache.log4j.PatternLayout
log4j.appender.stdout.layout.ConversionPattern=%d %p [%c] %m%n
log4j.appender.logfile=org.apache.log4j.FileAppender
log4j.appender.logfile.File=target/spring.log
log4j.appender.logfile.layout=org.apache.log4j.PatternLayout
log4j.appender.logfile.layout.ConversionPattern=%d %p [%c] %m%n
```

#### 在src/main/java下创建com.clint.WordCount包，在包下创建WcMapper、WcReducer、WcDriver三个类如下
WcMapper类：
```java
package com.clint.WordCount;

import java.io.IOException;

import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.LongWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Mapper;

public class WcMapper extends Mapper<LongWritable, Text, Text, IntWritable> {
    private Text k = new Text();
    private IntWritable v = new IntWritable(1);

    @Override
    protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
        // 1 获取一行
        String line = value.toString();
        // 2 按照空格切割数据
        String[] words = line.split(" ");
        // 3 遍历数组，把单词变成<k, v>输出
        for (String word : words) {
            k.set(word);
            context.write(k, v);
        }
    }
}
```

WcReducer类：
```java
package com.clint.WordCount;

import java.io.IOException;

import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Reducer;

public class WcReducer extends Reducer<Text, IntWritable, Text, IntWritable> {
    private IntWritable v = new IntWritable();

    @Override
    protected void reduce(Text key, Iterable<IntWritable> values, Context context) throws IOException,
            InterruptedException {
        // 1 累加求和
        int sum = 0;
        for (IntWritable count : values) {
            sum += count.get();
        }
        // 2 输出
        v.set(sum);
        context.write(key, v);
    }
}
```

WcDriver类：
```java
package com.clint.WordCount;

import java.io.IOException;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;

public class WcDriver {
    public static void main(String[] args) throws IOException, ClassNotFoundException, InterruptedException {
        // 1 获取配置信息以及封装任务
        Configuration configuration = new Configuration();
        Job job = Job.getInstance(configuration);

        // 2 设置 jar 加载路径
        job.setJarByClass(WcDriver.class);

        // 3 设置 map 和 reduce 类
        job.setMapperClass(WcMapper.class);
        job.setReducerClass(WcReducer.class);

        // 4 设置 map 输出
        job.setMapOutputKeyClass(Text.class);
        job.setMapOutputValueClass(IntWritable.class);

        // 5 设置reducer最终输出 kv 类型
        job.setOutputKeyClass(Text.class);
        job.setOutputValueClass(IntWritable.class);

        // 6 设置数据输入和输出路径
        FileInputFormat.setInputPaths(job, new Path(args[0]));
        FileOutputFormat.setOutputPath(job, new Path(args[1]));

        // 7 提交
        boolean result = job.waitForCompletion(true);
        System.exit(result ? 0 : 1);
    }
}
```

### 二、提交到集群上运行
#### 由于项目没有使用除Hadoop外的第三方依赖，直接打包即可;
```
mvn clean package
```

#### 提交作业
```
hadoop jar /usr/appjar/hadoop-word-count-1.0.jar \
com.heibaiying.WordCountApp \
/wordcount/input.txt /wordcount/output/WordCountApp
```

#### 查看统计结果
```
# 查看目录
hadoop fs -ls /wordcount/output/WordCountApp

# 查看统计结果
hadoop fs -cat /wordcount/output/WordCountApp/part-r-00000
```
