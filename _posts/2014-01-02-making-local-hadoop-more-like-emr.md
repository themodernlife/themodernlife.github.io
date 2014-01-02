---
layout: post
title:  "Making Your Local Hadoop more like AWS Elastic MapReduce"
date:   2014-01-02 11:21:13
categories: emr hadoop
---

At [MediaMath][mediamath] we're big users of Elastic MapReduce.  EMR's incredible flexibilty makes it a great fit for our analytics jobs.

An extermely important best practice for any analytics project is to ensure your local dev and test environments match your production environment as much as possible.  This eliminates the nasty surprise of launching a job that takes hours only to disocver that it fails late into the run due to some unmet dependency or config mistake.  Failing to invest time in the dev/test phase is a surefire way to blow big $$.

Lately I've been investigating some configuration settings you can make to your local Hadoop to bring it inline with what you'll find when you run a job on an EMR cluster.  This is especially important to us since we use S3 as a sort of centralized filesystem and EMR is designed to work wonderfully with S3.  Specificaly:

- Using s3:// URIs everwhere instead of s3n:// URIs
- Embedding AWS access keys
- Supporting transparent LZO compression


Installing Hadoop
-----------------

I run all my Hadoop jobs on my laptop using [Homebrew][homebrew].  Homebrew is a fantastic package manager for OS X that makes it a breeze to install general UNIX utilities as well as more complicated software packages (like Hadoop and Hive).

```bash
$> brew install hadoop
```

And you're good!


s3:// vs s3n:// URIs in HDFS
----------------------------

Ever wondered what the difference between an s3:// URI and an s3n:// URI is?  

Essentially up until December, 2010 S3 had a 5GB object size limit.  So, if you used the default S3 HDFS implementation (by specifying an s3n:// URI) you couldn't read/write files greater than 5GB.  That said, when you did read or write a file with HDFS there was a 1 to 1 correspondance with the object that got stored in S3.  

To process files larger than 5GB you had to use s3:// URIs in HDFS which actually chunked the file into multiple pieces behind the scenes before storing each piece as a separate object in S3.  So when accessing something via HDFS with an s3://bubket/object URI you might actually be downloading multiple "chunks" from S3.  [This page](http://wiki.apache.org/hadoop/AmazonS3) has some more info.

Nowadays the S3 limit is 5*TB*, so there isn't really a need to use s3:// URIs in HDFS anymore and indeed, in EMR, s3:// and s3n:// are both aliased to the same implementation (s3n).

Here's the relevant config:

```xml
<!-- Alias s3:// to s3n:// (EMR does this automatically) -->
<property>
    <name>fs.s3.impl</name>
    <value>org.apache.hadoop.fs.s3native.NativeS3FileSystem</value>
</property>
```


Embedding AWS Access Keys
---------------

If you want to run Hadoop jobs on your laptop but use data stored in S3, you'll need to ensure your credentials are stored in `mapred-site.xml`.  If you installed Hadoop via Homebrew, just edit `$(brew --prefix hadoop)/libexec/conf/mapred-site.xml`.  

NOTE: I set this up for both the s3n and s3 HDFS filesystem implementations, but since I only ever use the NativeS3FileSystem via s3:// URIs, the *s3* properties don't really matter (becase the S3FileSystem will never be used).

```xml
<property>
    <name>fs.s3.awsAccessKeyId</name>
    <value>XXXXXXXXXXXXXXXXXXXX</value>
</property>
<property>
    <name>fs.s3.awsSecretAccessKey</name>
    <value>XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX</value>
</property>
<property>
    <name>fs.s3n.awsAccessKeyId</name>
    <value>XXXXXXXXXXXXXXXXXXXX</value>
</property>
<property>
    <name>fs.s3n.awsSecretAccessKey</name>
    <value>XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX</value>
</property>
```


Hadoop and LZO compression
--------------------------

The log files we process in our analytics platform are compressed using LZO compression.  Luckily EMR can transparently decrompress these files, so no extra configuration is needed.  Our local Hadoop install, however, cannot.  Luckily Twitter has some open source code we can use [on GitHub][twitter-lzo].

Getting this to work requires a few steps:

- Compiling the native C code for LZO
- Compiling the java wrapper used by Hadoop
- Setting up the relevant classpath/library paths so Hadoop can find them
- Configuring Hadoop to use LZO compression when it finds a .lzo file

That's a lot to remember.  [Homebrew][homebrew] to the rescue!

```bash
$> brew tap themodernlife/misc
$> brew install hadoop-lzo
```

Basically this will use Homebrew to install lzo and maven so you can compile the Twitter Hadoop/LZO code for steps 1 and 2.  The compiled code will be installed into `$(brew --prefix hadoop-lzo)/libexec`.

Now we can configure Hadoop to use it.  First of all, the defualt `hadoop` executable actually *resets JAVA_LIBRARY_PATH* which means we can't get the native libraries onto the right path.  So, we have to edit `$(brew --prefix hadoop)/libexec/bin/hadoop` and comment out line #353

```bash
# setup 'java.library.path' for native-hadoop code if necessary
#JAVA_LIBRARY_PATH=''
```

Now we need to edit our `.bashrc`.  I'm also setting JAVA_HOME just to be safe.

```bash
# Java/Hadoop/Hive stuff
export JAVA_HOME=`/usr/libexec/java_home`
export HADOOP_CLASSPATH=/usr/local/Cellar/hadoop-lzo/0.4.19/libexec/hadoop-lzo-0.4.19.jar:$HADOOP_CLASSPATH
export JAVA_LIBRARY_PATH=/usr/local/Cellar/hadoop-lzo/0.4.19/libexec/native/Mac_OS_X-x86_64-64/lib
```

Lastly, let's edit `mapred-site.xml` again.

```xml
<!-- Transparent LZO compression + decompression (ensure you have the native libs installed) -->
<property>
    <name>io.compression.codecs</name>
    <value>org.apache.hadoop.io.compress.GzipCodec,org.apache.hadoop.io.compress.DefaultCodec,com.hadoop.compression.lzo.LzoCodec,com.hadoop.compression.lzo.LzopCodec,org.apache.hadoop.io.compress.BZip2Codec</value>
</property>
<property>
    <name>io.compression.codec.lzo.class</name>
    <value>com.hadoop.compression.lzo.LzoCodec</value>
</property>
```

Now your jobs should be able to transparently decompress LZO files, even when running on your laptop!

For more info on installing Hadoop LZO via Homebrew, checkout [the GitHub page for my Homebrew Tap][themodernlife-homebrew-misc]!


Investigating more settings
---------------------------

If you enable debuging when you spin up your EMR cluster, you can actually inspect the jobconf by downloading the file from S3.  This is a great way to see what other settings you may need to tweak to ensure that your jobs can be tested on your laptop before you send them off to EMR.





[mediamath]: http://mediamath.com
[homebrew]: http://brew.sh
[twitter-lzo]: https://github.com/twitter/hadoop-lz
[themodernlife-homebrew-misc]: https://github.com/themodernlife/homebrew-misc
