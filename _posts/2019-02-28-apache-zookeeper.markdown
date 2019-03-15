---
layout: post
title:  "Understanding Apache Zookeeper and Zab"
description: My understanding of Apache Zookeeper and Zab
img:
date: 2019-02-28  +1830
---

# Zookeeper

Understanding zookeeper by reading and understanding the following paper:

1. Hunt, Patrick, et al. "ZooKeeper: Wait-free Coordination for Internet-scale Systems." USENIX annual technical conference. Vol. 8. No. 9. 2010.

## 1.What is the research problem the paper attempts to address?
<!-- what is the niche of the paper? -->

Distributed systems need to coordinate among themselves for multiple purposes. These include but are not limited to leader election (which nodes perform which duties), distributed locks, distributed queues, group membership (which nodes are a part of the system), dynamic configuration etc.

NOTE: Even though distributed locks etc. are synchronous primitives, they can be implemented using asynchronous primitives.

All distributed systems must thus have an implementation of each of the coordination protocols that they require.

And why is this a problem ???

+ Duplicacy of code - assuming that all coordination protocols have some core in common.
+ Each of these coordination steps is actually a possible bottleneck and must be thus be extremely fast and optimized. Duplicacy of optimization.
+ Each distributed system might have different requirements for the coordination protocol depending on fault tolerance requirements, type of traffic/requests, consistency requirements, waiting time requirements etc. We cannot have a one-size-fits-all implementation of the higher level coordination protocol that can be used for all distributed system problems.

## 2.What are the claimed contributions of the paper?
<!-- what is innovative about this paper -->

They claim to create a skeleton coordination protocol core (**coordination kernel**) with the following properties:

1. It is wait-free / non-blocking / asynchronous:
	<!-- + TODO: Exactly how does it differ from a synchronous protocol? Is it more difficult to implement/more useful??? -->
2. It ensures a FIFO ordering of requests from a particular client:
	+ Requests sent by the client are executed in the order in which they are issued.
	+ Clients can fire a series of operations and be guaranteed that they are processed in the order they were given
3. It guarantees that writes are linearly serializable.
	+ This means that all systems agree on the same order of write operations.
4. It is fast, highly available and can process requests with low latency and high througput.
5. Can be used to create higher level coordination protocols using **coordination recipes**.

## 3.How do the authors substantiate their claims?
<!-- what makes the claims scientific -->

### ZooKeeper Architecture
1. ZooKeeper Data Model
	+ ZooKeeper provides a file system abstraction - a hierarchical namespace with data nodes (called znodes).
	+ The znodes can have data and children.
	+ The znodes service read and write requests from the client.
	+ Znodes can be ephemeral or regular.
	+ Sequential flag appends a znode with an increasing counter.
	+ Data is completely **replicated** across the different systems (no sharding).
2. ZooKeeper Sessions
	+ Clients can create sessions for sending requests to the znodes.
	+ Ephemeral znodes last only for a session.
3. ZooKeeper Watches
	+ The clients can request the znode to asynchronously *watch* a file for changes and give a one-time signal in case the file is modified.
	+ NOTE: Watch only triggers only once after the change.
4. Consistency Guarantees
	+ Sequential consistency: Requests sent by client are executed in the order they are sent.

### ZooKeeper Client API
1. Create
2. Delete
3. Exist
4. getData
5. setData
6. getChildren
7. sync

### Guarantees on properties of ZooKeeper
+ Data is replicated across all servers for high availability.
+ The data is stored in-memory giving low latency.
+ Clients connect to exactly one server.
+ Read requests are serviced by the connected server.
+ Write requests are forwarded to **leader** which executes the request and forwards the new state information to all **followers** using **Zookeeper Atomic Broadcast**.
+ It is very fast because it treats reads and writes separately.
	+ Reads are processed very fast because it is done locally.
	+ Writes are broadcasted to all systems using Zab.
	+ For a high read:write ratio, it will thus perform well.

### How to build higher constructs using ZooKeeper
All the higher constructs use the znodes (ephemeral and regular) and the watch functionality.

1. Configuration
	1. There exists a znode called "Config" with the configuration details.
	2. Processes being created read from Config.
	3. They also `watch` for changes in the configuration.
2. Group membership
	1. There exists a znode called "Group".
	2. When a member joins the group (starts), it creates a child node of Group with name as unique process name or with sequential flag set and with data as the process data.
	3. To check membership, others can do `getChildren(Group)`.
	4. Other processes can monitor changes in this process configuration by `watch`ing.
	4. To exit the group, simply `delete` the created child node.
3. Barrier
	1. Allows a group of processes to synchronize at the beginning and end of a computation.
	2. There exists a znode called "Barrier"
	3. To enter the barrier, each client creates a child node of Barrier.
	4. Use `getChildren(Barrier)` to check if enough processes have entered the Barrier.
	5. If yes, continue execution of code - barrier check passed.
	6. If no, set a `watch` on Barrier and on triggering goto step 4.
	7. For exiting, `delete` the created child node and `watch` the number of children.
3. Locks
	1. There exists a znode called "Lock"
	2. Create a child nodes (ephemeral) of Lock using sequential flags (using `create`).
	3. Check if you are the lowest numbered znode, if yes -> you get the lock (by calling `getChildren(Lock)`).
	4. Else, request a watch on the znode closest (and lower) than you.
	5. If watch triggers, goto 3 - could be just because the znode was deleted for various reasons without actually getting the lock.
	6. SPECIALITY: Prevents all clients from requesting for lock after someone unlocks by introducing a sequence of locks - apparently called the **herd effect**.
	7. Only try to obtain lock, after the client before you finishes. If they drop out, then since ephemeral node, the znode will be deleted, so no issues of hanging.
