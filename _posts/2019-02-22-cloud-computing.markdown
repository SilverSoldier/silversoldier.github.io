---
layout: post
title:  "Cloud Computing - Course Summary"
description: A summary of the cloud computing course I am taking - for preserving my learnings till posterity
img:
date: 2019-02-22  +1730
---

The entire article was written over the duration of a semester, so the writing is not consistent.

## Edge computing
It is a recent paradigm in which the computation is done on distributed device nodes along the path to the cloud server.

Provide resources, data analysis etc. closer to the actual data collection devices (IoT devices).

### Topics related to this
+ IoT
+ Industry 4.0

### Advantages of Edge Computing
Firstly, lesser data needs to be sent to the cloud leading to lesser bandwidth requirement. Also, if data is not shared in the cloud, it can help combat privacy issues, since all information is not collected by a single entity.

Security wise, it seems to be worse, since the number of untrusted devices increases, increasing the attack surface.

In case the application requires low latency, it can be beneficial.

## Cloud Computing?
Following are the main arguments for cloud computing
+ Remote service provision

Some important features of cloud computing are:
+ Large data centers running on commodity hardware (can fail easily).
+ Virtualization of computation and storage
+ Economics of scale: Cheaper than provisioning own-hardware
+ Utility Computing: Pay-for-use business model.
+ Elastic resource provisioning aka auto-scale: Scale up or down based on consumer demand
+ Accessible and highly available: Available over the network anywhere

### Types of Cloud Computing
1. Infrastructure-as-a-service
**Hyperviser** virtualizes a platform's OS => enables multiple OS' to share the same physical machine.
2. Platform-as-a-service
Abstracts infrastructure, OS and middleware, allows dynamic provisioning.
3. Software-as-a-service

## Hadoop MapReduce
Data parallel programming model for clusters

### Advantages
+ Provides fault tolerance (due to HDFS)
+ Scalable
+ Infrastructure abstracted out to developer

### Fault Tolerance
1. Task failure:
	Run it on another node. Both map and reduce will not have a problem.
2. Node failure:
	Distribute all tasks to other nodes.
3. Straggling task:
	Speculatively execute task on another node. Use whichever result is obtained first.

### Components
1. Job Tracker (Master):
	+ Accepts MR jobs
	+ Assigns tasks to workers
	+ Monitors task progress
	+ Handles failures by re-executing failed tasks
2. Task Tracker (Slave):
	+ Run Map and Reduce tasks as directed by master
	+ Manage intermediate outputs

### Tasks
1. Input Reader: Divides input into appropriate chunks for each mapper
2. Map Function: Map input to <key, value> pair - done in parallel.
3. Partition Function: Find the reducer given the key and number of reducers
4. Compare Function: Input for a reducer is pulled from intermediate outputs and sorted according to this function on values.
5. Reduce Function: Takes intermediate output values and outputs a <key, value> pair.


### HDFS
Supports write-once-read-many semantics => clients can only append to existing files

1. Name Node:
	+ Manages metadata of HDFS
	+ Controls replication of blocks to Data Node
		+ Listens for heartbeats from Data Nodes
		+ Load balancing
	+ Single Point of Failure
2. Data Node:
	+ Store data in blocks
	+ Clients write/read data in blocks in data nodes

### YARN (Yet Another Resource Negotiator)
YARN is a resource manager for the cluster.

**Resource Manager**

### Disadvantages
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

**Spark offers**:
+ Lazy Computations
+ In-memory data caching
+ Efficient Pipelining
+ Fault Tolerance: rebuild on failure

### RDD: Resilient Distributed Dataset
**Fault tolerant read only** collection of objects partitioned across many machines, that can be operated on in parallel which can be cached for efficiency.

**Operations**:

+ Transformations
	+ Create a new RDD from existing using functions like map, filter, distinct, union, ..
+ Actions
	+ Return a value after running a computation like collect, count, first, takeSample ...

### Fault Tolerance


## Kubernetes

## Apache Kafka
**Distributed publish-subscribe** system based on concept of **distributed commit log**. It is a **horizontally scalable messaging system**.

### Terminology
1. **Record**: Message or event. Unit of data in Kafka.
1. **Topic**: Group of records
2. **Broker**: Receives messages, assigns offset and commits messages to disk.
3. **Producer**: Write messages of a topic to a broker
	+ Clients that publish records to a Kafka cluster.
4. **Consumer**: Read from brokers for a specific topic
	+ Clients that consume records from a Kafka cluster.
	+ Read messages in the order they were produced.
	+ Must keep actively polling to check if new message has been produced.
5. **Partition**: Subgroup of topic

## Architecture
Kafka stores key-value messages that arrive from multiple producers. The data from various producers is partitioned into many *partitions* within different *topics*.

### Partitions
Partitions store records as an ordered immutable sequence. A partition is a single log stored durably on disk.

Records are added to partitions in an append-only fashion.  There can be replicas of partitions.

Messages do not have unique ids, they are uniquely determined by their offset in a partition  making it **stateless**. Producers do not store state regarding consumer (which message it has read etc.)

## Applications
Kafka is used typically for building real-time streaming applications or data pipelines which receive data stream from a system and transform or forward it to other applications.

## NoSQL Databases
### CAP Theorem
1. Consistency: All replicas have same data, client has same view of data.
2. Availability: System remains operational even if individual units fails.
3. Partition Tolerance: System remains operational even on communication malfunction (physical system partitions)

### Vs RDBMS
+ No support for join etc.
+ Relaxation of ACID properties (BASE properties).

### Benefits of NoSQL
+ Elastic Scaling: Due to sharding and replication, scales well
+ Management: Lesser Management, Simpler Data Models
+ Big Data: Can store a lot of data
+ Flexible Data Models: Changing schema in RDBMS requires a lot of change
+ Economics: Can have commodity servers due to fault tolerance

### Disadvantages of NOSQL
+ DBMS databases are mature
+ Data consistency, transactions ...

### Key Features
1. Scale horizontally simple operations
2. Replicate/distribute data over multiple servers
3. Simple Call Interface (unlike SQL)
4. Weaker concurrency model than ACID.
5. Efficient use of distributed indexes and RAM
6. Flexible Schema

### Key Value Stores
Possible features:

+ Versioning: Based on Vector Clocks
+ Partitioning and Replication
+ Consistent Hashing (DHT)
+ Fetch entire value - DB does not understand it

ex. Voldermort, Riak

### Document Databases
Documents stored in JSON format.

+ Value is usually structured

ex. MongoDB, CouchDB

### Column Family Databases
Each key is associated with multiple columns. 

+ Sparse distributed multi-dimensional sorted map

Ex. BigTable, Cassandra, HBase

# References
## Other courses online
1. [eecs](https://people.eecs.berkeley.edu/~istoica/classes/cs294/15)
