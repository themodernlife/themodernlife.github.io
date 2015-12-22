---
layout: post
title:  Getting started with Scalding
date:   2015-12-20 08:15:13
categories: scala hadoop hdfs sclading
---

I've been using [Scalding](https://github.com/twitter/scalding) for the last few years and really love how simple it makes writing scalalbe data processing jobs.  I think many of the issues beginners have with Scalding relate to project setup.  I hope this post simplifies things for people so they can started with less hassle.

## Building your project with SBT

The official [getting started guide](https://github.com/twitter/scalding/wiki/Getting-Started) is pretty terse.  For our purposes, all you need to get started is a very simple [SBT](http://www.scala-sbt.org/0.13/tutorial/index.html) project.

```
simple-scalding-example/
|-- build.sbt
|-- project
|   |-- build.properties
|   `-- plugins.sbt
|-- src
|   `-- main
|       `-- scala
|           `-- WordCountJob.scala
`-- log4j.properties
```

The most important thing the setup of the `build.sbt` file.  Our example has minimal dependencies:

```scala
libraryDependencies ++= Seq(
  "com.twitter" %% "scalding-core" % "0.15.0",
  "org.apache.hadoop" % "hadoop-client" % "2.2.0" % "provided",
  "org.slf4j" % "slf4j-log4j12" % "1.7.13" % "provided"
)
```

The `hadoop-client` jar is included for compilation, unit testing and running locally on your laptop but will not be pacakged into the final "fat jar" assembly shipped out to the cluster.  Additionally, a logging framework is included so that we get good logs when debugging locally.  A simple `log4j.properties` file is included in the repo.

The whole job can be run locally using real Hadoop libraries without any additional software install:

```shell
$ sbt 'run net.themodernlife.WordCount --hdfs --input build.sbt --output target/output'
```

It's even possible to get things like LZO-compression to work with this setup.  The main win here is that there is a very simple,
repeatable development environment that can be used to test all aspects of the job locally *before* sending things off to the cluster.