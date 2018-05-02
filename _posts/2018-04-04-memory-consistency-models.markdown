---
layout: post
title:  "Memory Consistency Models"
description: A brief outline of different memory consistency models.
img:
date: 2018-04-04  +0648
---

Memory Consistency Models
-------------------------

### Memory Coherence vs Consistency
+ Memory Coherence:
	+ Reads and writes to same memory location.
	+ Due to optimization of duplicating data in multiple caches.
+ Memory Consistency: 
	+ Reads and writes to different memory locations.
	+ Due to optimization of reordering memory operations.

### What is shared memory system?

### What is memory model?
+ Defined at any interface between programmer and system.
+ Different aspects of memory models:
	+ **Programmers**: Increase parallelism while maintaining correctness.
	+ **Compiler designers**: Increase compiler optimizations.
	+ **Hardware/System designers**: Increase hardware optimizations.

### What is memory consistency model?
+ A **memory consistency model**, for a shared memory multiprocessor specifies how read-write memory operations of multiple processors occur.

### Terms useful
+ Memory operation
+ Read, write operation
+ **Operation conflict**: If operations are to same location (in memory) and atleast one operation is a write.
+ Atomic
+ **Program order**:
	+ Partial order on memory operations consistent with per-processor total order on memory operations of that processor.
	+ The order in which a single thread issues method calls.
+ Result, Equivalence of result

## List of memory consistency models

### Strong Memory Consistency Models
#### Strict Consistency
+ Each read gets value written by last write across all processors.
+ Follows a global time.
+ All writes are instantaneously visible to all processes.

#### Sequential Consistency 
+ **Lamport:** A multiprocessor system is sequentiallly consistent if the output of execution is the same as if the operations of all the processors were executed in some sequential order, and all the operations of each processor appear in the order specified by the program.

+ Each process issues memory requests in **program order**.
+ Memory requests from all processors to a particular memory location are seen in the same order by all processors.
+ Writes to all locations are seen in the same order by all processors (cache coherence).
+ All writes appear to be instantaneous (atomic).
+ Problems:
	+ Prevents H/W optimizations like store buffers, out-of-order execution ...
	+ Limits compiler optimizations like register allocation, partial redundancy elimination ...

### Weak Memory Consistency Models
#### Causal Consistency
+ Each process observes causally related operations in common causal order.
+ All processes **agree on order of *causally* related operations**.
+ Each process of the system observes the cause operation before the effect operation.
+ Write operations related by potential causality are seen by each process in common order. Concurrent writes can occur in **any** order and can be seen in different orders by different processes.

#### PRAM Consistency
+ Pipelined Random Access Memory / FIFO consistency
+ All processes see **memory writes** from a single process in the **order in which they were issued**.
+ There is no guarantee about the order in which different processors see writes, except that writes from a single source **arrive in the order they were issued** - like a pipeline.

#### Delta Consistency
+ An update will propogate through the system and all replicas will be consistent after a fixed time period $\delta$.

#### Eventual Consistency
+ If no new updates are made to a memory location, it will eventually reflect the last updated value.
+ Called **optimistic replication**.

#### Weak ordering/consistency
+ Memory operations are divided into data operations and synchronization operations.
+ All synchronization operations are executed in program order. Synchronization accesses are sequentially consistent.
+ All other operations can be seen by different processors in any order given,
+ Set of data operations in between 2 synchronization operations is the same.

#### Release consistency
+ Relaxation of weak consistency model, where entrance synchronization operation and exit synchronization operation are different.
+ The synchronization operations are broken down into *acquire* and *release* operations. All pending acquires must be done before a release.
