---
layout: post
title:  "Remote Memory Access - a brief outline"
description: An introduction to Remote Memory Access (RMA).
img:
date: 2018-04-05  +0648
---

Parallel programming models are of 3 categories:
+ **Shared memory**: with implicit communication and explicit synchronization.
+ **Message passing**: with explicit communication and implicit synchronization.
+ **Remote memory access** and **Partioned global address space**: synchronization and communication are managed independantly.

## Two sided (Send/Recv) vs One sided (RDMA)
+ **Send/Recv**:
	+ Source issues 'Send' with location of data to be sent and target issues 'Recv' with location where data should be written.
	+ Each side has information necessary for communication, hence 'two-sided'.
+ **RDMA**:
	+ Source and target buffers are registered prior to communication.
	+ Registration of memory returns a handle to be used in RDMA operations.
	+ Only one side needs to have all the information necessary for the communication, hence 'one-sided'.

## Remote Memory Access (One-sided communication)
+ Remote Direct Memory Access is a direct memory access from the memory of one computer into that of another without involving either computer's OS.
+ RDMA model provides mechanisms that enable out of user-space operations to a previously registered user space memory buffer without OS involvement.

### Memory Regions and Process Groups
+ A section of memory in each process in a group is made accessible by all other processes within the group.
+ Process outside group cannot access memory of any other process.
+ Processes within the group can only access the exposed memory.
+ Memory not attached to the window cannot be accessed by any process (within or without the group).

+ Ex. In MPI, an MPI Process can declare a (contiguous) segment of memory as part of "window", allowing other processes to access this memory segment using one-sided operations such as PUT, GET, ACCUMULATE etc.


## References
+ [paper](http://www.mcs.anl.gov/papers/P4062-0413_1.pdf)
+ [cisco](https://blogs.cisco.com/performance/the-new-mpi-3-remote-memory-access-one-sided-interface)
+ [hpc wire](https://www.hpcwire.com/2006/08/18/a_critique_of_rdma-1.html)
