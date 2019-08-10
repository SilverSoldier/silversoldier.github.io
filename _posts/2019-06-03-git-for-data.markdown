---
layout: post
title:  "Version Control for Data"
description: Interesting points and topics in a talk about using DVCS for convergence and consistency
img:
date: 2019-03-06  +1900
---

Just attended a talk titled "Version Control for Data".

### Abstract

Programmers regularly use distributed version control systems (DVCS) such as Git to facilitate collaborative software development. The primary purpose of a DVCS is to maintain the integrity of source code in the presence of concurrent, possibly conflicting edits from collaborators. In addition to safely merging concurrent non-conflicting edits, a DVCS extensively tracks source code provenance to help programmers contextualize and resolve conflicts. Provenance also facilitates debugging by letting programmers see diffs between versions and quickly find those edits that introduced the offending conflict (e.g., via git blame). I posit that analogous workflows to collaborative software development also arise in distributed software execution. In this talk, I will argue that the characteristics that make a DVCS an ideal fit for the former also make it an ideal fit for the latter. I will present some previous work we've done on mergeable replicated data types (MRDTs) and how we realize them in a practical programming language. Then I'll discuss some open problems in implementing and verifying MRDTs and present ongoing work to address some of the issues. 

### Useful links suggested

(1) https://blog.acolyer.org/2015/01/14/mergeable-persistent-data-structures/
(2) http://kcsrk.info/papers/mergeable_types_ml17.pdf
(3) https://ipfs.io/ & https://ipld.io/

### Summary and awesome things I learnt

**Version Control Systems**:

Look into darcs, an interesting VCS.

**Lazy or strict**:

Consider a distributed system setting (such as a distributed DB/KV store), with each system getting write requests.

Suppose the system promises strict consistency and total ordering. We would need a consensus or an atomic broadcast protocol such as Paxos or Raft for the same. Such a system would have an extremely low latency which it traded in for correctness.

The system could also maintain a lazy version, which is eventual consistency (a model followed by many systems for performance and availability during a partition purposes).

We would like a system to have something in the middle.

**Convergent Replicated Data Types:**

CRDT's are objects that can be updated without expensive consensus because:

+ either they are orthogonal and don't affect each other.
+ or there is a linear merge operation due to commutativity.

Some popular CRDT Key-Value stores are Basho's Riak (Erlang), Facebook's Voldermort (Java)

**The 3 Way Merge**:

The 3 way merge is a function (used by Git to merge 2 versions), which has a signature like so:

```
merge(LeastCommonAncestor, Version1, Version2) -> MergedVersion
```

The merged version could be of the same datatype as Version1, say in a text file merging, all three versions are a list of strings.

Or, in order to accomodate such things as conflicting lines in a text file, we could also return a data type which incorporates the conflict and lets the user decide what to do with it.

It could return a list of set of strings.

For instance, multi value register in Dynamo.

In case of a register update, normally registers can only store 1 value. A multi-value register allows versions of values to be stored.

**Irmin**

**IPFS and IPLD:**

Inter Planetary File System and Inter Planetary - -

Unifying world's data.

