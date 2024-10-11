---
layout: post
title:  "PyTorch parallelism"
description: Parallelism and composability with PyTorch
img:
date: 2024-10-11  +1510
---

The goal is to understand how pytorch does parallelism, especially composing multiple parallelisms (such as FSDP, TP and PP) easily to create 2D and 3D parallelism strategies.

All my experiments are on CPUs, running on a container using image: `nvcr.io/nvidia/pytorch:24.09-py3`. So, all the code is compatible with both CPUs and GPUs, though I didn't really test on GPUs.

First, it is important to understand PyTorch's Device Mesh which is one of the key techniques to enable composability.

## Understanding Device Mesh

Let's start with the simplest code for creating and examining a device mesh.

```python
import torch.distributed as dist
from torch.distributed.device_mesh import init_device_mesh

# Code for helping init process group
device = 'cuda' if torch.cuda.is_available() else 'cpu'
backend = 'nccl' if device == 'cuda' else 'gloo'

rank = int(os.environ["RANK"])
world_size = int(os.environ["WORLD_SIZE"])
local_rank = int(os.environ["LOCAL_RANK"])

dist.init_process_group(backend=backend, world_size=world_size)
if device == 'cuda':
    torch.cuda.set_device(local_rank)

# Let's create a 2D device mesh with 6 devices
device_mesh = init_device_mesh(device, (2, 3))
```

The last line of the code actually creates the device mesh.

To run this:
```bash
torchrun --node_rank=0 --nproc_per_node=6 --nnodes=1 --standalone test.py
```

Note that the `nproc_per_node` has to match the total number of devices. If you try a smaller number, we get the error `Mesh should not be bigger than default world size, but found 6 ranks!`. If we have more processes, it can throw error `IndexError: list index out of range` when we try printing and examining the mesh later.

Here, we want the TP to happen over 3 devices and DP to happen across the 2 "nodes". It starts the indexing from the outermost, so if the grid has devices with IDs 0 - 5, they are placed into 2 rows with 3 columns each:

```
<-     ->
0   1   2
3   4   5
```

Where one parallel algorithm (say Tensor Parallelism) happens over the sets of machines {0, 1, 2} and {3, 4, 5} and the other (say Data Parallelism) happens across both sets.

We can verify this by printing the mesh for each dimension. But for this we need to name the dimensions else we get the error `Cannot slice a DeviceMesh without mesh_dim_names!`.

So let's specify their names and then try to print what the allocation of devices to the device mesh is:
```python
device_mesh = init_device_mesh(device, (2, 3), mesh_dim_names = ['dp', 'tp'])
tp_mesh = device_mesh['tp']
print(f'Rank: {dist.get_rank()}, {tp_mesh}')
```

Here, we create the mesh and for each device we print it overall rank in the distributed process group (`dist.get_rank()`) and the TP mesh as seen by this device.

The output looks like this:
```bash
Rank: 0, DeviceMesh([0, 1, 2], mesh_dim_names=('tp',))
Rank: 5, DeviceMesh([3, 4, 5], mesh_dim_names=('tp',))
Rank: 4, DeviceMesh([3, 4, 5], mesh_dim_names=('tp',))
Rank: 2, DeviceMesh([0, 1, 2], mesh_dim_names=('tp',))
Rank: 1, DeviceMesh([0, 1, 2], mesh_dim_names=('tp',))
Rank: 3, DeviceMesh([3, 4, 5], mesh_dim_names=('tp',))
```

The rank 1 device prints the DeviceMesh of {0, 1, 2} and the rank 5 device prints {3, 4, 5}.

Let's switch around the dimension of the device mesh:

```python
device_mesh = init_device_mesh(device, (3, 2), mesh_dim_names = ['dp', 'tp'])
```

Now, we expect the device ids in the mesh to look like:
```
<-TP->
0    1
2    3
4    5
```

We can also see the same by printing the mesh per rank.
```bash
Rank: 0, DeviceMesh([0, 1], mesh_dim_names=('tp',))
Rank: 1, DeviceMesh([0, 1], mesh_dim_names=('tp',))
Rank: 5, DeviceMesh([4, 5], mesh_dim_names=('tp',))
Rank: 2, DeviceMesh([2, 3], mesh_dim_names=('tp',))
Rank: 4, DeviceMesh([4, 5], mesh_dim_names=('tp',))
Rank: 3, DeviceMesh([2, 3], mesh_dim_names=('tp',))
```

Now that we have tried out a simple mesh, let's start using them by trying some composite algorithms.

### Hybrid FSDP - FSDP1 + DP
The first pair of algorithms we'll try are FSDP within "nodes" and DDP across "nodes". I put nodes in quotes, since it is not necessary for the devices to be on and across nodes. The DeviceMesh doesn't really know the location of the devices, just their arrangement.

