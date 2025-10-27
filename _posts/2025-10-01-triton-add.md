---
layout: post
title: Multi-dimensional operations in Triton
description: A different look at matrix addition in Triton
img:
date: 2025-10-01  +1000
---

Vector addition in Triton is the most basic example out in the wild. By reading and understanding it, one can understand how a Triton program works in terms of the offset for loading and storing and masks etc.

See this link for the official vector add example: https://triton-lang.org/main/getting-started/tutorials/01-vector-add.html

Confident that I understood the basics of Triton, I decided to look at Matrix Multiplication in Triton and found my foundations crumbling. Turns out, there's a huge step from 1D vector addition to 2D matrix multiplication. 

To bridge this gap, I have tried 2D matrix addition to understand how the dimensions add complexity to even the simple load and store operations.

We are going to do super simple matrix addition and play with the grid sizes a bit. Then, we'll add tiling and see how that changes things.

## 2D Matrix simple addition

This is common for all:
```
import triton
import triton.language as tl

DEVICE = triton.runtime.driver.active.get_active_torch_device()
```

Let's look at the kernel and the caller function:
```
@triton.jit
def simpleAdd(M, N, A_ptr, B_ptr, out_ptr):
    id = tl.program_id(axis=1)
    xid = id // N
    yid = id % N

    a = tl.load(A_ptr + xid * N + yid)
    b = tl.load(B_ptr + xid * N + yid)

    out = a + b

    tl.store(out_ptr + xid * N + yid, out)

def addition(A, B, M, N):
    C = torch.empty_like(A)
    bsize = M
    grid = (1, M * N)

    simpleAdd[grid](M, N, A, B, C)

    return C
```

Here we've created a grid of 1 block with M*N threads. Each thread is responsible for adding a single element. Therefore, `tl.program_id(axis=0)` is 0 for all the threads and ranges between 0 and `M.N` over axis 1. 

`a` and `b` are the values it loads using the indices. We need to specify the exact address by using both the x value and the y value. Ideally we don't need to split the xid and yid, seeing that we are reconstructing it again, but it helps to know, especially when doing more complex operations such as matrix multiplication.

To run, we use this common runner code:
```
M = 100
N = 100

A = torch.rand(M, N, device=DEVICE)
B = torch.rand(M, N, device=DEVICE)

out = addition(A, B, M, N)
act = A + B
print(torch.allclose(out, act))
```

A useful tip to debug the programs is using `TRITON_INTERPRET='1'` flag when running the code which runs on the CPU and allows print statements etc.

Now, let's see what changes to make when the grid is changed to M*N, 1 size.

First, the grid is changed to:
```
grid = (M * N, 1)
```

Secondly, the kernel now reads the id from axis 0 `id = tl.program_id(axis=0)`

Finally, the last simple addition, is with a M*N grid.

Again, we change the grid statement to `grid = (M, N)`.

Next, we read both the x and y ids from the program id (instead of deriving it by division)

```
    xid = tl.program_id(axis=0)
    yid = tl.program_id(axis=1)
```


Full code:

```python
import torch
import triton
import triton.language as tl

DEVICE = triton.runtime.driver.active.get_active_torch_device()

@triton.jit
def longGridAdd(M, N, A_ptr, B_ptr, out_ptr):
    id = tl.program_id(axis=1)
    xid = id // N
    yid = id % N

    a = tl.load(A_ptr + xid * N + yid)
    b = tl.load(B_ptr + xid * N + yid)

    out = a + b

    tl.store(out_ptr + xid * N + yid, out)

@triton.jit
def shortGridAdd(M, N, A_ptr, B_ptr, out_ptr):
    id = tl.program_id(axis=0)
    xid = id // N
    yid = id % N

    a = tl.load(A_ptr + xid * N + yid)
    b = tl.load(B_ptr + xid * N + yid)

    out = a + b

    tl.store(out_ptr + xid * N + yid, out)

@triton.jit
def squareGridAdd(M, N, A_ptr, B_ptr, out_ptr):
    xid = tl.program_id(axis=0)
    yid = tl.program_id(axis=1)

    a = tl.load(A_ptr + xid * N + yid)
    b = tl.load(B_ptr + xid * N + yid)

    out = a + b

    tl.store(out_ptr + xid * N + yid, out)

def addition(A, B, M, N):
    C = torch.empty_like(A)
    bsize = M
    longGrid = (1, M * N)
    shortGrid = (M * N, 1)
    squareGrid = (M, N)

    longGridAdd[longGrid](M, N, A, B, C)

    return C

M = 100
N = 100

A = torch.rand(M, N, device=DEVICE)
B = torch.rand(M, N, device=DEVICE)

out = addition(A, B, M, N)
act = A + B
print(torch.allclose(out, act))
```
I timed the three different runs and they seem to take roughly the same time, especially for the small matrix dimension that we're dealing with.

