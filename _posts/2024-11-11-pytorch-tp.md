---
layout: post
title: Understand PyTorch TP
description:  "PyTorch parallelism: Part 4"
img:
date: 2024-11-04  +1100
---

Code files: `pytorch/torch/distributed/tensor/parallel/style.py` and some of the `dtensor` related files.

The outermost API for Tensor parallelism is the [`parallelize_module`](https://pytorch.org/docs/stable/distributed.tensor.parallel.html#torch.distributed.tensor.parallel.parallelize_module) call. From the docs, an example of how you would use it is:
```
m = Model()
mesh = init_device_mesh("cuda", (8,))
m = parallelize_module(m, mesh, parallelize_plan = {"w1": ColwiseParallel(), "w2": RowwiseParallel()}
```

Essentially, the `parallelize_module` call takes in a plan which identifies the underlying modules by their FQN (fully qualified name) and specifies how we want to parallelise that module. Pytorch TP allows a few strategies (whose implementation is in `style.py`). The 3 available styles are:
- `ColwiseParallel`: Partition a Linear or Embedding module in a column-wise manner. 
- `RowwiseParallel`: Partition a Linear or Embedding module in a row-wise manner.
- `SequenceParallel`: Replicates LayerNorm, Dropout or RMSNorm module and runs on a shard of the input. This is for parallelizing computation which can be done parallely across all inputs.
 
All of these derive from the base class `ParallelStyle` which has an `_apply` function which these different classes override to define how they parallelize that module. `parallelize_module` is called recursively until it finds a leaf `ParallelStyle` type node which it then "applies". `_apply` returns a Distributed Module which is the module with its parameters, inputs and outputs "managed" to handle the parallelization.

Rowwise and Colwise ParallelStyles are mostly similar in functionality, excepting the changes related to sharding style. They both:
- Have an `input_layout` and `desired_input_layout` field. `input_layout` is the Placement Type of the input that will be passed (Note this needs to be known before-hand when defining the parallelization plan itself). `desired_input_layout` is how the parallelization needs it for the module. This is hard-set as  `Replicate` for ColwiseParallel and for RowwiseParallel, depends on whether it is Linear module (`Shard(-1)`) or Embedding module (`Replicate`). This is then used by the `prepare_input_fn` which redistributes from `input_layout` to `desired_input_layout`.
  - Note: this `prepare_input_fn` is not called, it is passed as a partial function as the DistributedModule, i.e. output of the `paralellize_module`, and registered as a `pre_forward` function which is only called when an actual input is passed to this module.
  - For instance, the torchtitan code for TP for Llama, defines the Embedding module to be RowwiseParallel but input is `Replicate`d and output is `Shard`ed.
- Have an `output_layout` which is actually the `desired_output_layout`, i.e. how does the user expect the output of this module to become. This is used by the `prepare_output_fn` which `redistribute`s the output from how the module outputs it (result of matrix multiplication) to `output_layout`.
  - Similar to above, this partial function is registered as a `post_forward` function.
- Have a `partition_fn` to actually shard or partition the module. Row and column parallel define separate `partition_linear` and `partition_embedding`
All of the above are returned as functions to the `distribute_module`.

SequenceParallel:
- Replicates module parameters, runs sharded computation with input sharded along sequence dimension
- Has `replicate_module_fn` which simply initializes a DTensor using the `from_local()` call.
- `prepare_input_fn`: Takes care of redistributing the input if is not already sharded along sequence dimension (Shard(1))
- `prepare_output_fn`: Return as is

The input and output functions handle what massaging needs to be done on the inputs before calling and outputs after calling. These will be installed as `pre_forward` and `post_forward` hooks and will involve the communication primitives.

I am going to take a toy `nn.Module` and do a trial run of the TP parallel algorithm for better understanding, this is both the offline steps and the online steps.
1. Assume a simple `nn.Module` with a single Linear layer of size 10. The weights are (10, 10). I am going to create them with continuous values for better visualization.
2. Let's parallelize it in a ColwiseParallel manner, so that the weight matrix becomes 10x5. (However, the weight matrix is transposed before input is multiplied, so the logic for partitioning the Linear layer results in 5 x 10 size matrix.)
3. We create a parallelization plan with the linear layer set to ColwiseParallel and call `parallelize_module`.
4. This will call the `apply` of the ColwiseParallel style which passes the partition function for the Linear module, `prepare_input_fn` and a `prepare_output_fn` to the `distribute_module` API. That API will run the `partition_fn` to shard the parameters and register the `prepare_input_fn` and `prepare_output_fn` as `pre_forward` and `post_forward` hooks respectively.
5. Verify the model is sharded by printing the model weights. Notice that they are actually row-wise sharded as explained before.
   ```
Global rank: 0, Weights array: [DTensor(local_tensor=tensor([[101., 102., 103., 104., 105., 106., 107., 108., 109., 110.],
        [111., 112., 113., 114., 115., 116., 117., 118., 119., 120.],
        [121., 122., 123., 124., 125., 126., 127., 128., 129., 130.],
        [131., 132., 133., 134., 135., 136., 137., 138., 139., 140.],
        [141., 142., 143., 144., 145., 146., 147., 148., 149., 150.]]), device_mesh=DeviceMesh('cpu', [0, 1]), placements=(Shard(dim=0),))]
Global rank: 1, Weights array: [DTensor(local_tensor=tensor([[151., 152., 153., 154., 155., 156., 157., 158., 159., 160.],
        [161., 162., 163., 164., 165., 166., 167., 168., 169., 170.],
        [171., 172., 173., 174., 175., 176., 177., 178., 179., 180.],
        [181., 182., 183., 184., 185., 186., 187., 188., 189., 190.],
        [191., 192., 193., 194., 195., 196., 197., 198., 199., 200.]]), device_mesh=DeviceMesh('cpu', [0, 1]), placements=(Shard(dim=0),))]
    ```
6. When `model.forward` is called, it first runs the pre_forward function which sees that the inputs which are currently Tensors that need to be converted to DTensors with the Replicate type using the `from_local` method. It finds that the input layout and desired input layout is the same, so does not call redistribute.
   ```
   print(f'Input=== from: {input_layouts} to: {desired_input_layouts}')
   Input=== from: (Replicate(),) to: (Replicate(),)
   ```
7. Then the inputs run through the sharded Linear module to get the output.
8. Then the post_forward is called which does the output conversion from existing layout to desired layout. Existing is how the module outputs and desired is what was registered according to ColwiseParallel. This raises the question of what does the module give as output. See [my previous blog]() on how DTensor matrix multiplication works and how output depends on input types. Output of Replicate x Shard(i) is a Shard(i) along the same dimension. But since there is a transpose involved in weight multiplication, the Shard dimension is also swapped, therefore Shard(1) instead of Shard(0) that we expect. Desired output is Shard(-1) which is the same. Hence no redistribute is needed.
   ```
   print(f'Output=== from: {outputs.placements} to: {output_layouts}')
   Output=== from: (Shard(dim=1),) to: (Shard(dim=-1),)
   ```

(Code output for points 6 and 8 is by editing the `torch/distributed/tensor/parallel/style.py` and adding print statements in the `ColwiseParallel` class in the `prepare_input` and `prepare_output` functions respectively.)

That's it. A single col-wise sharded Linear layer does not require any redistribution at all. So, let's try to force a redistribution.

## Option 1:

Let's add another Linear module that is Rowwise sharded. This is the logic that is done in the Megatron TP paper as well because it does not require any communication in between the layers in that case.
We create a net2, a new Linear Layer. The forward function is now `net2(net1(x))`. The parallelize module will shard net2 in a Rowwise manner.
```
class ToyModel(nn.Module):
    def __init__(self):
        super().__init__()
        self.net1 = nn.Linear(10, 10, bias=False)
        self.net2 = nn.Linear(10, 10, bias=False)

		self.net1.weight = nn.Parameter(torch.arange(101., 201.).view(10, 10))
        self.net2.weight = nn.Parameter(torch.arange(101., 201.).view(10, 10))

    def forward(self, x):
        return self.net2(self.net1(x))

model = parallelize_module(
    model,
    mesh,
    {
        "net1": ColwiseParallel(),
        "net2": RowwiseParallel(),
    }
)

```

Let's examine the outputs as before: 
```
Global rank: 0, Weights array: [DTensor(local_tensor=tensor([
	[101., 102., 103., 104., 105., 106., 107., 108., 109., 110.],
	[111., 112., 113., 114., 115., 116., 117., 118., 119., 120.],
	[121., 122., 123., 124., 125., 126., 127., 128., 129., 130.],
	[131., 132., 133., 134., 135., 136., 137., 138., 139., 140.],
	[141., 142., 143., 144., 145., 146., 147., 148., 149., 150.]]), device_mesh=DeviceMesh('cpu', [0, 1]), placements=(Shard(dim=0),)), DTensor(local_tensor=tensor([
	[101., 102., 103., 104., 105.],
	[111., 112., 113., 114., 115.],
	[121., 122., 123., 124., 125.],
	[131., 132., 133., 134., 135.],
	[141., 142., 143., 144., 145.],
	[151., 152., 153., 154., 155.],
	[161., 162., 163., 164., 165.],
	[171., 172., 173., 174., 175.],
	[181., 182., 183., 184., 185.],
	[191., 192., 193., 194., 195.]]), device_mesh=DeviceMesh('cpu', [0, 1]), placements=(Shard(dim=1),))]

# net1 (Colwise Parallel)
Input=== from: (Replicate(),) to: (Replicate(),)
Output=== from: (Shard(dim=1),) to: (Shard(dim=-1),)
# net2 (Rowwise Parallel)
Input=== from: (Shard(dim=-1),) to: (Shard(dim=-1),)
Output=== from: (Partial(sum),) to: (Replicate(),)
[rank0]:W1024 06:54:08.024000 46430 torch/distributed/tensor/_redistribute.py:202] redistribute from P to R on mesh dim 0
```
1. The first 2 matrices are weight shards on rank 0. net1 was Shard(0) and is 5x10. net2 is Shard(1) and is 10x5.
2. net1: Input and output transformations are as before
3. net2: Input does not require redistribute. Output of Shard x Shard is Partial as per previous blog. Desired output is Replicate, so redistribute is needed.

## Option 2:

Let's try and mix things up by instead adding another Linear module that is Colwise sharded. I initially did the same as before, just declaring net2 also as ColwiseParallel, expecting it to just work. But it complained that the matrix multiplication is (8x5) @ (10x10) and therefore incorrect. net1's output is `Shard(-1)` and net2's expected input is `Replicate` so I assumed it would do the needful redistribution. However, `input_layout` argument needs to be manually specified with the actual current input's layout. As an aside, I feel this could be somehow automated instead of expecting the user to give the correct layout. 

```
model = parallelize_module(
    model,
    mesh,
    {
        "net1": ColwiseParallel(),
        "net2": ColwiseParallel(input_layouts=Shard(-1)),
    }
)
```

```
Global rank: 0, Weights array: [DTensor(local_tensor=tensor([
	[101., 102., 103., 104., 105., 106., 107., 108., 109., 110.],
	[111., 112., 113., 114., 115., 116., 117., 118., 119., 120.],
	[121., 122., 123., 124., 125., 126., 127., 128., 129., 130.],
	[131., 132., 133., 134., 135., 136., 137., 138., 139., 140.],
	[141., 142., 143., 144., 145., 146., 147., 148., 149., 150.]]), device_mesh=DeviceMesh('cpu', [0, 1]), placements=(Shard(dim=0),)), DTensor(local_tensor=tensor([
	[101., 102., 103., 104., 105., 106., 107., 108., 109., 110.],
	[111., 112., 113., 114., 115., 116., 117., 118., 119., 120.],
	[121., 122., 123., 124., 125., 126., 127., 128., 129., 130.],
	[131., 132., 133., 134., 135., 136., 137., 138., 139., 140.],
	[141., 142., 143., 144., 145., 146., 147., 148., 149., 150.]]), device_mesh=DeviceMesh('cpu', [0, 1]), placements=(Shard(dim=0),))]
# Input, output for net1
Input=== from: (Replicate(),) to: (Replicate(),)
Output=== from: (Shard(dim=1),) to: (Shard(dim=-1),)

# Input, output for net 2
Input=== from: (Shard(dim=-1),) to: (Replicate(),)
[rank0]:W1024 06:58:20.513000 46578 torch/distributed/tensor/_redistribute.py:202] redistribute from S(1) to R on mesh dim 0
Output=== from: (Shard(dim=1),) to: (Shard(dim=-1),)
```

1. Both input weight shards are 5x10.
2. net1: output and desired output are same
3. net2: input is actually Shard(-1), while desired for ColwiseParallel is actually Replicate. So redistribute is forced.

Finally, this also explains how composability is achieved. Earlier I thought there would be an FSDP forward pass and a TP forward pass, leading to the question of which one needs to be applied. Now I understand, the forward pass is that of the Module. FSDP and TP only introduce pre-forward and post-forward hooks for doing stuff before running the Module. In case of FSDP, this would be bringing together the shards to recompose the entire module. In case of TP, it could be sharding the input to match the sharding of the module. For composing FSDP and TP, all that happens is that the pre-forward of FSDP would run, followed by the pre-forward of TP.

Misc points:
1. `parallelize_module` call: takes only a 1D mesh. If the mesh is bigger, you only pass the slice needed. But what if 2D tensor parallelism (or does no such concept exist?).
