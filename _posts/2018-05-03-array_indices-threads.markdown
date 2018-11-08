---
layout: post
title:  "Dividing array indices among threads"
description: How to divide m array indices or m loop iteration (almost) equally among n threads when array indices is not a multiple of the number of threads.
img:
date: 2018-05-03  +0706
---

# Dividing m array indices among n threads when number of elements to divide is not a multiple of the number of threads or processes.

Many times, when writing a parallel program in Java or OpenMPI, (not OpenMP), I have a *m* sized array or *m* iterations of a loop that I need to divide as equally as possible among *n* threads or processes and m is not a multiple of n, I have always been playing around with different ways of dividing and keep creating a sub-optimal implementation from scratch every time.

The *m* array elements or loop iterations will hereby be referred to as **tasks**.

The *n* threads/processes will hereby be referred to as UE (Unit of Execution), borrowing the terminology from a great book Patterns of Parallel Programs.

I'll skip writing in detail, the 2 most obvious methods

+ Method I: 
	We give each UE `m/n` tasks which misses some of tasks.

+ Method II: which gives all the remaining to the last UE

and jump to the actual solution.

We first each of the UE's an equal chunk of the tasks. The remainder will be given to the first remainder UEs.

The code is as follows (in C):
```
int n_tasks, n_units, id;
// Assuming each UE will execute the following code
int chunk_size = (n_tasks/n_units);
int rmdr = n_tasks - chunk_size * n_units;
chunk_start = chunk_size * rank;
if(rank < rmdr){
	chunk_size++;
	chunk_start += rank;
}
else
	chunk_start += rmdr;

chunk_end = chunk_start + chunk_size;
```

T. Mattson, B. Sanders, and B. Massingill, Patterns for Parallel Programming , Addison-Wesley, 2005, ISBN 0-321-22811-1.
