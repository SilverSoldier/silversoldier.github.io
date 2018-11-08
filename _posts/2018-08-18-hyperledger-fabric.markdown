---
layout: post
title:  "Hyperledger Fabric - A Distributed Operating System for Permissioned Blockchains"
description: A summary of the paper by the same name.
img:
date: 2018-08-18  +1448
---

Blockchain is an *immutable ledger* of *transactions*, maintained by a distributed network of untrusting peers.

### Public vs Permissioned Blockchains
+ Public: all nodes can participate and write to the ledger. Has an associated cryptocurrency and consensus protocol.
+ Permissioned: 
	
### order-execute architecture:
+ This is the architecture traditionally used by most blockchain platforms.
+ Transactions are executed after consensus (ordering) and all participants execute all contracts sequentially.
+ Limits scalability, requires sequential execution of transactions and endorsement by all peers.

#### Detailed working
In a PoW based permissionless blockchain
1. Every peer assembles a block containing valid transactions.
2. The peer tries to solve a PoW puzzle.
3. The peer which solves the puzzle first, broadcasts the solution via a gossip protocol.
4. Every peer receiving the solution validates solution and all the transactions.

All peers execute transactions sequentially (within a block) and among all blocks.

### execute-order-validate architecture:
+ The architecture used in Hyperledger Fabric. Lets transactions execute before the blockchain reaches consensus.
+ Transactions are executed and endorsed first and then ordered and validated.
+ Separates transaction flow into modular building blocks.

## Fabric Architecture
### Node types
### execute-order-validate

## Fabric Phases
+ Execution:
	+ A client sends a proposal to all the nodes.
	+ Nodes need to execute as per endoresement policy.