4. Queues
5. Leader Election
6. 2 Phase Commit
	+ It is a atomic commitment protocol (basically a consensus protocol) where all nodes agree on whether a transaction should be committed or aborted.

# Zookeeper Atomic Broadcast

Zookeeper requires an atomic broadcast algorithm to ensure FIFO ordering of client write requests while maintaining consistency.

It is to Zookeeper what Paxos is to Chubby (Is this correct??? - Yes, verified. Also it is what raft is to etcd).

An atomic broadcast protocol ensures total order consistency, wherein the entire system appears to be a single machine (instead of being distributed).

The protocol should satisfy the following conditions:

1. **Fault tolerance and efficient recovery:** Protocol should work even if the primary crashes.
2. **Reliable Delivery:** If a message is delivered by one process, all processes should deliver it.
2. **Total ordering:**
	+ If a message *m1* is delivered before message *m2* in one server, it must be delivered in the same order in other servers.
	+ *m1* can be delivered either before or after *m2*.

3. **Multiple outstanding transactions:** Multiple Zookeeper client operations can be outstanding (proposed but not yet delivered) and they must be executed in FIFO for better performance.
4. **Highly performant:**
	+ All Zookeeper applications will require ZAB, so it must have high througput and low latency.
5. **Primary order:**
	1. Local primary order
		If a primary broadcasts *m1* before *m2*, other processes must commit *m1* before *m2*
	2. Global primary order
		If a primary broadcasts *m1* and fails and another primary is elected which broadcasts *m2*. A process that commits both *m1* and *m2* must commit *m1* before *m2*.
	+ **Differs from Causal Order**: HOW???
6. **Consistency of processes:**
	1. Integrity: If a process commits a transaction *m1*, then some process must have broadcast it.
	2. Agreement: All processes commit 

### Similar yet dissimilar to
+ Paxos:
	+ Paxos employs consensus to serialize operations at a leader and apply the operations at each replica in this exact serialized order dictated by the leader.


### Transaction
+ Transaction identifier is zxid.
+ Transaction is \<v, z\>, where v is transaction value and z is transaction id.
+ Transaction id is \<e, c\>, where e is epoch and c is counter.
+ *epoch* identifies a leader phase, *counter* signal transaction number within a epoch.
	+ **epoch helps in global order, counter helps in local order.**
	+ Together \<e,c\> establish a total order.

### Phases of the algorithm

**Phase 0: Leader Election**

  + There is a leader oracle which tells each process the identity of the prospective leader.
	
  + Fast Leader Election: The process having the most up-to-date state is elected as the leader, to minimize state changes necessary.
  + Processes are in ELECTION stage.

**Phase 1: Discovery** across the cluster

  + In case FLE is not followed, the prospective leader must get up-to-date on the transactions committed by a quorum of followers.
  + Followers communicate with the prospective leader and inform it of its last transaction.
  + Leader establishes a new epoch by sending NEWEPOCH message. Processes acknowledge this.
  + Leader must receive acknowledgement from a quorum (majority) of the alive processes.
  + Leader is still in-vote stage not yet elected. 
  + Followers promise to not go back to previous epoch.
  + Leader then receives every follower's history of transactions and chooses the most up-to-date one as its history (latest epoch, latest counter).

**Phase 2: Synchronization** of transactions

  + The leader broadcasts a NEWLEADER message with new epoch number and its up-to-date history of transactions.
  + The followers synchronize by delivering all transactions in leader's history.
  + The leader is confirmed.

**Phase 3: Broadcast** of relevant information

  + If all goes well, this is the stage most nodes/processes must be in.
  + Leader broadcasts `ready(e)` to signal that Zab layer is established.
  + If a client issues at write request, it will broadcast to leader.
  + Proposing and committing transactions
	+ Leader orders and proposes transactions to followers using incrementing counter of same epoch.
	+ Followers send ACK message after saving transaction to local storage.
	+ If quorum of followers acknowledge, leader broadcasts a COMMIT message.
	+ Followers now commit the transaction (and all uncommitted transactions with zxid lesser than this) in ORDER.
	  + Is this 2-phase commit ????
		  + Yes, verified in Ref#3
		  + It is apparently a simplified version, which assumes no failures.

### Useful information
1. Zab uses TCP to ensure a FIFO channel between the servers which guarantees delivery and order of the messages.
2. Deliver is used in the meaning of commit.
3. Server and process are used interchangably to denote a Zookeeper running program.
4. Transaction and message are used interchangably to signify the communication between server and client.
5. Quorum (as used by Zookeeper) signifies a majority of processes.
	+ Thus, we assume **2f + 1** total processes, with minimum **f** processes alive.
	+ TODO: Could quorum mean anything else as well???

# References
## ZooKeeper
1. The paper
2. Morning Paper's summary of the paper
3. [Tom Wheeler's Slides](http://www.tomwheeler.com/publications/2012/zookeeper_tomwheeler_ll-20120607.pdf)
4. [Guide to creating higher level constructs using ZooKeeper](https://zookeeper.apache.org/doc/r3.1.2/recipes.html)
## ZAB
1. [Morning Paper on Zab](https://blog.acolyer.org/2015/03/09/zab-high-performance-broadcast-for-primary-backup-systems/)
2. [Morning Paper on Zab: Theory and Practice](https://blog.acolyer.org/2015/03/10/zookeepers-atomic-broadcast-protocol-theory-and-practice/)
3. [Official Zookeeper Website](https://zookeeper.apache.org/doc/r3.4.13/zookeeperInternals.html)
4. [Some course's slides](https://cnitarot.github.io/courses/ds_Fall_2016/505_chubby_zook.pdf)
5. [Zab vs Paxos](https://cwiki.apache.org/confluence/display/ZOOKEEPER/Zab+vs.+Paxos)
