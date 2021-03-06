```
                        ___                    ___          ___     
                       /\__\                  /\  \        /\  \    ______________
                      /:/  /       ___        \:\  \       \:\  \   
                     /:/  /       /\__\        \:\  \       \:\  \  ______________
                    /:/  /  ___  /:/__/    _____\:\  \  _____\:\  \ 
                   /:/__/  /\__\/::\  \   /::::::::\__\/::::::::\__\ ______________
                   \:\  \ /:/  /\/\:\  \__\:\~~\~~\/__/\:\~~\~~\/__/
                    \:\  /:/  /  ~~\:\/\__\\:\  \       \:\  \       _________________
                     \:\/:/  /      \::/  / \:\  \       \:\  \     ______________
                      \::/  /       /:/  /   \:\__\       \:\__\    ______________
                       \/__/        \/__/     \/__/        \/__/    ______________

```


# CINN : a Compiler Infrastructure for Neural Networks

[Install](./docs/install.md) | 
[Roadmap](./docs/roadmap.md)

CINN is a machine learning compiler and executor for multiple hardwares. 
It is designed to provide multiple layers of APIs to make DNN computation graph easier to define, faster to execute, and more convenient to extend with hardware backends.
Currently, it focuses on X86 CPUs and NVidia GPUs. CINN is under active maintenance. 
In the following, you are going to see how to get started, how to load a model and execute it in CINN, how CINN actually works, and its license.

##  0 Getting Started

### Compile and execute the code
To compile the CINN's code, one needs to build the docker image first:

```sh
cd tools/docker
ln -s Dockerfile.cpu Dockerfile
docker build . -t cinn-dev
```

Then start a docker container, and compile the code inside it:

```sh
# inside the docker container

# compile and install isl
sh tools/ci_build.sh

# compile the tests and python library
./build.sh ci
```

After compilation, you can launch the C++ and python tests:
```sh
cd build
ctest -V
```
### Reference the API usage
Read the code in the tests

For Python APIs, reference the code inside `python/tests`.

The C++ APIs are in `cinn/*/*_test.cc`, the high level APIs in `hlir/frontend`, and the lower level APIs in `cinn/cinn.h`.

## 1 Example

The following is an example of our C++ APIs. Similar Python APIs are also available with minor differences.

### 1.1 Load a PaddlePaddle model and execute

```c++
#include "cinn/frontend/interpreter.h"
using cinn::hlir::framework;

Interpreter inter({"input0"}/*list of inputs' names*/, 
                  {{1, 30}}/*list of inputs' shapes*/);
inter.LoadPaddleModel(/*string of model directory*/);
auto input_handle = inter.GetTensor("input0");
auto output_handle = inter.GetTensor("output0");
// set data to input_handle
inter.Run();
// get output content from output_handle
```

### 1.2 Use CINN lower level DSL to define computations and execute them

The following is a naive matrix-multiplication implementation using the CINN DSL:

```c++
#include "cinn/cinn.h"
using namespace cinn;

// Declare constants
Expr M(10), N(20), K(30);

// Declare the inputs 
auto A = Placeholder<float>("A", {M, K});
auto B = Placeholder<float>("B", {K, N});

auto k1 = Var(K.as_int32(), "k1");
auto C  = Compute(
    {M, N}, [&](Var i, Var j) { return ReduceSum(A(i, k1) * B(k1, j), {k1}); }, "C");

Target target = common::DefaultHostTarget();

int block_size = 32;

// The stages holds all the schedules for each tensors.
auto stages = CreateStages({C});

// Blocking optimization by loop tiling stragety.
auto [i_outer, i_inner, j_outer, j_inner] = stages[C]->Tile(0, 1, bn, bn);
auto [k_outer, k_inner]                   = stages[C]->Split("k0", 4);
stages[C]->Reorder({i_outer, j_outer, k_outer, k_inner, i_inner, j_inner});

// Generate C source code:
Module::Builder builder("module_block", target);
auto func = Lower("matmul_block", stages, {A, B, C});
builder.AddFunction(func);

CodeGenCX86 compiler(target, CodeGenCX86::Feature::AVX512);
Outputs outputs;
outputs = outputs.c_header("./test02_matmul_block.h").c_source("./test02_matmul_block.cc");
compiler.Compile(builder.Build(), outputs);
```

