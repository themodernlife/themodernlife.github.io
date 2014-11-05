---
layout: post
title:  "Understanding compression in Hadoop and Spark"
date:   2014-09-24 11:50:13
categories: hadoop spark compression
---





Usually you'll see this:

```xml
<property>
    <name>io.compression.codecs</name>
    <value>org.apache.hadoop.io.compress.GzipCodec,org.apache.hadoop.io.compress.DefaultCodec,com.hadoop.compression.lzo.LzoCodec,com.hadoop.compression.lzo.LzopCodec,org.apache.hadoop.io.compress.BZip2Codec</value>
</property>
<property>
    <name>io.compression.codec.lzo.class</name>
    <value>com.hadoop.compression.lzo.LzoCodec</value>
</property>
```

`org.apache.hadoop.io.compress.CompressionCodecFactory` will create a bunch of lookup tables from
that entry.

First it will instantiate each Codec (via `ReflectionUtils`) and call some methods on each codec to get its metadata

```
codecs = {java.util.TreeMap@6277} size = 7
[0] = {java.util.TreeMap$Entry@6345}"2zb." -> {org.apache.hadoop.io.compress.BZip2Codec@6302}
[1] = {java.util.TreeMap$Entry@6347}"4zl." -> {org.apache.hadoop.io.compress.Lz4Codec@6299}
[2] = {java.util.TreeMap$Entry@6349}"etalfed." -> {org.apache.hadoop.io.compress.DefaultCodec@6296}
[3] = {java.util.TreeMap$Entry@6351}"etalfed_ozl." -> {com.hadoop.compression.lzo.LzoCodec@6305}
[4] = {java.util.TreeMap$Entry@6353}"ozl." -> {com.hadoop.compression.lzo.LzopCodec@6287}
[5] = {java.util.TreeMap$Entry@6355}"yppans." -> {org.apache.hadoop.io.compress.SnappyCodec@6290}
[6] = {java.util.TreeMap$Entry@6357}"zg." -> {org.apache.hadoop.io.compress.GzipCodec@6293}
codecsByName = {java.util.HashMap@6278} size = 16
[0] = {java.util.HashMap$Entry@6311}"gzip" -> {org.apache.hadoop.io.compress.GzipCodec@6293}
[1] = {java.util.HashMap$Entry@6313}"snappycodec" -> {org.apache.hadoop.io.compress.SnappyCodec@6290}
[2] = {java.util.HashMap$Entry@6315}"defaultcodec" -> {org.apache.hadoop.io.compress.DefaultCodec@6296}
[3] = {java.util.HashMap$Entry@6317}"lz4codec" -> {org.apache.hadoop.io.compress.Lz4Codec@6299}
[4] = {java.util.HashMap$Entry@6319}"snappy" -> {org.apache.hadoop.io.compress.SnappyCodec@6290}
[5] = {java.util.HashMap$Entry@6321}"bzip2" -> {org.apache.hadoop.io.compress.BZip2Codec@6302}
[6] = {java.util.HashMap$Entry@6323}"default" -> {org.apache.hadoop.io.compress.DefaultCodec@6296}
[7] = {java.util.HashMap$Entry@6325}"lz4" -> {org.apache.hadoop.io.compress.Lz4Codec@6299}
[8] = {java.util.HashMap$Entry@6327}"deflate" -> {org.apache.hadoop.io.compress.DeflateCodec@6308}
[9] = {java.util.HashMap$Entry@6329}"gzipcodec" -> {org.apache.hadoop.io.compress.GzipCodec@6293}
[10] = {java.util.HashMap$Entry@6331}"lzopcodec" -> {com.hadoop.compression.lzo.LzopCodec@6287}
[11] = {java.util.HashMap$Entry@6333}"lzop" -> {com.hadoop.compression.lzo.LzopCodec@6287}
[12] = {java.util.HashMap$Entry@6335}"bzip2codec" -> {org.apache.hadoop.io.compress.BZip2Codec@6302}
[13] = {java.util.HashMap$Entry@6337}"lzo" -> {com.hadoop.compression.lzo.LzoCodec@6305}
[14] = {java.util.HashMap$Entry@6339}"lzocodec" -> {com.hadoop.compression.lzo.LzoCodec@6305}
[15] = {java.util.HashMap$Entry@6341}"deflatecodec" -> {org.apache.hadoop.io.compress.DeflateCodec@6308}
codecsByClassName = {java.util.HashMap@6279} size = 8
[0] = {java.util.HashMap$Entry@6285}"com.hadoop.compression.lzo.LzopCodec" -> {com.hadoop.compression.lzo.LzopCodec@6287}
[1] = {java.util.HashMap$Entry@6288}"org.apache.hadoop.io.compress.SnappyCodec" -> {org.apache.hadoop.io.compress.SnappyCodec@6290}
[2] = {java.util.HashMap$Entry@6291}"org.apache.hadoop.io.compress.GzipCodec" -> {org.apache.hadoop.io.compress.GzipCodec@6293}
[3] = {java.util.HashMap$Entry@6294}"org.apache.hadoop.io.compress.DefaultCodec" -> {org.apache.hadoop.io.compress.DefaultCodec@6296}
[4] = {java.util.HashMap$Entry@6297}"org.apache.hadoop.io.compress.Lz4Codec" -> {org.apache.hadoop.io.compress.Lz4Codec@6299}
[5] = {java.util.HashMap$Entry@6300}"org.apache.hadoop.io.compress.BZip2Codec" -> {org.apache.hadoop.io.compress.BZip2Codec@6302}
[6] = {java.util.HashMap$Entry@6303}"com.hadoop.compression.lzo.LzoCodec" -> {com.hadoop.compression.lzo.LzoCodec@6305}
[7] = {java.util.HashMap$Entry@6306}"org.apache.hadoop.io.compress.DeflateCodec" -> {org.apache.hadoop.io.compress.DeflateCodec@6308}
```

