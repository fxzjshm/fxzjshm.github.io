---
layout: post
title: Test AdaptiveCpp on Rusticl
date: 2025-01-06
author: Fxzjshm
category: Linux
tags: [Linux]
---

## Intro

Recently Rusticl has implemented shared virtual memory (SVM),
see [Karol Herbst's post](https://chaos.social/@karolherbst/113766937621167352) & [related Phoronix news](https://www.phoronix.com/news/Rusticl-Cross-Vendor-SVM).

This enables SYCL on Rusticl on e.g. RadeonSI, which may be compared to SYCL on ROCm; before that, let's take a first-time look.

Although for now Rusticl on RadeonSI has only coarse-grained SVM but  AdaptiveCpp requires fine-grained SVM (author thought it isn't too strict, see [AdaptiveCpp#1113](https://github.com/AdaptiveCpp/AdaptiveCpp/issues/1113)), it is possible to emulate USM upon coarse-grained SVM using [intel/opencl-intercept-layer](https://github.com/intel/opencl-intercept-layer).


## Build

Firstly, download source code.

```bash
git clone https://gitlab.freedesktop.org/karolherbst/mesa.git -b rusticl/svm/coarse
cd mesa
```

Build MESA with Rusticl enabled:

```bash
meson setup build -Dprefix="/media/build-cache/mesa-install" -Dgallium-rusticl=true -Dllvm=enabled -Dgallium-opencl=icd -Drust_std=2021 -Dgallium-drivers="radeonsi,virgl,svga,swrast,iris,crocus,i915,zink"
ninja
ninja install
```

Note, for now MESA requires LLVM 19, and meson may select corrent Clang but bindgen might not, reported at [mesa/mesa#11869](https://gitlab.freedesktop.org/mesa/mesa/-/issues/11869#note_2726412).

Then build AdaptiveCpp and its tests, and opencl-intercept-layer.

```bash
git clone https://github.com/AdaptiveCpp/AdaptiveCpp
cd AdaptiveCpp
mkdir build
cd build
cmake .. -DCMAKE_INSTALL_PREFIX=/opt/AdaptiveCpp -DCMAKE_BUILD_TYPE=RelWithDebInfo
make install -j`nproc`
cd ../..

git clone https://github.com/intel/opencl-intercept-layer
cd opencl-intercept-layer
mkdir build
cd build
cmake .. -DCMAKE_INSTALL_PREFIX=/opt/opencl-intercept-layer -DCMAKE_BUILD_TYPE=RelWithDebInfo
make install -j`nproc`
cd ..

mkdir acpp-tests
cmake ../AdaptiveCpp/tests/ -DAdaptiveCpp_DIR=/opt/AdaptiveCpp/lib/cmake/AdaptiveCpp/ -DCMAKE_BUILD_TYPE=Debug
make -j`nproc`
```


## Run

For runtime, set some environment variables to enable Rusticl:

```bash
export RUSTICL_ENABLE=radeonsi
# or
export RUSTICL_ENABLE=llvmpipe
```

Note, enabling devices without SVM (e.g. zink) may let test program fail on SVM allocation.

Next, set device visibility of AdaptiveCpp:

```bash
export ACPP_VISIBILITY_MASK="ocl:0"
```

where the number can be inferred by e.g. `clinfo` (not always precise, though). Some programs may print device name on execution, which can be used to verify if it really runs on Rusticl.

Then enable USM emulation w/o logging:

```bash
export LD_PRELOAD=/opt/opencl-intercept-layer/lib/libOpenCL.so                                      
export CLI_Emulate_cl_intel_unified_shared_memory=1
export CLI_SuppressLogging=1
```

Then just run the test!

```bash
./sycl_tests
```

It will print which device is default on middle of execution:

```
Default-selected queue runs on device: AMD Radeon Graphics (radeonsi, rembrandt, LLVM 19.1.6, DRM 3.59, 6.13.0-rc3-rust)
# or like
Default-selected queue runs on device: llvmpipe (LLVM 19.1.6, 256 bits)
```

and massive amount of warnings related to SPIR-V:

```
SPIR-V WARNING:
    In file home/fxzjshm/workspace/mesa/src/compiler/spirv/vtn_cfg.c:119
    Function parameter Decoration not handled: SpvFunctionParameterAttributeNoCapture
    1880 bytes into the SPIR-V binary
```

---

## Some preliminary results (to be debugged)

### Missing feature when calling llvm-spirv

* `SPV_EXT_shader_atomic_float_add` in `atomic_tests/fetch_op`

```
RequiresExtension: Feature requires the following SPIR-V extension:
 SPV_EXT_shader_atomic_float_add
[AdaptiveCpp Error] from /home/fxzjshm/workspace/hipSYCL/include/hipSYCL/glue/llvm-sscp/jit.hpp:320 @ compile(): jit::compile: Encountered errors:
0: LLVMToSpirv: llvm-spirv invocation failed with exit code 19

unknown location(0): fatal error: in "atomic_tests/fetch_op": hipsycl::sycl::exception: from /home/fxzjshm/workspace/hipSYCL/include/hipSYCL/glue/llvm-sscp/jit.hpp:320 @ compile(): jit::compile: Encountered errors:
0: LLVMToSpirv: llvm-spirv invocation failed with exit code 19
```

* `SPV_INTEL_subgroups` in `group_functions_tests/subgroup_shuffle_like<{char, unsigned int, float, double}>`

```
RequiresExtension: Feature requires the following SPIR-V extension:
 SPV_INTEL_subgroups
[AdaptiveCpp Error] from /home/fxzjshm/workspace/hipSYCL/include/hipSYCL/glue/llvm-sscp/jit.hpp:320 @ compile(): jit::compile: Encountered errors:
0: LLVMToSpirv: llvm-spirv invocation failed with exit code 19

unknown location(0): fatal error: in "group_functions_tests/subgroup_shuffle_like<char>": signal: integer divide by zero; address of failing instruction: 0x5fa617e2519a
/home/fxzjshm/workspace/hipSYCL/tests/sycl/group_functions/group_functions_misc.cpp(513): last checkpoint
```

### extension_tests/cg_property_retarget (radeonsi)
```
/home/fxzjshm/workspace/hipSYCL/tests/sycl/extensions.cpp(537): error: in "extension_tests/cg_property_retarget": check ptr[0] == 2 has failed [0 != 2]
```

### extension_tests/prefetch_host (radeonsi)

```
/home/fxzjshm/workspace/hipSYCL/tests/sycl/extensions.cpp(694): error: in "extension_tests/prefetch_host": check shared_mem[i] == i + 1 has failed [0 != 1]
...
/home/fxzjshm/workspace/hipSYCL/tests/sycl/extensions.cpp(694): error: in "extension_tests/prefetch_host": check shared_mem[i] == i + 1 has failed [4095 != 4096]
```

### extension_tests/buffers_over_usm_pointers (radeonsi)

```
/home/fxzjshm/workspace/hipSYCL/tests/sycl/extensions.cpp(808): error: in "extension_tests/buffers_over_usm_pointers": check alloc1[i] == i has failed
/home/fxzjshm/workspace/hipSYCL/tests/sycl/extensions.cpp(830): error: in "extension_tests/buffers_over_usm_pointers": check alloc2[i] == i has failed
```

### group_functions_tests/* (radeonsi, llvmpipe)

```
/home/fxzjshm/workspace/hipSYCL/tests/sycl/group_functions/group_functions.hpp(241): error: in "group_functions_tests/group_x_of_local": 60:0 at position 0 instead of 1 for case: everything true all_of
/home/fxzjshm/workspace/hipSYCL/tests/sycl/group_functions/group_functions.hpp(241): error: in "group_functions_tests/sub_group_x_of_local": 129:0 at position 0 instead of 1 for case: everything true all_of bool sub group
/home/fxzjshm/workspace/hipSYCL/tests/sycl/group_functions/group_functions.hpp(241): error: in "group_functions_tests/group_x_of_ptr_function": 208:0 at position 0 instead of 1 for case: everything false all_of
/home/fxzjshm/workspace/hipSYCL/tests/sycl/group_functions/group_functions.hpp(241): error: in "group_functions_tests/group_x_of_function": 288:0 at position 0 instead of 1 for case: everything false all_of
/home/fxzjshm/workspace/hipSYCL/tests/sycl/group_functions/group_functions.hpp(241): error: in "group_functions_tests/sub_group_x_of_function": 360:0 at position 0 instead of 1 for case: everything false all_of
/home/fxzjshm/workspace/hipSYCL/tests/sycl/group_functions/group_functions_reduce.cpp(47): error: in "group_functions_tests/group_reduce_mul<char>": 0 at position 0 instead of 2 for group 0 for local_size 25 and case: no init multiplication
/home/fxzjshm/workspace/hipSYCL/tests/sycl/group_functions/group_functions_reduce.cpp(47): error: in "group_functions_tests/group_reduce_mul<unsigned int>": 0 at position 0 instead of 2 for group 0 for local_size 25 and case: no init multiplication
/home/fxzjshm/workspace/hipSYCL/tests/sycl/group_functions/group_functions_reduce.cpp(47): error: in "group_functions_tests/group_reduce_mul<float>": 0 at position 0 instead of 2 for group 0 for local_size 25 and case: no init multiplication
/home/fxzjshm/workspace/hipSYCL/tests/sycl/group_functions/group_functions_reduce.cpp(47): error: in "group_functions_tests/group_reduce_mul<double>": 0 at position 0 instead of 2 for group 0 for local_size 25 and case: no init multiplication
/home/fxzjshm/workspace/hipSYCL/tests/sycl/group_functions/group_functions_reduce.cpp(47): error: in "group_functions_tests/group_reduce_mul<hipsycl__sycl__vec<int_ 2_ hipsycl__sycl__detail__vec_storage<int_ 2>>>": (0, 0) at position 0 instead of (2, 2) for group 0 for local_size 25 and case: no init multiplication
```

### InvalidBitWidth: Invalid bit width in input: 48

```
InvalidBitWidth: Invalid bit width in input: 48
[AdaptiveCpp Error] from /home/fxzjshm/workspace/hipSYCL/include/hipSYCL/glue/llvm-sscp/jit.hpp:320 @ compile(): jit::compile: Encountered errors:
0: LLVMToSpirv: llvm-spirv invocation failed with exit code 10
unknown location(0): fatal error: in "marray_tests/marray_ops<short>": signal: SIGABRT (application abort requested)
```

### reduction_tests/* (radeonsi)

```
/home/fxzjshm/workspace/hipSYCL/tests/sycl/reduction.cpp(44): error: in "reduction_tests/single_kernel_single_scalar_reduction<char>": check expected_result == *output_data has failed [0xffffffc0 != 0]
/home/fxzjshm/workspace/hipSYCL/tests/sycl/reduction.cpp(44): error: in "reduction_tests/single_kernel_single_scalar_reduction<unsigned int>": check expected_result == *output_data has failed [8128 != 0]
/home/fxzjshm/workspace/hipSYCL/tests/sycl/reduction.cpp(44): error: in "reduction_tests/single_kernel_single_scalar_reduction<int>": check expected_result == *output_data has failed [8128 != 0]
/home/fxzjshm/workspace/hipSYCL/tests/sycl/reduction.cpp(44): error: in "reduction_tests/single_kernel_single_scalar_reduction<long long>": check expected_result == *output_data has failed [8128 != 0]
/home/fxzjshm/workspace/hipSYCL/tests/sycl/reduction.cpp(42): error: in "reduction_tests/single_kernel_single_scalar_reduction<float>": check expected_result == *output_data has failed [8128 != 0]: absolute value exceeds tolerance [|8128| > 0.001]
/home/fxzjshm/workspace/hipSYCL/tests/sycl/reduction.cpp(42): error: in "reduction_tests/single_kernel_single_scalar_reduction<double>": check expected_result == *output_data has failed [8128 != 0]
/home/fxzjshm/workspace/hipSYCL/tests/sycl/reduction.cpp(44): error: in "reduction_tests/two_kernels_single_scalar_reduction<unsigned int>": check expected_result == *output_data has failed [134209536 != 0]
...
```

### usm_tests/allocations_in_kernels (radeonsi)

```
/home/fxzjshm/workspace/hipSYCL/tests/sycl/usm.cpp(224): error: in "usm_tests/allocations_in_kernels": check shared_allocation[i] == i + 3 has failed [0 != 3]
/home/fxzjshm/workspace/hipSYCL/tests/sycl/usm.cpp(226): error: in "usm_tests/allocations_in_kernels": check mapped_host_allocation[i] == i + 3 has failed [0 != 3]
```

### usm_tests/memcpy (radeonsi)

```
/home/fxzjshm/workspace/hipSYCL/tests/sycl/usm.cpp(282): error: in "usm_tests/memcpy": check shared_mem[i] == initial_data[i] has failed [0 != 1]
...
/home/fxzjshm/workspace/hipSYCL/tests/sycl/usm.cpp(312): error: in "usm_tests/memcpy": check host_mem2[i] == initial_data[i] has failed [0 != 1]
...
/home/fxzjshm/workspace/hipSYCL/tests/sycl/usm.cpp(347): error: in "usm_tests/memcpy": check shared_mem2[i] == initial_data[i] has failed [0 != 1]
```

### usm_tests/usm_fill (radeonsi)

```
/home/fxzjshm/workspace/hipSYCL/tests/sycl/usm.cpp(369): error: in "usm_tests/usm_fill": check shared_mem[i] == fill_value has failed [0 != 1234567890]
```

### usm_tests/memset (radeonsi, llvmpipe)

```
[AdaptiveCpp Error] from /home/fxzjshm/workspace/hipSYCL/src/runtime/ocl/ocl_queue.cpp:331 @ submit_memset(): ocl_queue: enqueuing memset failed (error code = CL:-30)
/home/fxzjshm/workspace/hipSYCL/tests/sycl/usm.cpp(389): error: in "usm_tests/memset": check host_mem[i] == 0 has failed [0x8f != 0]
...
```

### usm_tests/prefetch (radeonsi)

```
/home/fxzjshm/workspace/hipSYCL/tests/sycl/usm.cpp(419): error: in "usm_tests/prefetch": check shared_mem[i] == i + 1 has failed [0 != 1]
```
