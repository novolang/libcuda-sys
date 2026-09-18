# Changelog

All notable changes to libcuda-sys are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.1.1 — 2026-09-18

The documentation and comments in plain prose; no declaration changed.

### Corrected against the CUDA runtime API reference

- `cudaGetErrorString` and `cudaGetErrorName` answer the address of a C
  string. Every other entry point answers a `cudaError_t`.
- `cudaGetDeviceCount` answers `cudaErrorNoDevice` on a machine the
  runtime finds no card on, and `cudaErrorInsufficientDriver` when the
  driver is older than the toolkit.
- `cudaMemcpyKind` defines `cudaMemcpyHostToHost` as 0. Any value
  outside the five answers `cudaErrorInvalidMemcpyDirection`.
- `cudaDeviceSynchronize` answers a failure when one of the preceding
  tasks failed. It does not promise the last one.
- `cudaMalloc` writes the address into a slot the caller supplies,
  where `malloc` answers the address directly.

## 0.1.0 — 2026-09-15

The first release: thirteen entry points of the CUDA runtime library,
one `@ffi` declaration each, and no logic.

### Added

- `libcuda` — the whole surface, in four groups.
  - The device: `cuda_get_device_count`, `cuda_set_device` and
    `cuda_get_device`.
  - Memory: `cuda_malloc`, `cuda_free`, `cuda_memcpy`, `cuda_memset`
    and `cuda_mem_get_info`.
  - Synchronisation: `cuda_device_synchronize`.
  - Errors: `cuda_get_last_error`, `cuda_peek_at_last_error`,
    `cuda_get_error_string` and `cuda_get_error_name`.
- `tests/libcuda_tests.nv` — nine tests over the thirteen entry points.
  Each accepts both a machine with a card and a machine without one.

### Where the surface came from

The inference programs in this repository make eight runtime calls:
`cudaMalloc`, `cudaFree`, `cudaMemcpy`, `cudaMemset`,
`cudaGetDeviceCount`, `cudaSetDevice`, `cudaGetLastError` and
`cudaDeviceSynchronize`. Five more are here for three reasons. An error
code cannot be reported without a name and a description. The read half
of `cudaSetDevice` belongs beside the write half. Whether the weights
fit is the question asked before the first upload.

### The wrapped library is the runtime, not the driver

`wraps = "libcudart"`. `libcuda.so` is the CUDA driver, a different
library with a different C API, and none of these symbols are in it.
The package's name follows the naming convention for a bindings package
rather than the file name of the library it binds.

### Named as missing

**A way to run a kernel.** This package moves data to a card and back.
The computation itself is written in CUDA C and compiled by `nvcc`, and
there is no novo-lang path to one. Everything here is the half of a GPU
program that novo-lang can express today.

**Passing a structure by value.** `cudaGetDeviceProperties`,
`cudaStream_t` and `cudaEvent_t` all need one, so the properties call,
the asynchronous copies and the timing measurements are absent.
