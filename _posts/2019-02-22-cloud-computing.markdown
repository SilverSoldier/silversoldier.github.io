---
layout: post
title:  "Cloud Computing - Course Summary"
description: A summary of the cloud computing course I am taking - for preserving my learnings till posterity
img:
date: 2019-02-22  +1730
---

## Edge computing
It is a recent paradigm in which the computation is done on distributed device nodes along the path to the cloud server.

Provide resources, data analysis etc. closer to the actual data collection devices (IoT devices).

### Topics related to this
+ IoT
+ Industry 4.0

### Advantages of Edge Computing
Firstly, lesser data needs to be sent to the cloud leading to lesser bandwidth requirement. Also, if data is not shared in the cloud, it can help combat privacy issues, since all information is not collected by a single entity.

Security wise, it seems to be worse, since the number of untrusted devices increases, increasing the attack surface.

## Why cloud computing?
Following main arguments for:
+ Pay-as-use
+ Remote service provision
+ Elastic resource provisioning aka auto-scale

## Hadoop and HDFS

### YARN (Yet Another Resource Negotiator)
YARN is a resource manager for the cluster.

#### Disadvantages
The main disadvantages of Hadoop are:
+ **Keyspace design**:
  A major design decision is the keyspace design and the functionality division among the mapper and the reducer.

  It is advantageous to have more functionality on the mapper side since it will reduce the network bandwidth and the sorting and shuffling of data to the reducer side.

  Also, computation on the mapper side can be made balanced (sampling the input and division among the mappers to get an equal load), whereas computation on the reducer side is heavily input dependant and thus cannot be predicted.

  However, we cannot have a fat mapper and a lean reducer since the mapper must get only a section of the input and there is only so much that can be done with a slice of the input.

+ **Multiple iterations KILL performance**:
  In order to do multiple map-reduce iterations, the output of the reducer (which is written to the HDFS), must be read again by the mapper as input.

  Reading from the disk is **always costly**.

  And now-a-days, everyone needs to do data analysis and most ML algorithms involve iterations over a dataset.

**Enter Spark**

## Apache Spark
Spark aims to solve the problems Hadoop suffers from, namely bad performance when multiple iterations on the same dataset occurs. It stores data in-memory, thereby processing it faster than Hadoop which writes the data to the disk between each stage of processing.

It uses a in-memory dataset called Reliable Distributed Dataset, which is NOT a data structure.

# References
## Other courses online
1. [eecs](https://people.eecs.berkeley.edu/~istoica/classes/cs294/15)