What does this do?

```xml
<property>
    <name>io.compression.codec.lzo.class</name>
    <value>com.hadoop.compression.lzo.LzoCodec</value>
</property>
```

The only reference I could find is inside `org.apache.hadoop.io.file.tfile.Compression.java`.  When creating a TFile, you can choose a compression.  I'm speculating here, but I think the package name for LZO compression has changed a bit over the years, so the TFile implementation will try to resolve it dynamically.

Here's the relevant Enum definition from `org.apache.hadoop.io.file.tfile.Compression.java`

```java
LZO(TFile.COMPRESSION_LZO) {
  private transient boolean checked = false;
  private static final String defaultClazz =
      "org.apache.hadoop.io.compress.LzoCodec";
  private transient CompressionCodec codec = null;

  @Override
  public synchronized boolean isSupported() {
    if (!checked) {
      checked = true;
      String extClazz =
          (conf.get(CONF_LZO_CLASS) == null ? System
              .getProperty(CONF_LZO_CLASS) : null);
      String clazz = (extClazz != null) ? extClazz : defaultClazz;
      try {
        LOG.info("Trying to load Lzo codec class: " + clazz);
        codec =
            (CompressionCodec) ReflectionUtils.newInstance(Class
                .forName(clazz), conf);
      } catch (ClassNotFoundException e) {
        // that is okay
      }
    }
    return codec != null;
  }

  @Override
  CompressionCodec getCodec() throws IOException {
    if (!isSupported()) {
      throw new IOException(
          "LZO codec class not specified. Did you forget to set property "
              + CONF_LZO_CLASS + "?");
    }

    return codec;
  }

  @Override
  public synchronized InputStream createDecompressionStream(
      InputStream downStream, Decompressor decompressor,
      int downStreamBufferSize) throws IOException {
    if (!isSupported()) {
      throw new IOException(
          "LZO codec class not specified. Did you forget to set property "
              + CONF_LZO_CLASS + "?");
    }
    InputStream bis1 = null;
    if (downStreamBufferSize > 0) {
      bis1 = new BufferedInputStream(downStream, downStreamBufferSize);
    } else {
      bis1 = downStream;
    }
    conf.setInt("io.compression.codec.lzo.buffersize", 64 * 1024);
    CompressionInputStream cis =
        codec.createInputStream(bis1, decompressor);
    BufferedInputStream bis2 = new BufferedInputStream(cis, DATA_IBUF_SIZE);
    return bis2;
  }

  @Override
  public synchronized OutputStream createCompressionStream(
      OutputStream downStream, Compressor compressor,
      int downStreamBufferSize) throws IOException {
    if (!isSupported()) {
      throw new IOException(
          "LZO codec class not specified. Did you forget to set property "
              + CONF_LZO_CLASS + "?");
    }
    OutputStream bos1 = null;
    if (downStreamBufferSize > 0) {
      bos1 = new BufferedOutputStream(downStream, downStreamBufferSize);
    } else {
      bos1 = downStream;
    }
    conf.setInt("io.compression.codec.lzo.buffersize", 64 * 1024);
    CompressionOutputStream cos =
        codec.createOutputStream(bos1, compressor);
    BufferedOutputStream bos2 =
        new BufferedOutputStream(new FinishOnFlushCompressionStream(cos),
            DATA_OBUF_SIZE);
    return bos2;
  }
},
```
