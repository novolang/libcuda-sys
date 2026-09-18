# libcuda-sys

CUDA is NVIDIA's platform for running general-purpose computation on a
graphics card. The CUDA runtime library is the C API a program uses to
select a card, allocate memory on it, copy data to and from it, and
wait for the work to finish. It is documented in
[the CUDA Runtime API reference](https://docs.nvidia.com/cuda/cuda-runtime-api/).
This package declares thirteen of that library's entry points to
novo-lang, one declaration each.

Every function here is a declaration of a function in the CUDA runtime.
The package contains no logic of its own, and it does nothing without
the CUDA toolkit installed. The thirteen entry points are the ones a
program needs to move data to a card and back. The section "What is not
included" says what a program cannot do with them alone.

## What it is

A graphics card is a separate computer. It has its own memory, its own
processors, and no access to the memory of the machine it is plugged
into. Running a computation on it means three steps: copy the input into
the card's memory, run the computation, and copy the result back. This
package is the first and third steps.

**Device memory** is the card's memory. `cuda_malloc` reserves a block
of it and writes the address into a slot the caller supplies, where
`malloc` answers the address directly. The address means nothing to the
processor running the program. Only the card can read it, and only
through the runtime.

A **copy** moves bytes between the two memories. `cuda_memcpy` takes a
destination address, a source address, a byte count and a direction. The
direction is the fourth argument. Getting it wrong is the common
mistake, because the runtime cannot tell a host address from a device
address by looking at it.

The runtime is **asynchronous** in places. A computation the card is
asked to run returns to the program immediately, and the program finds
out whether it succeeded later. `cuda_device_synchronize` waits for
everything outstanding, and `cuda_get_last_error` reports the failure
of something that has already returned.

An **error** is an integer. Zero is success. Every other value has a
name, such as `cudaErrorMemoryAllocation`, and a sentence of
description, and the runtime supplies both.

The CUDA toolkit ships two libraries with two different C APIs. This
package binds the **runtime**, `libcudart.so`, whose symbols begin
`cuda`. The
other is the **driver**, `libcuda.so`, whose symbols begin `cu`. They
are not interchangeable and none of the symbols below are in the driver.

## Install

```
novo pkg add libcuda-sys
```

Adding the package does not install the C library. On Debian and Ubuntu
the runtime comes from the CUDA toolkit:

```
sudo apt install nvidia-cuda-toolkit
```

NVIDIA's own installer puts the toolkit in `/usr/local/cuda`, where the
linker does not look by default. A program that links against a toolkit
installed that way adds the directory to its own link flags.

A card and a driver are separate from the toolkit. A machine with the
toolkit and no card links and runs, and every call answers a failure
code saying there is no device.

## Example

Eight bytes to the card and back:

```novo ignore
use libcuda

fn main() [io, ffi]
    let count_slot = ptr.alloc_word()
    if libcuda.cuda_get_device_count(count_slot) != 0 or ptr.read_word(count_slot) == 0
        println("no CUDA device")
        return

    let dev_slot = ptr.alloc_word()
    let host_in = ptr.alloc_word()
    let host_out = ptr.alloc_word()
    ptr.write_word(host_in, 42)

    let rc = libcuda.cuda_malloc(dev_slot, 8)
    if rc != 0
        println(ptr.read_str(libcuda.cuda_get_error_string(rc)))
        return
    let dev = ptr.read_word(dev_slot)

    // 1 is host to device, 2 is device to host.
    let _up = libcuda.cuda_memcpy(dev, host_in, 8, 1)
    let _down = libcuda.cuda_memcpy(host_out, dev, 8, 2)
    println("the card gave back ${ptr.read_word(host_out)}")

    let _freed = libcuda.cuda_free(dev)
    ptr.free(dev_slot)
    ptr.free(host_in)
    ptr.free(host_out)
```

The example is fenced as an illustration rather than a compiled block
because it needs the CUDA toolkit, which this repository does not ship.

## What the package contains

| Module | Contents |
| --- | --- |
| `libcuda` | Every entry point: the device calls, the memory calls, the copy, the barrier and the error calls. |

The four groups:

| Group | Entry points | What it does |
| --- | --- | --- |
| Device | 3 | Counts the cards, chooses one for the calling thread, and says which one is chosen. |
| Memory | 5 | Allocates and frees device memory, copies between the two memories, clears a block, and reports how much memory is left. |
| Synchronisation | 1 | Waits for everything the card has been asked to do. |
| Errors | 4 | Reads the last error, peeks at it, and turns a code into a name and a sentence. |

## The rules a user needs

1. **A device address is an `Int`, and the program may not read it.**
   `cuda_malloc` answers an address in the card's memory. Passing it to
   anything but a CUDA call reads memory that does not belong to the
   program.
2. **An out-parameter is the address of a caller-owned slot.**
   `cuda_malloc`, `cuda_get_device_count`, `cuda_get_device` and
   `cuda_mem_get_info` each write into a slot the caller supplies.
   `ptr.alloc_word` returns the address of an eight-byte slot, which is
   wide enough for all of them, and `ptr.free` releases it.
3. **Zero is success and every other code is a failure.** The name of
   the code comes from `cuda_get_error_name` and the sentence from
   `cuda_get_error_string`. Both answer the address of a C string the
   runtime owns; read it with `ptr.read_str` and do not free it.
4. **The copy direction is an argument, and it is not checked.** 0 is
   host to host, 1 is host to device, 2 is device to host, 3 is device
   to device, and 4 asks the runtime to work it out from the two
   addresses. The runtime
   cannot tell the two kinds of address apart by inspection, so 1 with
   the arguments the wrong way round is a fault rather than a message.
5. **A byte count is a byte count.** Neither side checks the length of
   a buffer, so a copy that names more bytes than the source holds
   reads past it.
6. **The device belongs to the thread, not to the program.**
   `cuda_set_device` chooses the card for the calling thread only. Every
   allocation and every copy after it belongs to that card, and an
   address from one card is not valid on another.
7. **A failure may arrive later than the call that caused it.** A card
   runs work asynchronously, so `cuda_device_synchronize` and
   `cuda_get_last_error` are where a fault surfaces.
   `cuda_get_last_error` clears the error; `cuda_peek_at_last_error`
   leaves it.

## What is not included

- **Running anything.** A CUDA computation is a *kernel*, written in
  CUDA C and compiled by NVIDIA's `nvcc`. There is no novo-lang syntax
  for one and no way to write one through this package. A program that
  needs a kernel compiles it with `nvcc` and calls it through its own
  foreign declaration. This package moves the data the kernel reads.
- **Streams and events.** `cudaStream_t` and `cudaEvent_t` are opaque
  handles the runtime passes by value, and the calls that take them are
  the asynchronous copy, the asynchronous launch, and the timing
  measurements. They are left out until there is a consumer that needs
  overlapping work.
- **The linear algebra library.** `cublasSgemm` and its neighbours are
  in `libcublas`, a second library. A binding package declares exactly
  one library, so cuBLAS belongs in a package of its own.
- **Device properties.** `cudaGetDeviceProperties` fills in a structure
  of some six hundred bytes. The novo-lang foreign function interface
  passes integers, floats and strings, so a structure has to be laid out
  by hand, and this one changes shape between toolkit versions.
  `cuda_mem_get_info` answers the one property most programs ask for.
- **Managed and pinned host memory.** `cudaMallocManaged` and
  `cudaHostAlloc` change how the two memories relate to each other, and
  a program that wants them wants the streams as well.
- **The driver API.** `cuInit`, `cuMemAlloc` and the rest are in
  `libcuda.so`. They are a lower-level C API for the same hardware, and
  a package that bound them would be a different package.

## Related packages

`libhip-sys` is the same shape for AMD cards. Its library is the HIP
runtime, and its entry points are the same thirteen operations under
different names.

`std.gpu` in the standard library reaches a graphics card through
wgpu-native, and runs on NVIDIA, AMD, Intel and Apple hardware alike.
It is the right choice for a program that wants a graphics card rather
than an NVIDIA one, and it installs nothing by hand.

There is no novo-lang replacement for this package and none is planned.
A graphics card is a piece of hardware whose published C API is the
vendor's own library. There is no format to reimplement.

## Tests

`tests/libcuda_tests.nv` holds nine tests over the thirteen entry
points:

```
novo test tests/libcuda_tests.nv
```

The suite links against the CUDA runtime, so it needs the toolkit
installed. Without it the link fails, naming `-lcudart`. `novo pkg
build` type-checks the declarations and needs nothing installed.

The tests do not need a card. Each one accepts both answers. On a
machine with a card the successful path is asserted to be
self-consistent, and on a machine without one the failure is asserted to
carry a name. The round trip through device memory writes eight bytes
up, reads them back, clears them and reads them back again.

## Licence

Apache-2.0. See [LICENSE](LICENSE).

The CUDA toolkit is distributed under NVIDIA's own licence, and
installing it is the reader's own step.
