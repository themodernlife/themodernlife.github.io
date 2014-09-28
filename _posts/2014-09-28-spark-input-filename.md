---
layout: post
title:  Getting the current filename with Spark and HDFS
date:   2014-09-28 11:50:13
categories: scala spark hadoop hdfs
---

It's occasionally useful when writing map/reduce jobs to get a hold of the current filename that's being processed.
There's a few ways to do this, depending on the version of Spark that you're using.

Spark 1.1.0 introduced a new method on [HadoopRDD](https://spark.apache.org/docs/1.1.0/api/scala/index.html#org.apache.spark.rdd.HadoopRDD)
that makes this super easy:

```scala
import org.apache.hadoop.io.LongWritable
import org.apache.hadoop.io.Text
import org.apache.hadoop.mapred.{FileSplit, TextInputFormat}
import org.apache.spark.rdd.HadoopRDD

// Create the text file
val text = sc.hadoopFile("file:///tmp/files", classOf[TextInputFormat], classOf[LongWritable], classOf[Text], sc.defaultMinPartitions)
// text: org.apache.spark.rdd.RDD[(org.apache.hadoop.io.LongWritable, org.apache.hadoop.io.Text)] = file:///tmp/files HadoopRDD[2] at hadoopFile at <console>:16

// Cast to a HadoopRDD
val hadoopRdd = text.asInstanceOf[HadoopRDD[LongWritable, Text]]
// hadoopRdd: org.apache.spark.rdd.HadoopRDD[org.apache.hadoop.io.LongWritable,org.apache.hadoop.io.Text] = file:///tmp/files HadoopRDD[2] at hadoopFile at <console>:16

val fileAndLine = hadoopRdd.mapPartitionsWithInputSplit { (inputSplit, iterator) ⇒
  val file = inputSplit.asInstanceOf[FileSplit]
  iterator.map { tpl ⇒ (file.getPath, tpl._2) }
}

fileAndLine.foreach(println)
// prints lines like (file:/tmp/files/part-00000,line1...), (file:/tmp/files/part-00000,line2...), ...
```

There are a few things to note:
- You can't simply use `sc.textFile` because under the hood, the `textFile` method actually maps the resulting
`HadoopRDD`, returning a `MappedRDD` which is not what we want
- This will only work if your `InputFormat` is a subclass of `FileInputFormat`
- `mapPartitionsWithInputSplit` returns a new iterator, in this case we simply map over it (which is done lazily!) to include the split's filename
