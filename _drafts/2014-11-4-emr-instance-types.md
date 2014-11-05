KILL AWS KEYS!

[hadoop@ip-10-165-19-146 ~]$ cat etc/hadoop/hdfs-site.xml
<?xml version="1.0"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>
  <property><name>dfs.datanode.max.xcievers</name><value>4096</value></property>
  <property><name>dfs.datanode.https.address</name><value>0.0.0.0:9402</value></property>
  <property><name>dfs.datanode.du.reserved</name><value>536870912</value></property>
  <property><name>dfs.namenode.handler.count</name><value>64</value></property>
  <property><name>io.file.buffer.size</name><value>65536</value></property>
  <property><name>dfs.block.size</name><value>134217728</value></property>
  <property><name>dfs.data.dir</name><value>/mnt/var/lib/hadoop/dfs,/mnt1/var/lib/hadoop/dfs,/mnt2/var/lib/hadoop/dfs,/mnt3/var/lib/hadoop/dfs</value></property>
  <property><name>dfs.secondary.http.address</name><value>0.0.0.0:9104</value></property>
  <property><name>dfs.replication</name><value>1</value></property>
  <property><name>dfs.http.address</name><value>0.0.0.0:9101</value></property>
  <property><name>dfs.https.address</name><value>0.0.0.0:9202</value></property>
  <property><name>dfs.datanode.http.address</name><value>0.0.0.0:9102</value></property>
  <property><name>dfs.datanode.address</name><value>0.0.0.0:9200</value></property>
  <property><name>dfs.name.dir</name><value>/mnt/var/lib/hadoop/dfs-name,/mnt1/var/lib/hadoop/dfs-name,/mnt2/var/lib/hadoop/dfs-name,/mnt3/var/lib/hadoop/dfs-name</value></property>
  <property><name>dfs.webhdfs.enabled</name><value>true</value></property>
  <property><name>dfs.datanode.ipc.address</name><value>0.0.0.0:9201</value></property>
</configuration>
[hadoop@ip-10-165-19-146 ~]$ mount
proc on /proc type proc (rw,relatime)
sysfs on /sys type sysfs (rw,relatime)
devtmpfs on /dev type devtmpfs (rw,relatime,size=7685316k,nr_inodes=1921329,mode=755)
devpts on /dev/pts type devpts (rw,relatime,gid=5,mode=620,ptmxmode=000)
tmpfs on /dev/shm type tmpfs (rw,relatime)
/dev/xvda on / type ext4 (rw,noatime,data=ordered)
none on /proc/sys/fs/binfmt_misc type binfmt_misc (rw,relatime)
/dev/xvdb on /mnt type xfs (rw,relatime,attr2,inode64,noquota)
/dev/xvdc on /mnt1 type xfs (rw,relatime,attr2,inode64,noquota)
/dev/xvdd on /mnt2 type xfs (rw,relatime,attr2,inode64,noquota)
/dev/xvde on /mnt3 type xfs (rw,relatime,attr2,inode64,noquota)
/dev/xvdb on /media/ephemeral0 type xfs (rw,relatime,attr2,inode64,noquota)
[hadoop@ip-10-165-19-146 ~]$ df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/xvda       9.8G  5.0G  4.3G  54% /
devtmpfs        7.4G   28K  7.4G   1% /dev
tmpfs           7.4G     0  7.4G   0% /dev/shm
/dev/xvdb       420G   38M  420G   1% /mnt
/dev/xvdc       420G   34M  420G   1% /mnt1
/dev/xvdd       420G   34M  420G   1% /mnt2
/dev/xvde       420G   34M  420G   1% /mnt3
[hadoop@ip-10-165-19-146 ~]$ cat conf/hadoop
hadoopDefaultMetricsList    hadoop-env.cmd              hadoop-env.sh               hadoop-metrics2.properties  hadoop-metrics.properties   hadoop-policy.xml
[hadoop@ip-10-165-19-146 ~]$ cat etc/hadoop/core-site.xml
<?xml version="1.0"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>
  <property><name>hadoop.proxyuser.hadoop.groups</name><value>*</value></property>
  <property><name>fs.s3n.impl</name><value>com.amazon.ws.emr.hadoop.fs.EmrFileSystem</value></property>
  <property><name>fs.s3.impl</name><value>com.amazon.ws.emr.hadoop.fs.EmrFileSystem</value></property>
  <property><name>fs.default.name</name><value>hdfs://10.165.19.146:9000</value></property>
  <property><name>hadoop.tmp.dir</name><value>/mnt/var/lib/hadoop/tmp</value></property>
  <property><name>fs.s3n.awsSecretAccessKey</name><value>XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX</value></property>
  <property><name>fs.s3n.awsAccessKeyId</name><value>XXXXXXXXXXXXXXXXXXXX</value></property>
  <property><name>fs.s3.buffer.dir</name><value>/mnt/var/lib/hadoop/s3,/mnt1/var/lib/hadoop/s3,/mnt2/var/lib/hadoop/s3,/mnt3/var/lib/hadoop/s3</value></property>
  <property><name>io.compression.codecs</name><value>org.apache.hadoop.io.compress.GzipCodec,org.apache.hadoop.io.compress.DefaultCodec,com.hadoop.compression.lzo.LzoCodec,com.hadoop.compression.lzo.LzopCodec,org.apache.hadoop.io.compress.BZip2Codec,org.apache.hadoop.io.compress.SnappyCodec</value></property>
  <property><name>fs.s3bfs.impl</name><value>org.apache.hadoop.fs.s3.S3FileSystem</value></property>
  <property><name>fs.s3n.endpoint</name><value>s3.amazonaws.com</value></property>
  <property><name>hadoop.metrics.defaultFile</name><value>/home/hadoop/conf/hadoopDefaultMetricsList</value></property>
  <property><name>fs.s3.awsSecretAccessKey</name><value>XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX</value></property>
  <property><name>hadoop.proxyuser.hadoop.hosts</name><value>*</value></property>
  <property><name>fs.s3bfs.awsAccessKeyId</name><value>XXXXXXXXXXXXXXXXXXXX</value></property>
  <property><name>fs.s3bfs.awsSecretAccessKey</name><value>XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX</value></property>
  <property><name>hadoop.metrics.list</name><value>TotalLoad,CapacityTotalGB,UnderReplicatedBlocks,CapacityRemainingGB,PendingDeletionBlocks,PendingReplicationBlocks,CorruptBlocks,CapacityUsedGB,numLiveDataNodes,numDeadDataNodes,MissingBlocks</value></property>
  <property><name>io.compression.codec.lzo.class</name><value>com.hadoop.compression.lzo.LzoCodec</value></property>
  <property><name>fs.s3.awsAccessKeyId</name><value>XXXXXXXXXXXXXXXXXXXX</value></property>
