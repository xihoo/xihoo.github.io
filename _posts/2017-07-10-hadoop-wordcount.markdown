---
layout:     post
title:      "Hadoop的MapReduce编程初探"
subtitle:   " \"简析WordCount\""
date:       2017-07-10 
author:     "xihoo"
tags:
    - Hadoop
    - java
---

## MapReduce基础

初次接触MapReduce首先想起的是内置在Python中的`map()` `reduce()`函数，在Hadoop中MapReduce框架复杂
了许多，map负责把任务分解成多个任务，reduce负责把分解后多任务处理的结果汇总起来  

1. **在Map阶段**，MapReduce框架将任务的输入数据分割成固定大小的split，然后将每一个split分解成键值
对<K1,V1>,每一个split对应一个Mapper，执行用户自定义的`map()`函数(以键值对<K1,V1>作为输入)，执行完
毕后得到键值对<K2,V2>
2. **中间阶段**，对<K2,V2>按照K2进行排序，key值相同的放在一起形成一个列表<K2,list<V2>>,然后根据
key值的范围分组，对应不同的Reduce任务
3. **Reduce阶段**，Reducer把从不同Mapper接收来的数据整合在一起并进行排序，然后调用用户自定义的
reduce函数，得到键值对<K3,V3>

***

## WordCount源码分析

### 1.Map过程

Map过程需要继承`org.apache.hadoop.mapreduce`包中的Mapper类，并重写其map方法。

```java

public static class TokenizerMapper
                extends Mapper<Object, Text, Text, IntWritable>{
                private final static IntWritable one = new IntWritable(1);
                private Text word = new Text();
                public void map(Object key, Text value, Context context)throws IOException, InterruptedException {
                    StringTokenizer itr = new StringTokenizer(value.toString());
                    while (itr.hasMoreTokens()) {
                        word.set(itr.nextToken());
                        context.write(word, one);
                    } 
                } 
        }

```

map方法的输入的value值是文件中的一行，key值是相对文件开端的偏移，利用StringTokenizer类将每一行
拆分成一个个的word，并将<\word,1>作为输出。

### 2.Reduce过程

Reduce过程需要继承`org.apache.hadoop.mapreduce`包中的Reducer类，并重写reduce方法，其输入
参数中key为单个单词，values是单词的计数值组成的列表。

```java

public static class IntSumReducer extends Reducer<Text,IntWritable,Text,IntWritable> {
            private IntWritable result = new IntWritable();
            public void reduce(Text key, Iterable<IntWritable> values,Context context)throws IOException, InterruptedException {
                int sum = 0;
                for (IntWritable val : values) {
                    sum += val.get();
                }
                result.set(sum);
                context.write(key, result);
            } //reduce
        }

```

遍历values求和即得到某个单词的计数值。

### 3.main函数

Job对象负责管理和运行一个计算任务。

```java 

public static void main(String[] args) throws Exception {  
        Configuration conf = new Configuration();  
        Job job = new Job(conf);  
        job.setJarByClass(WordCount.class);  
        job.setJobName("wordcount");  
  
        job.setOutputKeyClass(Text.class);  
        job.setOutputValueClass(IntWritable.class);  
  
        job.setMapperClass(TokenizerMapper.class);  
        job.setReducerClass(IntSumReducer.class);  
  
        job.setInputFormatClass(TextInputFormat.class);  
        job.setOutputFormatClass(TextOutputFormat.class);  
  
        FileInputFormat.addInputPath(job, new Path(args[0]));  
        FileOutputFormat.setOutputPath(job, new Path(args[1]));  
  
        job.waitForCompletion(true);  
    }

```

`main()`函数中设置了使用TokenizerMapper完成map过程，使用IntSumReducer完成Combine和Reduce
过程，设置类型参数，Text和IntWritable都是Hadoop的封装；

***

## 简析WordCount的处理过程

每一个文件对应一个split，传入Mapper的是每一行的内容，map之后是一个个单词，计数值都为1，combine过程
累加了计数值，传入Reducer之后排序，对相同key值的计数值累加，最后输出。

![](/img/map1.png)
![](/img/map2.png)
![](/img/reduce.png)