This can genrate the optimized C source code like:

```c++
void matmul_block(void* _args, int32_t num_args)
{
  const cinn_buffer_t* _A = cinn_pod_value_to_buffer_p(&(((cinn_pod_value_t*)(_args))[0]));
  const cinn_buffer_t* _B = cinn_pod_value_to_buffer_p(&(((cinn_pod_value_t*)(_args))[1]));
  cinn_buffer_t* _C = cinn_pod_value_to_buffer_p(&(((cinn_pod_value_t*)(_args))[2]));
  cinn_buffer_malloc((void*)(0), _C);
  const float* A = ((const float*)(_A->memory));
  const float* B = ((const float*)(_B->memory));
  float* C = ((float*)(_C->memory));
  float* C__reduce_init = ((float*)(_C->memory));
  for (int32_t i = 0; i < 1024; i += 1) {
    for (int32_t j = 0; j < 1024; j += 1) {
      C__reduce_init[((1024 * i) + j)] = 0;
    };
  };
  for (int32_t i_outer = 0; i_outer < 32; i_outer += 1) {
    for (int32_t j_outer = 0; j_outer < 32; j_outer += 1) {
      for (int32_t k0_outer = 0; k0_outer < 256; k0_outer += 1) {
        for (int32_t k0_inner = 0; k0_inner < 4; k0_inner += 1) {
          for (int32_t i_inner = 0; i_inner < 32; i_inner += 1) {
            for (int32_t j_inner = 0; j_inner < 32; j_inner += 1) {
              C[((1024 * i_inner) + ((32768 * i_outer) + ((32 * j_outer) + j_inner)))] = (C[((1024 * i_inner) + ((32768 * i_outer) + ((32 * j_outer) + j_inner)))] + (A[((1024 * i_inner) + ((32768 * i_outer) + ((4 * k0_outer) + k0_inner)))] * B[((32 * j_outer) + ((1024 * k0_inner) + ((4096 * k0_outer) + j_inner)))]));
            };
          };
        };
      };
    };
  };
  cinn_buffer_free((void*)(0), _C);
}
```

If you change `CodeGenCX86` usage to `CodeGenLLVM`, it will produce a LLVM JIT-compiled function instead, invoked realtime.

## 2 How it works

The CINN lowers a traditional DNN model into a two-level intermediate representation(IR), the high-level IR(HLIR) and the CINN IR.

The HLIR helps to define domain-specific computations and perform overall optimizations on the IR-graph, while the CINN IR helps to represent computations semantically and finally lower them to a hardware backend.

Both levels of IRs have the similar SSA graph, analysis, and optimization facilities.

CINN is based on the polyhedral compilation, and thus it is easy to extend with more loop optimizations.
The schedule transformation is applied between the lowering from HLIR to CINN IR.

The overall architecture is as follows:

![CINN architecture](./docs/images/cinn-architecutre.png)

### 2.1 Fundamental Concepts

There are two levels of APIs in CINN, high-level IR and the CINN IR at the lower lower. The followings are fundamental concepts in these two types of IRs.

In HLIR:

- `Primitive Emitter`(PE), encapsulates the computation of different tensor-based algorithms,
- `frontend::Executor`, the container to execute a model (of PaddlePaddle),
- `frotnend::Program`, the program helps to define a machine learning computation,
- `hlir::framework::Tensor`, multi-dimensional arrays helps to manage a memory buffer.

In CINN IR:

- `Compute`, the method to define a computation,
- `Lower`, the method to lower a computation to the corresponding IR,
- `LoweredFunc`, the function defined in CINN IR,
- `Var`, a scalar variable,
- `Expr`, an expression represents any CINN IR node(no specified Statement node),
- `Stage`, holds some schedule details of a tensor,

## 3 License

CINN is licensed under the Apache 2.0 License.
