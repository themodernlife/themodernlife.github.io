---
layout: post
title:  "Making Your Local Hadoop more like AWS Elastic MapReduce"
date:   2014-01-02 11:21:13
categories: emr hadoop
---

The more [Elastic Mapreduce](http://aws.amazon.com/elasticmapreduce/) jobs we launch into production, the more important it becomes to have detailed operational monitoring.  EMR jobs create a bunch of useful stats in [CloudWatch](http://aws.amazon.com/cloudwatch/), but I what I have really been looking for is alerting when 