</configuration>
[hadoop@ip-10-165-19-146 ~]$ cat etc/hadoop/mapred-site.xml
<?xml version="1.0"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>
  <property><name>mapred.output.committer.class</name><value>org.apache.hadoop.mapred.DirectFileOutputCommitter</value></property>
  <property><name>mapreduce.task.io.sort.mb</name><value>200</value></property>
  <property><name>mapreduce.reduce.cpu.vcores</name><value>1</value></property>
  <property><name>mapred.output.direct.EmrFileSystem</name><value>true</value></property>
  <property><name>mapreduce.task.io.sort.factor</name><value>48</value></property>
  <property><name>mapreduce.jobhistory.address</name><value>ip-10-165-19-146.ec2.internal:10020</value></property>
  <property><name>mapreduce.job.userlog.retain.hours</name><value>48</value></property>
  <property><name>mapreduce.framework.name</name><value>yarn</value></property>
  <property><name>mapreduce.reduce.java.opts</name><value>-Xmx768m</value></property>
  <property><name>mapreduce.map.java.opts</name><value>-Xmx512m</value></property>
  <property><name>mapreduce.reduce.shuffle.parallelcopies</name><value>20</value></property>
  <property><name>mapreduce.map.memory.mb</name><value>768</value></property>
  <property><name>hadoop.job.history.user.location</name><value>none</value></property>
  <property><name>mapreduce.reduce.speculative</name><value>true</value></property>
  <property><name>mapreduce.application.classpath</name><value>$HADOOP_MAPRED_HOME/share/hadoop/mapreduce/*,$HADOOP_MAPRED_HOME/share/hadoop/mapreduce/lib/*,/usr/share/aws/emr/emrfs/lib/*,/usr/share/aws/emr/lib/*</value></property>
  <property><name>mapreduce.map.cpu.vcores</name><value>1</value></property>
  <property><name>mapreduce.map.speculative</name><value>true</value></property>
  <property><name>mapred.output.direct.NativeS3FileSystem</name><value>true</value></property>
  <property><name>mapreduce.job.maps</name><value>2</value></property>
  <property><name>mapreduce.reduce.memory.mb</name><value>1024</value></property>
  <property><name>mapreduce.map.output.compress</name><value>true</value></property>
  <property><name>mapred.local.dir</name><value>/mnt/var/lib/hadoop/mapred,/mnt1/var/lib/hadoop/mapred,/mnt2/var/lib/hadoop/mapred,/mnt3/var/lib/hadoop/mapred</value></property>
  <property><name>mapreduce.job.reduces</name><value>1</value></property>
  <property><name>mapreduce.jobhistory.webapp.address</name><value>ip-10-165-19-146.ec2.internal:19888</value></property>
  <property><name>yarn.app.mapreduce.am.job.task.listener.thread-count</name><value>60</value></property>
  <property><name>yarn.app.mapreduce.am.resource.mb</name><value>1024</value></property>
  <property><name>mapreduce.job.jvm.numtasks</name><value>20</value></property>
  <property><name>mapreduce.map.output.compress.codec</name><value>org.apache.hadoop.io.compress.SnappyCodec</value></property>
</configuration>