When I try increasing the matrix size, I run into this error: `RuntimeError: Triton Error [CUDA]: invalid argument`, so clearly there is a limit to the number of threads that can be created.

We have a simple example here where every thread works on a single element of the matrix. Now let's add tiling to make each thread work on a group of elements.

## Let's add Tiling/Blocking

Tiled code always seems to look nasty. We're going to add a tile dimension variables (and this is the size of the matrix itself now). This tile has to be a multiple of 2.

First, we introduce 2 new variables, Blocksize_X and Blocksize_Y. This is the block each thread will operate on. 

Let's go with the square grid that we created earlier. With each thread operating on a larger block, the grid size is now obviously different. We'll need to roughly divide the Matrix size by the block size. This is where corner cases come into the picture.

This part would look like this, where I define a block size along the 2 dimensions and calculate the number of grids and blocks as described above:
```
BS_X = 64
BS_Y = 64
grid = (triton.cdiv(M, BS_X), triton.cdiv(N, BS_Y))
```

The index calculation in the kernel is going to be a bit more complex. The goal is to load a rectangle block of size BS_X x BS_Y at each thread. We can calculate the indices of the 2d sub-block using the same calculation as before (0 ... BS_X - 1) offset by the `xid` multiplied by the width of each block.

```
xid = tl.program_id(axis=0)
yid = tl.program_id(axis=1)

ind_x = tl.arange(0, BS_X) + xid * BS_X
ind_y = tl.arange(0, BS_Y) + yid * BS_Y
```

The nastiness comes when we try to convert this into the address locations in 1D space.

Suppose the block we are addressing is as shown in the below diagram where M = 8, N = 4, BS_X = 4, BS_Y = 2.

```
x x x x x x x x
x x x x x x x x
o o o o x x x x
o o o o x x x x
```

`ind_x` is (0, 1, 2, 3) and `ind_y` is (2, 3). We need the outputs `(16, 17, 18, 19) and (24, 25, 26, 27)`.

Pytorch allows converting 1D into 2D by adding a new dimension as `[:, None]`. This particular one converts into a vertical tensor.

In the previous case, we indexed the A matrix as `A_ptr + xid * N + yid` which is equivalent to `A_ptr + xid + yid * M`. We'll now do the same math in 2D by multiplying ind_y by X dimension of the matrix. In this example, `((2), (3))` will become `((16), (24))` which should excite you because of how close it is to where we want to be. Finally, we just add the (0, 1, 2, 3) matrix to this, which makes this a rectangular matrix. We use these indices to index into A and B for loading and output for storing.

```
def blockAdd(M, N, A_ptr, B_ptr, out_ptr, BS_X: tl.constexpr, BS_Y: tl.constexpr):
    xid = tl.program_id(axis=0)
    yid = tl.program_id(axis=1)

    ind_x = tl.arange(0, BS_X) + xid * BS_X
    ind_y = tl.arange(0, BS_Y) + yid * BS_Y

    indices = ind_x[None, :] + ind_y[:, None] * M

    a = tl.load(A_ptr + indices)
    b = tl.load(B_ptr + indices)

    out = a + b

    tl.store(out_ptr + indices, out)
```

As mentioned in the triton docs, the arguments to arange are expected to be `tl.constexpr` type and power of 2. If we set BS_X to something like 100, we get this error:

```
ValueError: arange's range must be a power of 2
```

As you can see, I've not handled the over-indexing issues. Frankly it didn't throw any errors at me, so I'll not over-complicate my code with the masks and condition checks.

Complete code:

```
import torch
import triton
import triton.language as tl
import time

DEVICE = triton.runtime.driver.active.get_active_torch_device()

@triton.jit
def blockAdd(M, N, A_ptr, B_ptr, out_ptr, BS_X: tl.constexpr, BS_Y: tl.constexpr):
    xid = tl.program_id(axis=0)
    yid = tl.program_id(axis=1)

    ind_x = tl.arange(0, BS_X) + xid * BS_X
    ind_y = tl.arange(0, BS_Y) + yid * BS_Y

    indices = ind_x[None, :] + ind_y[:, None] * M

    a = tl.load(A_ptr + indices)
    b = tl.load(B_ptr + indices)

    out = a + b

    tl.store(out_ptr + indices, out)

def addition(A, B, M, N):
    C = torch.empty_like(A)
    BS_X = 128
    BS_Y = 128
    grid = (triton.cdiv(M, BS_X), triton.cdiv(N, BS_Y))

    blockAdd[grid](M, N, A, B, C, BS_X, BS_Y)

    return C

M = 4096
N = 4096

A = torch.rand(M, N, device=DEVICE)
B = torch.rand(M, N, device=DEVICE)
```


Next post, I'll look at matrix multiplication and derive the optimizations from scratch.
