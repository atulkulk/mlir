# GPU Dialect

Note: this dialect is more likely to change than others in the near future; use
with caution.

This dialect provides middle-level abstractions for launching GPU kernels
following a programming model similar to that of CUDA or OpenCL. It provides
abstractions for kernel invocations (and may eventually provide those for device
management) that are not present at the lower level (e.g., as LLVM IR intrinsics
for GPUs). Its goal is to abstract away device- and driver-specific
manipulations to launch a GPU kernel and provide a simple path towards GPU
execution from MLIR. It may be targeted, for example, by DSLs using MLIR. The
dialect uses `gpu` as its canonical prefix.

## Operations

### `gpu.launch`

Launch a kernel on the specified grid of thread blocks. The body of the kernel
is defined by the single region that this operation contains. The operation
takes at least six operands, with first three operands being grid sizes along
x,y,z dimensions, the following three arguments being block sizes along x,y,z
dimension, and the remaining operands are arguments of the kernel. When a
lower-dimensional kernel is required, unused sizes must be explicitly set to
`1`.

The body region has at least _twelve_ arguments, grouped as follows:

-   three arguments that contain block identifiers along x,y,z dimensions;
-   three arguments that contain thread identifiers along x,y,z dimensions;
-   operands of the `gpu.launch` operation as is, including six leading operands
    for grid and block sizes.

Operations inside the body region, and any operations in the nested regions, are
_not_ allowed to use values defined outside the _body_ region, as if this region
was a function. If necessary, values must be passed as kernel arguments into the
body region. Nested regions inside the kernel body are allowed to use values
defined in their ancestor regions as long as they don't cross the kernel body
region boundary.

Custom syntax for this operation is currently not available.

Example:

```mlir {.mlir}
// Generic syntax explains how the pretty syntax maps to the IR structure.
"gpu.launch"(%cst, %cst, %c1,  // Grid sizes.
                    %cst, %c1, %c1,   // Block sizes.
                    %arg0, %arg1)     // Actual arguments.
    {/*attributes*/}
    // All sizes and identifiers have "index" size.
    : (index, index, index, index, index, index, f32, memref<?xf32, 1>) -> () {
// The operation passes block and thread identifiers, followed by grid and block
// sizes, followed by actual arguments to the entry block of the region.
^bb0(%bx : index, %by : index, %bz : index,
     %tx : index, %ty : index, %tz : index,
     %num_bx : index, %num_by : index, %num_bz : index,
     %num_tx : index, %num_ty : index, %num_tz : index,
     %arg0 : f32, %arg1 : memref<?xf32, 1>):
  "some_op"(%bx, %tx) : (index, index) -> ()
  %3 = "std.load"(%arg1, %bx) : (memref<?xf32, 1>, index) -> f32
}
```

Rationale: using operation/block arguments gives analyses a clear way of
understanding that a value has additional semantics (e.g., we will need to know
what value corresponds to threadIdx.x for coalescing). We can recover these
properties by analyzing the operations producing values, but it is easier just
to have that information by construction.