Also, there are 2 versions of FSDP which are referred to as FSDP1 and FSDP2. While the APIs are mostly compatible, they differ especially in how composition is done.

First, I use FSDP1, following the [PyTorch Tutorial for DeviceMesh for Hybrid FSDP](https://pytorch.org/tutorials/recipes/distributed_device_mesh.html#how-to-use-devicemesh-with-hsdp
). I make some small changes for execution on a CPU.

This is the code, where we define a toy model and then apply FSDP on it with a mesh of shape (2, 4).

```python
import torch
import os
from torch import distributed as dist
import torch.nn as nn
from torch.distributed.device_mesh import init_device_mesh
from torch.distributed.fsdp import FullyShardedDataParallel as FSDP, ShardingStrategy

import warnings # ignore all warning messages
warnings.filterwarnings("ignore")

class ToyModel(nn.Module):
    def __init__(self):
        super(ToyModel, self).__init__()
        self.net1 = nn.Linear(10, 10)
        self.relu = nn.ReLU()

    def forward(self, x):
        return self.relu(self.net1(x))

model = ToyModel()

device = 'cuda' if torch.cuda.is_available() else 'cpu'

# HSDP: Mesh Shape (2, 4)
"""
Create a mesh of this shape with FSDP across the row and DP across the columns
<-   FSDP  ->
0   1   2   3
4   5   6   7
"""
mesh = init_device_mesh(device, (2, 4), mesh_dim_names=["DP", "FSDP"])

model = FSDP(
    model, device_mesh = mesh,
    sharding_strategy=ShardingStrategy.HYBRID_SHARD
)
```

The last 4 lines are key, which pass the model and the device mesh to the FSDP method and use the Hybrid Sharding Strategy which is exactly meant to compose FSDP and DDP.

We can run it as before:
```bash
torchrun --node_rank=0 --nproc_per_node=8 --nnodes=1 --standalone hsdp1.py
```

It just runs and finishes with no indication on whether the FSDP happened correctly or not. Wouldn't it be nice if we could also exactly see how the weights were sharded.

For this, I took inspiration from different people's dev/debugging code. What we'll do is set the weights of the model (preferably with contiguous valuse) and let each device print it's set of weights to verify that the sharding of the weights, i.e. both the quantity and the value are as expected. 

We'll first update the model to set the model weights:
```bash
class ToyModel(nn.Module):
    def __init__(self):
        super(ToyModel, self).__init__()
        self.net1 = nn.Linear(10, 10, bias=False)
        self.relu = nn.ReLU()

        with torch.no_grad():
            self.net1.weight = nn.Parameter(torch.arange(101., 201.))

    def forward(self, x):
        return self.relu(self.net1(x))
```

I set bias to be false since it messes up the round number of weights and causes some confusion when sharded. We set the weights by directly setting the `self.net1.weight` field. This expects an input of type `nn.Parameter` which we will typecast to. Finally, we pick 100 contiguous numbers by using `torch.arange(101., 201.)` to ensure floating point numbers. You can use any range you want.

Now for the "printing" code:
```python
device = 'cuda' if torch.cuda.is_available() else 'cpu'
backend = 'nccl' if device == 'cuda' else 'gloo'

world_size = int(os.environ["WORLD_SIZE"])
local_rank = int(os.environ["LOCAL_RANK"])

dist.init_process_group(backend=backend, world_size=world_size)
if device == 'cuda':
    torch.cuda.set_device(local_rank)

print(f'Global rank: {dist.get_rank()}, dp_rank: {mesh["DP"].get_local_rank()}, fsdp_rank:{mesh["FSDP"].get_local_rank()}, Weights array: {list(model.parameters())}')
```

The first set of lines help initialize the process group. The final line prints the
1. global rank of each GPU: 0 - 7
2. The DP rank of each GPU: Since DP dimension is of size 2, this will be 0 or 1.
3. The FSDP rank of each GPU: Since FSDP dimension is of size 4, this should be a number from 0 - 3
4. and finally the set of weights or the model parameters being held by this particular GPU.

On running this, I show a portion of the output (the lines in a different order probably):
```bash
Global rank: 6, dp_rank: 1, fsdp_rank:2, Weights array: Parameter containing:
tensor([151., 152., 153., 154., 155., 156., 157., 158., 159., 160., 161., 162.,
        163., 164., 165., 166., 167., 168., 169., 170., 171., 172., 173., 174.,
        175.], requires_grad=True)
Global rank: 2, dp_rank: 0, fsdp_rank:2, Weights array: Parameter containing:
tensor([151., 152., 153., 154., 155., 156., 157., 158., 159., 160., 161., 162.,
        163., 164., 165., 166., 167., 168., 169., 170., 171., 172., 173., 174.,
        175.], requires_grad=True)
Global rank: 3, dp_rank: 0, fsdp_rank:3, Weights array: Parameter containing:
tensor([176., 177., 178., 179., 180., 181., 182., 183., 184., 185., 186., 187.,
        188., 189., 190., 191., 192., 193., 194., 195., 196., 197., 198., 199.,
        200.], requires_grad=True)
...
```

