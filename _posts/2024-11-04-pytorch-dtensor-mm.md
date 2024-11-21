---
layout: post
title:  DTensor Matrix Multiplication
description: "PyTorch parallelism: Part 2"
img:
date: 2024-11-04  +1045
---

Part 2 of Pytorch Parallelism Series. Previous part is:
Part 1: [Pytorch Device Mesh and Composability](https://gkavya.in/pytorch-parallelism/)

Future parts are:

Part 3: [Understanding DTensor Redistribute](https://gkavya.in/dtensor-redistribute/)

Part 4: [Understanding PyTorch TP](https://gkavya.in/pytorch-tp/)

Motivation: to understand TP, I was looking into how DTensor `redistribute` works. But for that, I needed to first understand how DTensor matrix multiplication works (i.e. understanding a Rowwise or Colwise sharded module's output).

First, what is the `DTensor` or distributed tensor. A DTensor is created by "distributing" a normal tensor over a mesh using a placement strategy (my term, not official). This placement strategy is simply an array of Placement Types, one for each dimension of the mesh.

Placement Type is of 3 types - Shard, Replicate and Partial:
- `Shard` takes a dimension as an argument and splits the tensor along that dimension into N shards as per the DeviceMesh dimension.
- `Replicate` takes no argument and replicates the tensor along the DeviceMesh dimension.
- `Partial` represents multiple DTensors with partial values which need to be reduced (using a reduce fn) to get the full Tensor.

To quickly give an example, given a full tensor of size 8, 2 `Shard`'ed DTensors are of size 4 each, the `Replicate` of size 8 and the 2 `Partial` DTensors of size 8 each but need to be added (or reduced by any other operation) to get the full tensor.

A DTensor has been "placed" according to a Placement Type. Matrix multiplication of 2 DTensors of different Placement Types results in different local DTensors, not just value-wise (which is expected) but also output "Placement Type"-wise. This blog is to try out the different combos and note the results. As a bonus, we can also see how redistribute works internally.

(The PyTorch code implementing this logic is in `torch/distributed/tensor/_ops/_matrix_ops.py`, which defines the strategy for transpose, matrix multiplication etc. While transpose is simple enough, the multiplication logic is slightly complicated and includes calculating redistribute costs which I don't cover here).

First, we'll setup the mesh as before (single dimensional of size 2) to try the different combos one by one.

```python
import torch
import os

from torch import distributed as dist
from torch.distributed.device_mesh import init_device_mesh
from torch.distributed._tensor import Replicate, Shard, DTensor

(device, backend) = ('cuda', 'nccl') if torch.cuda.is_available() else ('cpu', 'gloo')

dist.init_process_group(backend=backend, world_size=int(os.environ["WORLD_SIZE"]))
if device == 'cuda':
    torch.cuda.set_device(int(os.environ["LOCAL_RANK"]))

mesh = init_device_mesh(device, (2,))

tensor1 = torch.arange(1, 25).view(4, 6) 
tensor2 = torch.arange(1, 25).view(6, 4)
```

Then we define the 2 tensors that we want to multiply. I am going to take the continuous numbers matrices similar to the code from the other blogs in this series. The first matrix (X) is 4 x 6 and the second (Y) is 6 x 4.

Instructions to run:
```
torchrun --nproc_per_node=2 mm_test.py
```

## Replicate x Replicate
Replicate x Replicate is the simplest:

```python
# Replicate x Replicate
dtensor1 = DTensor.from_local(tensor1, mesh, placements=[Replicate()])
dtensor2 = DTensor.from_local(tensor2, mesh, placements=[Replicate()])
x = dtensor1
y = dtensor2
print(f'Rank: {dist.get_rank()}, input: {y}')
z = x @ y
print(f'Rank: {dist.get_rank()}, output: {z}')
```

We define the 2 DTensors using the `from_local` API. Since, we only want to do `Replicate`, the inputs are directly these DTensors (without having to `redistribute`). We are also going to print per-rank input (only Y) and output.

```bash
Rank: 0, input: DTensor(local_tensor=tensor([
	[ 1,  2,  3,  4],
        [ 5,  6,  7,  8],
        [ 9, 10, 11, 12],
        [13, 14, 15, 16],
        [17, 18, 19, 20],
        [21, 22, 23, 24]]),
	device_mesh=DeviceMesh('cpu', [0, 1]), placements=(Replicate(),))
Rank: 1, input: DTensor(local_tensor=tensor([
	[ 1,  2,  3,  4],
        [ 5,  6,  7,  8],
        [ 9, 10, 11, 12],
        [13, 14, 15, 16],
        [17, 18, 19, 20],
        [21, 22, 23, 24]]),
	device_mesh=DeviceMesh('cpu', [0, 1]), placements=(Replicate(),))

Rank: 1, output: DTensor(local_tensor=tensor([
	[ 301,  322,  343,  364],
        [ 697,  754,  811,  868],
        [1093, 1186, 1279, 1372],
        [1489, 1618, 1747, 1876]]),
	device_mesh=DeviceMesh('cpu', [0, 1]), placements=(Replicate(),))
Rank: 0, output: DTensor(local_tensor=tensor([
	[ 301,  322,  343,  364],
        [ 697,  754,  811,  868],
        [1093, 1186, 1279, 1372],
        [1489, 1618, 1747, 1876]]),
	device_mesh=DeviceMesh('cpu', [0, 1]), placements=(Replicate(),))
```

We see that the inputs on each rank are identical, as generated. The outputs on each rank are also identical, DTensors of type `Replicate` and of size 4 x 4. Nothing special here.

## Replicate x Shard
When the first matrix is of size 4 x 6 and the second is 6 x 4, and we want the second to be sharded, the second matrix needs to be sharded along the second dimension, making it 6 x 2. I first tried defining using the `from_local` call, but it only seems to define the DTensor of the size and Placement Type specified (without actually sharding if needed). 

To create the DTensor and actually shard it, we need to use the `redistribute` API using the DTensor created above (`dtensor1` and `dtensor2`) to create the redistributed versions (`x` and `y`). Code below, with print statements as before to view both the input and output on both the ranks.

```python
# Replicate x Shard
x = DTensor.redistribute(dtensor1, mesh, placements=[Replicate()])
y = DTensor.redistribute(dtensor2, mesh, placements=[Shard(-1)])
print(f'Rank: {dist.get_rank()}, input: {y}')
z = x @ y
print(f'Rank: {dist.get_rank()}, output: {z}')
```

For the output, apart from the prints in the program, I added a few prints in the local copy of `torch/distributed/tensor/_redistribute.py` and `torch/distributed/tensor/_api.py` to verify the path that is invoked. Apart from printing the function names, I also print the `transform_infos` local variable at `redistribute_local_tensor()` which has the list of transforms which need to be done on the DTensor. It has an array of TransformInfo objects which is represented as a tuple: (source placement, destination placement).

```bash
Redistribute called
Redistribute.apply called
redistribute_local_tensor called
[_TransformInfo(mesh_dim=0, src_dst_placements=(Replicate(), Shard(dim=1)), logical_shape=[6, 4])]
[rank0]:W1023 10:45:23.998000 44598 torch/distributed/tensor/_redistribute.py:204] redistribute from R to S(1) on mesh dim 0
Rank: 0, input: DTensor(local_tensor=tensor([
	[ 1,  2],
	[ 5,  6],
	[ 9, 10],
	[13, 14],
	[17, 18],
	[21, 22]]),
	device_mesh=DeviceMesh('cpu', [0, 1]), placements=(Shard(dim=1),))
[rank1]:W1023 10:45:23.998000 44599 torch/distributed/tensor/_redistribute.py:204] redistribute from R to S(1) on mesh dim 0
Rank: 1, input: DTensor(local_tensor=tensor([
	[ 3,  4],
	[ 7,  8],
	[11, 12],
	[15, 16],
	[19, 20],
	[23, 24]]),
	device_mesh=DeviceMesh('cpu', [0, 1]), placements=(Shard(dim=1),))
Rank: 0, output: DTensor(local_tensor=tensor([
	[ 301,  322],
	[ 697,  754],
	[1093, 1186],
	[1489, 1618]]),
	device_mesh=DeviceMesh('cpu', [0, 1]), placements=(Shard(dim=1),))
Rank: 1, output: DTensor(local_tensor=tensor([
	[ 343,  364],
	[ 811,  868],
	[1279, 1372],
	[1747, 1876]]),
	device_mesh=DeviceMesh('cpu', [0, 1]), placements=(Shard(dim=1),))
```

First, let's look at the input. The logs show redistribute being called to transform the input matrix from Replicate to Shard along dimension 1. The matrix itself can be seen of size 6x2 and sharded across the 2 ranks.

The output matrix is of size 4x2 on each rank, and this is just a shard of the total output. The output DTensor has the Placement Type of Shard(1). If we want to see the full matrix put together now, we can do `z.full_tensor()` which again calls `redistribute` from Shard back to Replicate which internally calls `_to_replicate_tensor()` and performs an `all_gather_tensor` to put the result tensor together.

```bash
[rank0]:W1024 05:10:32.598000 45384 torch/distributed/tensor/_redistribute.py:202] redistribute from S(1) to R on mesh dim 0
tensor([[ 301,  322,  343,  364],
        [ 697,  754,  811,  868],
        [1093, 1186, 1279, 1372],
        [1489, 1618, 1747, 1876]])
```

## Shard x Shard

The most interesting case. We want to shard both the X (4x6) and Y (6x4) matrices. Logically, X should be sharded along the second dimension to become 4x3 and Y, along the first to get a 3x4 matrix. Finally, each rank will have a 4x4 matrix which need to be added together to get the result.

To understand, 
```python
X = [x1 x2]
Y = [y1
	y2]
	 
X@Y = x1@y1 + x2@y2
```

Code for matrix multiplication:
```python
# Shard x Shard
x = DTensor.redistribute(dtensor1, mesh, placements=[Shard(-1)])
y = DTensor.redistribute(dtensor2, mesh, placements=[Shard(0)])
print(f'Rank: {dist.get_rank()}, input: {y}')
z = x @ y
print(f'Rank: {dist.get_rank()}, output: {z}')
```

Output logs:
```bash
[_TransformInfo(mesh_dim=0, src_dst_placements=(Replicate(), Shard(dim=1)), logical_shape=[4, 6])]
[rank1]:W1023 11:08:57.521000 44822 torch/distributed/tensor/_redistribute.py:204] redistribute from R to S(1) on mesh dim 0
[_TransformInfo(mesh_dim=0, src_dst_placements=(Replicate(), Shard(dim=0)), logical_shape=[6, 4])]
[rank1]:W1023 11:08:57.522000 44822 torch/distributed/tensor/_redistribute.py:204] redistribute from R to S(0) on mesh dim 0
Rank: 1, input: DTensor(local_tensor=tensor([
	[13, 14, 15, 16],
	[17, 18, 19, 20],
	[21, 22, 23, 24]]),
	device_mesh=DeviceMesh('cpu', [0, 1]), placements=(Shard(dim=0),))
Rank: 0, input: DTensor(local_tensor=tensor([
	[ 1,  2,  3,  4],
	[ 5,  6,  7,  8],
	[ 9, 10, 11, 12]]),
	device_mesh=DeviceMesh('cpu', [0, 1]), placements=(Shard(dim=0),))
```

Let's examine the inputs. The 4x6 matrix, i.e. X, got redistributed from R to S(1). Y got redistributed from R to S(0). Y matrix is a 3x4 shard.

Now, logs of the output matrix:
```bash
Rank: 1, output: DTensor(local_tensor=tensor([
	[ 263,  278,  293,  308],
	[ 569,  602,  635,  668],
	[ 875,  926,  977, 1028],
	[1181, 1250, 1319, 1388]]),
	device_mesh=DeviceMesh('cpu', [0, 1]), placements=(Partial(sum),))
Rank: 0, output: DTensor(local_tensor=tensor([
	[ 38,  44,  50,  56],
	[128, 152, 176, 200],
	[218, 260, 302, 344],
	[308, 368, 428, 488]]),
	device_mesh=DeviceMesh('cpu', [0, 1]), placements=(Partial(sum),))
```

Post the matrix multiplication, we see the output at each rank is a 4x4 matrix of PlacementType Partial which needs to be reduced by sum. And true enough, each rank holds a part of the answer. Now to see the actual result, calling `full_tensor` will internally call redistribute of Partial to Replicate which calls an `all_reduce`:

```bash
[rank0]:W1024 05:20:41.007000 45460 torch/distributed/tensor/_redistribute.py:202] redistribute from P to R on mesh dim 0
tensor([[ 301,  322,  343,  364],
        [ 697,  754,  811,  868],
        [1093, 1186, 1279, 1372],
        [1489, 1618, 1747, 1876]])
```

## Transpose
Finally, just because it comes up in TP, I want to see what happens when a transpose operation is performed. I am going to take a 4x6 matrix of type `Shard(0)`, perform transpose and see the output of the transpose operation.

```python
# Matrix is of size 4 x 6
tensor = torch.arange(1, 25).view(4, 6) 
dtensor = DTensor.redistribute(tensor, mesh, placements=[Shard(0)])
x = DTensor.from_local(dtensor, mesh, placements=[Replicate()])
z = x.T
```

```bash
[rank0]:W1024 05:58:34.029000 45534 torch/distributed/tensor/_redistribute.py:202] redistribute from R to S(0) on mesh dim 0
Rank: 0, input: DTensor(local_tensor=tensor([
	[ 1,  5,  9, 13, 17, 21],
	[ 2,  6, 10, 14, 18, 22]]),
	device_mesh=DeviceMesh('cpu', [0, 1]), placements=(Shard(dim=0),))
Rank: 1, input: DTensor(local_tensor=tensor([
	[ 3,  7, 11, 15, 19, 23],
	[ 4,  8, 12, 16, 20, 24]]),
	device_mesh=DeviceMesh('cpu', [0, 1]), placements=(Shard(dim=0),))

Rank: 0, output: DTensor(local_tensor=tensor([
	[ 1,  7],
	[ 2,  8],
	[ 3,  9],
	[ 4, 10],
	[ 5, 11],
	[ 6, 12]]),
	device_mesh=DeviceMesh('cpu', [0, 1]), placements=(Shard(dim=1),))
Rank: 1, output: DTensor(local_tensor=tensor([
	[13, 19],
	[14, 20],
	[15, 21],
	[16, 22],
	[17, 23],
	[18, 24]]),
	device_mesh=DeviceMesh('cpu', [0, 1]), placements=(Shard(dim=1),))
```

Note that the output is Shard(1) while input was Shard(0), so DTensor captured the transpose operation and changed the shard dimension of the output accordingly.

Full code for the matrix multiplications:
```python
import torch
import os

from torch import distributed as dist
from torch.distributed.device_mesh import init_device_mesh
from torch.distributed._tensor import Replicate, Shard, DTensor

""" 
DTensor matrix multiplication examples
"""

(device, backend) = ('cuda', 'nccl') if torch.cuda.is_available() else ('cpu', 'gloo')

dist.init_process_group(backend=backend, world_size=int(os.environ["WORLD_SIZE"]))
if device == 'cuda':
    torch.cuda.set_device(int(os.environ["LOCAL_RANK"]))

mesh = init_device_mesh(device, (2,))

tensor1 = torch.arange(1, 25).view(4, 6) 
tensor2 = torch.arange(1, 25).view(6, 4)

# Replicate x Replicate
dtensor1 = DTensor.from_local(tensor1, mesh, placements=[Replicate()])
dtensor2 = DTensor.from_local(tensor2, mesh, placements=[Replicate()])
x = dtensor1
y = dtensor2

# Replicate x Shard
#x = DTensor.redistribute(dtensor1, mesh, placements=[Replicate()])
#y = DTensor.redistribute(dtensor2, mesh, placements=[Shard(0)])

# Shard x Shard
#x = DTensor.redistribute(dtensor1, mesh, placements=[Shard(-1)])
#y = DTensor.redistribute(dtensor2, mesh, placements=[Shard(0)])

print(f'Rank: {dist.get_rank()}, input: {y}')
z = x @ y
print(f'Rank: {dist.get_rank()}, output: {z}')
if dist.get_rank() == 0:
    print(z.full_tensor())
```
