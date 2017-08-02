---
layout:     post
title:      "Hadoop的MapReduce编程实践"
subtitle:   " \"K-means的MapReduce实现\""
date:       2017-07-12 
author:     "xihoo"
tags:
    - Hadoop
    - java
---

## K-means算法

**K-means**算法是典型的基于距离的聚类算法，采用距离作为相似性的评价指标。  

算法过程如下：  

1. 选择k个初始中心点。  

2. 计算n个数据与中心点的距离，数据距离哪个中心点最近就划分为哪个类  

3. 根据每个类的数据重新计算中心点  

4. 重复第2，3步，直到中心点的变化小于给定的阈值  

## Map过程

代码如下：
```java

public static class Map extends Mapper<LongWritable, Text, IntWritable, Text>{
        ArrayList<ArrayList<Double>> centers = null;
        int k = 0;
        protected void setup(Context context) throws IOException,
                InterruptedException {
            centers = Utils.getCentersFromHDFS(context.getConfiguration().get("centersPath"),false);
            k = centers.size();
        }
        protected void map(LongWritable key, Text value, Context context)
                throws IOException, InterruptedException {
            ArrayList<Double> fileds = Utils.textToArray(value);
            int sizeOfFileds = fileds.size();
            double minDistance = 99999999;
            int centerIndex = 0;
            for(int i=0;i<k;i++){//取出k个中心点与当前读取的记录做计算
                double currentDistance = 0;
                for(int j=0;j<sizeOfFileds;j++){
                    double centerPoint = Math.abs(centers.get(i).get(j));
                    double filed = Math.abs(fileds.get(j));
                    currentDistance += Math.pow((centerPoint - filed) / (centerPoint + filed), 2);
                }
                if(currentDistance<minDistance){//循环找出距离该记录最接近的中心点的ID
                    minDistance = currentDistance;
                    centerIndex = i;
                }
            }
            context.write(new IntWritable(centerIndex+1), value);
        }
    }

```

每个Mapper读取一行数据，将数据与k个中心点相比较，输出的key为中心点ID，value为数据；

***

## Reduce过程

代码如下：
```java

public static class Reduce extends Reducer<IntWritable, Text, Text, Text>{
        protected void reduce(IntWritable key, Iterable<Text> value,Context context)
                throws IOException, InterruptedException {
            ArrayList<ArrayList<Double>> filedsList = new ArrayList<ArrayList<Double>>();
            for(Iterator<Text> it =value.iterator();it.hasNext();){
                ArrayList<Double> tempList = Utils.textToArray(it.next());
                filedsList.add(tempList);
            }
            int filedSize = filedsList.get(0).size();
            double[] avg = new double[filedSize];
            for(int i=0;i<filedSize;i++){
                double sum = 0;
                int size = filedsList.size();
                for(int j=0;j<size;j++){
                    sum += filedsList.get(j).get(i);
                }
                avg[i] = sum / size;
            }
            context.write(new Text("") , new Text(Arrays.toString(avg).replace("[", "").replace("]", "")));
        }
        
    }
```

传入Reducer时的数据是key为中心点ID，value为所属中心点ID的所有数据，在Reducer中，计算出中心点所属类
的平均值所为新的中心点，并输出这个平均值  

## run类与主函数

代码如下：  

```java

public static void run(String centerPath,String dataPath,String newCenterPath,boolean runReduce) throws IOException, ClassNotFoundException, InterruptedException{
        Configuration conf = new Configuration();
        conf.set("centersPath", centerPath);
        Job job = new Job(conf, "mykmeans");
        job.setJarByClass(MapReduce.class);
        job.setMapperClass(Map.class);
        job.setMapOutputKeyClass(IntWritable.class);
        job.setMapOutputValueClass(Text.class);
        if(runReduce){//最后依次输出不需要reduce
            job.setReducerClass(Reduce.class);
            job.setOutputKeyClass(Text.class);
            job.setOutputValueClass(Text.class);
        }
        FileInputFormat.addInputPath(job, new Path(dataPath));
        FileOutputFormat.setOutputPath(job, new Path(newCenterPath));
        System.out.println(job.waitForCompletion(true));
    }
public static void main(String[] args) throws ClassNotFoundException, IOException, InterruptedException {
        String centerPath = "hdfs://localhost:9000/input/centers.txt";
        String dataPath = "hdfs://localhost:9000/input/wine.txt";
        String newCenterPath = "hdfs://localhost:9000/out/kmean";
        int count = 0;
        while(true){
            run(centerPath,dataPath,newCenterPath,true);
            System.out.println(" 第 " + ++count + " 次计算 ");
            if(Utils.compareCenters(centerPath,newCenterPath )){
                run(centerPath,dataPath,newCenterPath,false);
                break;
            }
        }
    }
```

run类是实现一次mapreduce过程，在主函数中对比reduce求出的平均值与原来的中心，如果不相同，这将清空原
中心的数据文件，将reduce的结果写到中心文件中，删掉reduce的输出目录以便下次输出。继续运行任务。对比
reduce求出的平均值与原来的中心，如果相同。则删掉reduce的输出目录，运行一个没有reduce的任务将中心ID
与值对应输出  

Utils类中封装了`getCentersFromHDFS()` `deletePath()`  `textToArray()`  `compareCenters()`
这些函数。