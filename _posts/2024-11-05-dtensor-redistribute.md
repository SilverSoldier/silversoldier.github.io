---
layout: post
title:  Understanding DTensor Redistribute 
description: "PyTorch parallelism: Part 3"
img:
date: 2024-11-04  +1100
---

Part 3 of Pytorch Parallelism Series. Previous parts are:
1. [Pytorch Device Mesh and Composability](https://gkavya.in/pytorch-parallelism/)
2. [DTensor Matrix Multiplication](https://gkavya.in/pytorch-dtensor-mm/)


Motivation: To understand PyTorch TP whose magic happens under the "Redistribute" call.

## What is redistribute()?
Redistribute is a torch.autograd.Function class with a forward function. It takes an input DTensor and a target Placement and calls `redistribute_local_tensor` if something needs to be changed and finally returns the "redistributed" DTensor (torch.distributed.tensor) as an output.
There is also the `distribute_tensor` API, which takes a Tensor and converts into a DTensor.

`redistribute_local_tensor(local_tensor, current_spec, target_spec)` takes a local DTensor and applies the target spec to get the new version of the DTensor, and does this by making collective/communication calls. Spec is a data type encompassing mesh and placement.
Currently it does not support if the source and target meshes are different. Which means, it only supports a change in placement strategy within the same mesh.

It is called by the `redistribute()` API of `DTensor`, which in turn is used by parallelization algorithm (atleast TP does). For ex. in TP, the current DTensor is passed and redistribution is done to achieve the next module's input DTensor.

## How does it work?

The function has 2 parts: first is creating a list of transforms to go from the source placement to the target placement. This logic is handled by the `gen_transform_infos_non_cached` function. For a 1D mesh, redistribution is simple, but for an N-D mesh, this function implementation is a little bit complicated. The problem is in the case of nested sharding (sharding across multiple mesh dimensions), for which they do a reverse sweep over the mesh dimensions and if nested sharding occurs, need to Replicate once for this transform. Otherwise, in the simple case, the Transform path is for every mesh dimension, the path from source to dest placement from the inner (last dimension) to the outer (first dimension).

Second, once the path of transforms is known, the actual transformation needs to happen, depending on the current Placement and the target Placement. Below is the list of different combos, with the expected collective in square brackets and the actual function called (from the source code) following that.
1. Partial -> Replicate [all_reduce]: `_reduce_value()` 
2. Partial -> Shard [reduce_scatter]: `reduce_shard_value()` -> `reduce_shard_tensor()` -> `reduce_scatter_tensor()`
3. Shard -> Replicate [all_gather]: `to_replicate_tensor()` -> `all_gather_tensor()` 
4. Shard -> Partial [?]: for backward apparently just convert to replicate
5. Shard -> Shard [all_to_all for sharding over new dimension]: `to_new_shard_dim()` -> `shard_dim_alltoall()` -> {CPU/`all_gather()` + `chunk()`}/{GPU/`shard_dim_alltoall()`} 
6. Replicate -> Partial [split randomly?]: 
7. Replicate -> Shard [split into chunks]: `_replicate_to_shard()` -> `_split_tensor()`

The first 3 are straightforward, the rest all quickly become weird cases. For 4 and 6 where Partial is the target, my first thought would be to get some random numbers and make sure the sum adds up? Thankfully, both of these have some explanation as comments in the code.

For 4. Replicate -> Partial says to skip when backward (basically forward input as is). For non-backward, it calls a `_partial_value` function which is defined per op. For embedding op, it seems to be defined like a form of sharding while for math they have some logic depending on the function to sort of split uniformly by value.

For 6. Shard -> Partial is ONLY supported for backward and basically just behaves like Replicate for that. This is a bit confusing and misleading, but I am not an expert on when this is called, so should probably go with them on this.

I first thought 5 was a resharding, something like go from Shard into 4 before to Shard into 2 now. But that doesn't make sense when you think about it, since the `N_GPU` is not expected to be a dynamic parameter (atleast for now, could these Placements allow dynamism by re-using these APIs? food for thought). So 5 is about sharding but in a different dimension from before. When would this be needed? 2 levels of sharding occurs when composing, say FSDP + TP, where the same tensor is sharded once for TP and is sharded again (from another dimension for FSDP).

5's logic needed a little bit understanding. [`all_to_all`](https://pytorch.org/docs/stable/distributed.html#torch.distributed.all_to_all) is a neater version of gather then scatter, which automatically re-arranges a tensor along a different dimension. Given inputs of (0,1) on rank 1 and (2, 3) on rank 2, an `all_to_all` will results in rank 1 having (0, 2) and rank 2 having (1, 3). However, Gloo does not support `all_to_all`, so the code also handles fall back to `all_gather` to replicate the whole tensor, followed by chunking along the shard dimension to get each rank's chunk. Else, it directly calls the `alltoall` over the process group.

Finally, 7 doesn't have any collective operations, because it simply shards. It internally calls `_split_tensor()` which uses `torch.chunk` to split a tensor into N chunks along a particular dimension and handling padding etc.

Misc points:
1. One note is that while jagged mesh dimension support is said to be "experimental", all these cases seem to handle padding of the mesh (lots of code to handle such cases).
2. Uneven shards of each dimension may be possible with dynamic padding