We can notice 2 things:
1. All the parameters are equally sharded among the GPUs with FSDP rank 0 - 3.
2. GPUs with the same FSDP rank by different DP rank share the same shard of the model parameters.

So, the FSDP split happens as we expected and the DDP replication of parameters is also as expected.

### Hybrid FSDP - FSDPv2 + DP
This time, we're going to use the FSDP2 library. The code doesn't change too much.

In fact, just a couple of lines of change. We'll need to import and use the `fully_shard` call instead of the `FullyShardedDataParallel` call.

``` python
from torch.distributed._composable.fsdp import fully_shard
model = fully_shard(model, mesh=mesh["FSDP"])
```

The code understands that the model needs to be split along the FSDP dimension alone, therefore automatically resulting in replication along the other axis (DP).

We can run this as well to see the print output which we expect to be the same. While the tensors are split exactly the same, the weights output is a DTensor this time, instead of a normal tensor.

``` bash
Global rank: 4, dp_rank: 1, fsdp_rank:0, Weights array: [DTensor(local_tensor=tensor([101., 102., 103., 104., 105., 106., 107., 108., 109., 110., 111., 112.,
        113., 114., 115., 116., 117., 118., 119., 120., 121., 122., 123., 124.,
        125.], device='cuda:4'), device_mesh=DeviceMesh([4, 5, 6, 7], mesh_dim_names=('FSDP',)), placements=(Shard(dim=0),))]
...
```

The DTensor consists of the weights array followed by the device mesh itself, indicating what the process group is, for the FSDP dimension. There is nothing for the DP dimension since the model doesn't change across that dimension. It also includes a `placements` field that I haven't fully understood yet.

### FSDP2 + TP
Since FSDP + DDP is a supported scenario by FSDP, it was quite easy to compose them. Suppose we have a slightly more complicated scenario - composing FSDP with TP. We follow the following tutorial.

The model definition and the process group initialization code remains the same. This time, I make the model's linear layer of size `12 x 12` to make it a round number for splitting.

Next, we'll create the mesh, same dimension as before. This time we want to do TP within a "node" and FSDP across "nodes".

```python
# TP + FSDP: Mesh Shape (2, 4)
mesh = init_device_mesh(device, (2, 4), mesh_dim_names = ['FSDP', 'TP'])
```

Now we need to apply TP. This is done using the `parallelize_module` functionality of torch distributed. Since we only have one linear network, we'll split it column wise.

```python
from torch.distributed.tensor.parallel import parallelize_module, ColwiseParallel

# Apply TP
model = parallelize_module(
    model,
    mesh['TP'],
    {
        "net1": ColwiseParallel(),
    }
)
```

Then, we'll apply FSDP.

```python
# Apply FSDP2
model = fully_shard(model, mesh=mesh['FSDP'], reshard_after_forward=True)
```

Finally, print the sharded weights on each GPU using the output of `model.parameters()`.

```python
print(f'Global rank: {dist.get_rank()}, fsdp_rank: {mesh["FSDP"].get_local_rank()}, tp_rank:{mesh["TP"].get_local_rank()}, Weights array: {list(model.parameters())}')
```

This time, we will also check the full set of weights after putting it back together using the `param.full_tensor()`.

```python
# Check the full unsharded weights
full_tensor = [param.full_tensor() for param in model.parameters()]
if dist.get_rank() == 0:
    print(full_tensor)
```

If you see the put-together set of weights, you'll notice something wrong.

```bash
[tensor([101., 102., 103., 104., 105., 106., 107., 108., 109., 110., 111., 112.,
        113., 114., 115., 116., 117., 118., 137., 138., 139., 140., 141., 142.,
        143., 144., 145., 146., 147., 148., 149., 150., 151., 152., 153., 154.,
        173., 174., 175., 176., 177., 178., 179., 180., 181., 182., 183., 184.,
        185., 186., 187., 188., 189., 190., 209., 210., 211., 212., 213., 214.,
	...
       device='cuda:0', grad_fn=<_ToTorchTensorBackward>)]
```

The full set of weights are not in the right order! This is a bug in the TP + FSDP2 implementation currently. Link to [a Github issue](https://github.com/pytorch/pytorch/issues/129229) and [another one](https://github.com/pytorch/pytorch/issues/129206) on this.

This is apparently because DTensor assumes left-to-right sharding order FSDP, then TP whereas the actual logic is TP, then FSDP (to reduce inter-node communication).

The fix for this is in [this PR](https://github.com/pytorch/pytorch/pull/130760). I don't fully understand it and will probably put out a second in this series on a deeper dive into how composability is done and other low level details.
