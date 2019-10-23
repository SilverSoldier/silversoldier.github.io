---
layout: post
title:  "Distributed Systems - summary of course on Blockchain and Distributed Ledgers"
description: A summary of the course contents on B&DL whose first half is basically an intro to distributed systems
img:
date: 2019-04-10  +1000
---

## Time, Clocks and the Ordering of Events in a Distributed System
### System and Problem Setup
Time in distributed systems is a fickle concept. There is no global, single-source-of-truth clock since the clock ticks are measured using system clock cycles and due to difference in clock speeds of different systems, there is eventually a skew that is introduced.

Thus, there is no global time.

A system is composed of multiple processes and a process is composed of many, many events. The whole concept of time is because we would like to know the ordering between the events. 

Why we would like to know that?

For instance, resource contention. If 2 processes request for an event, we would like to give to the first requester (for fairness), and would thus want to know the ordering between the request events.

### Logical Clocks
Logical clocks are the simplest clocks where each process is associated with a number/counter. Events are associated with the clock value at the time of occurring.

*Clock Condition*: If an event *a* happens before event *b* then C(a) is less than C(b)

#### Total ordering of events
Using logical clocks it is possible to have a total ordering of events. We use the partial order defined before and in case of ties, have an arbitrary tie-breaker using the process ids.

Total ordering of events is useful in applications like the distributed mutual exclusion problem.

#### Problems
Ordering preserves the happens-before relation but does not always imply happens-before. Cannot guarantee causality from logical clock.

### Vector Clocks
Each process maintains a vector of the clock counters of the other processes, according to it.

After receiving a message, it will increment its counter by one and change other counters to reflect maximum value.

#### Problems
1. Communication cost: An entire vector of clock times needs to be communicated with every message.

## 2 Phase commit
The protocol consists of a co-ordinator node. All client requests are sent to the co-ordinator.

Co-ordinator achieves consensus among the replicas in 2 phases:
1. Co-ordinator sends PREPARE message. Replicas respond with ACCEPT if they accept, else ABORT.
2. If co-ordinator receives an ABORT message, it sends ABORT to all processes. Else, it will send COMMIT value.

### Failures
1. Co-ordinator fails after initiating phase 1.

  Some replicas might be waiting for a response from co-ordinator for phase 2 and might wait indefinitely. This can be solved with a timeout.

  In case a node does not receive a response from a co-ordinator after phase 1 within the timeout, it will start the election process to elect a new co-ordinator process.

2. Participant fails after receiving message for phase 1.

  Co-ordinator does not know whether participant will ACCEPT or ABORT and thus cannot proceed.

3. Co-ordinator fails in phase 2 after sending a COMMIT message to a participant which also failed.

  Replicas which sent ACCEPT in previous phase are now blocked (with mutexes locked) and cannot COMMIT or ABORT since they don't know what A's response message was.


2PC is a blocking protocol. It is safe but not live.

2PC suffers from allowing nodes to irreversibly commit an outcome before ensuring that the others know the outcome.

## 3 Phase Commit

Augment 2PC to not block on node failures.

1. Co-ordinator sends PREPARE messages, replicas respond with ACCEPT or ABORT.
2. Co-ordinator sends PRE-COMMIT message if all replicas responded with an ACCEPT.
3. If co-ordinator receives ACCEPT for all PRE-COMMIT messages, then sends an ACCEPT message.

### Failures
1. Co-ordinator or participant during commit phase

  Recovery co-ordinator takes over. If atleast one process received a PRE-COMMIT message, then they can all commit. After recovery, the failed participant will contact the replicas to become up-to-date.

  If no node received a  PRE-COMMIT message, they can all ABORT safely, since the failed node could have only received a PRE-COMMIT message (not a COMMIT message, since they have not ACCEPT'ed the PRE-COMMIT messages) and would have thus not COMMIT'ted the transaction.

2. Due to timeouts and network partitions 

  Ex: [Link](https://roxanageambasu.github.io/ds-class//assets/lectures/lecture17.pdf)

## FLP Theorem: Impossibility of Distributed Consensus with One Faulty Process
### Consensus Problem
Consensus is the problem of getting a distributed set of processes to agree on one value.

1. Agreement: No 2 processes can commit different values.
2. Validity (Non-triviality): The committed value must be proposed by atleast one node.
3. Termination: Nodes commit eventually.

### System Setting
In a synchronous system, there is a time bound on the response of the processes and thus, consensus can be solved in a synchronous system. If a process stops/faults, it can be easily identified and is no longer considered in the consensus protocol.

Synchronous systems detect failures by waiting for the RTT of the response and in case it is not received, mark the process as failed.

In an asynchronous system, failure detection is not easy. There is no time bound on the response time of processes. Thus, a failed process cannot be detected as there is no difference between an unresponsive/failed process and a process that is simply taking a long time to respond.

Thus, there is no protocol which solves the distributed consensus problem in an asynchronous system.

### Theorem
It is impossible for a set of processes in an asynchronous system to agree to a single binary value even if only a single process is subject to an unannounced failure.

There is no completely asynchronous consensus protocol that can handle even a single process' unannounced failure.

## CAP Theorem
The FLP system model assumes reliable communication, asynchronous system and crash failures.

CAP considers unreliable channels in its fault model and holds for both synchronous and asynchronous systems.

## 2 Generals Problem
There is no deterministic algorithm for reaching consensus in a model where arbitrary number of messages can be lost undetectably to the sender, in both synchronous and asynchronous systems.

## Paxos
Completely safe and largely live agreement protocol. See [my writeup on Paxos](http://gkavya.in/paxos/) for my summary of the paper. But definitely see [Prof. Murat's blogspot] (http://muratbuffalo.blogspot.com/2010/10/paxos-taught.html) to understand Paxos.

## Raft
Consensus algorithm which is acclaimed for being easily able to understand (which Paxos was not).

It breaks the problem into 3 components:
1. **Leader Election**
  All client requests are sent to the leader which is responsible for maintaining an ordered replicated log.
  
  Leader election is necessary when a follower has not received a heart-beat message from the leader for a duration. Followers upgrade themselves to candidates and send *requestVote()* RPCs.

  Every leader is associated with a term (one-one relationship) and all messages during that term are marked with the term number.

2. **Replicated Log**
  Leader writes log entries and instructs followers to update their logs using the *appendLog()* RPCs. After leader receives a majority consensus for the update, the state is modified.

  The appendLog() RPC also contains the previous log entry details. If any follower finds discrepancies in the log entries, it requests earlier log entries from the leader.

3. **Safety/Consistency**
  If the log of the candidate is not up-to-date there can be consistency issues.

## PBFT - Practical Byzantine Fault Tolerance

Any 2 quorums of size 2f + 1 must have atleast one honest node in intersection.

Primary runs the protocol, replicas request view change if primary fails.

## Papers
1. Algorand: Scaling Byzantine Agreements for Cryptocurrencies
2. Hybrids on Steroids: SGX-Based High Performance BFT
3. Hawk: Blockchain Model of Cryptography and Privacy Preserving Smart Contracts
4. Pinocchio: Nearly Practical Verifiable
5. The consensus number of a Cryptocurrency


# Resources
1. [muratbuffalo](https://muratbuffalo.blogspot.com/2015/02/paper-summary-perspectives-on-cap.html)
