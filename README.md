<div align="center"><img src="./docs/logo/fizzy_logo.svg" width="150px" height="150px"/></div>

# Fizzy

[![webassembly badge]][webassembly]
[![readme style standard badge]][standard readme]
[![circleci badge]][circleci]
[![codecov badge]][codecov]
[![license badge]][Apache License, Version 2.0]

Fizzy aims to be a fast, deterministic, and pedantic WebAssembly interpreter written in C++.

## Goals

I) Code quality
- [x] Clean and modern C++17 codebase without external dependencies
- [x] Easily embeddable (*and take part of the standardisation of the "C/C++ embedding API"*)

II) Simplicity
- [x] Only implement WebAssembly 1.0 (and none of the proposals)
- [x] Interpreter only
- [x] Support only WebAssembly binary encoding as an input (no support for the text format (`.wat`/`.wast`))

III) Conformance
- [x] Should have 99+% unit test coverage
- [x] Should pass the official WebAssembly test suite
- [x] Should have stricter tests than the official test suite

IV) First class support for determistic applications (*blockchain*)
- [ ] Support canonical handling of floating point (i.e. NaNs stricter than in the spec)
- [ ] Support an efficient big integer API (256-bit and perhaps 384-bit)
- [ ] Support optional runtime metering in the interpreter
- [ ] Support enforcing a call depth bound
- [ ] Further restrictions of complexity (e.g. number of locals, number of function parameters, number of labels, etc.)

## Building and using

Fizzy uses CMake as a build system:
```sh
$ mkdir build && cd build
$ cmake ..
$ cmake --build .
```

This will build Fizzy as a library. [C API] is provided for embedding Fizzy engine in applications.

A small example of using the C API, which loads a wasm binary and executes a function named `main`:

```c
#include <fizzy/fizzy.h>

bool execute_main(const uint8_t* wasm_binary, size_t wasm_binary_size)
{
    // Parse and validate binary module.
    const FizzyModule* module = fizzy_parse(wasm_binary, wasm_binary_size);

    // Find main function.
    uint32_t main_fn_index;
    if (!fizzy_find_exported_function_index(module, "main", &main_fn_index))
        return false;

    // Instantiate module without imports.
    FizzyInstance* instance = fizzy_instantiate(module, NULL, 0, NULL, NULL, NULL, 0);

    // Execute main function without arguments.
    const FizzyExecutionResult result = fizzy_execute(instance, main_fn_index, NULL, 0);
    if (result.trapped)
        return false;

    fizzy_free_instance(instance);
    return true;
}
```

Fizzy also provides a CMake package for easy integration in projects that use it:
```cmake
find_package(fizzy CONFIG REQUIRED)
...
target_link_libraries(app_name PRIVATE fizzy::fizzy)
```

Fizzy also has a Rust binding. It is published on [crates.io](https://crates.io/crates/fizzy) and the
official documentation with examples can be read on [docs.rs](https://docs.rs/fizzy/).

## WASI

Building with the `FIZZY_WASI` option will output a `fizzy-wasi` binary implementing
the [WASI] API (a very limited subset of [wasi_snapshot_preview1], to be precise).
It uses [uvwasi] under the hood. It can be used to execute WASI-compatible binaries on the command line.

```sh
$ mkdir build && cd build
$ cmake -DFIZZY_WASI=ON ..
$ cmake --build .
```

Try it with a Hello World example:
```sh
$ bin/fizzy-wasi ../test/smoketests/wasi/helloworld.wasm
hello world
```

## Testing tools

Building with the `FIZZY_TESTING` option will output a few useful utilities:

```sh
$ mkdir build && cd build
$ cmake -DFIZZY_TESTING=ON ..
$ cmake --build .
```

These utilities are as follows:

### fizzy-bench

This can be used to run benchmarks available in the `test/benchmarks` directory.
Read [this guide](./test/benchmarks/README.md) for a short introduction.

### fizzy-spectests

Fizzy is capable of executing the official WebAssembly tests.
Follow [this guide](./test/spectests/README.md) for using the tool.

### fizzy-testfloat

A tool for testing implementations of floating-point instructions.
Follow [this guide](./test/testfloat/README.md) for using the tool.

### fizzy-unittests

This is the unit tests suite of Fizzy.

## Special notes about floating point

### Exceptions

Certain floating point operations can emit *exceptions* as defined by the IEEE 754 standard.
(Here is a [summary from the GNU C Library](https://www.gnu.org/software/libc/manual/html_node/FP-Exceptions.html)).
It is however up to the language how these manifest, and in C/C++ depending on the settings they can result in
a) setting of a flag; b) or in *traps*, such as the `SIGFPE` signal.

Fizzy does not manipulate this setting, but expects it to be left at option a) of above, which is the default.
In the GNU C Library this can be controlled via the [`feenableexcept` and `fedisableexcept` functions](https://www.gnu.org/savannah-checkouts/gnu/libc/manual/html_node/Control-Functions.html).

The user of the Fizzy library has to ensure this is not set to *trap* mode.
The behavior with traps enabled is unpredictable and correct behaviour is not guaranteed, thus we strongly advise against using them.

### Rounding

The IEEE 754 standard defines four rounding directions (or modes):
- to nearest, tie to even (default),
- toward -∞ (rounding down),
- toward +∞ (rounding up),
- toward  0 (truncation).

The WebAssembly specification expects the default, "to nearest", rounding mode for instructions
whose result can be influenced by rounding.

Fizzy does not manipulate this setting, and expects the same rounding mode as WebAssembly,
otherwise the result of some instructions may be different.
In the GNU C Library the rounding mode can be controlled via the [`fesetround` and `fegetround` functions](https://www.gnu.org/software/libc/manual/html_node/Rounding.html).

If strict compliance is sought with WebAssembly, then the user of Fizzy must ensure to keep the default rounding mode.

### x87 FPU

On Intel i386 architecture (32-bit) the [x87](https://en.wikipedia.org/wiki/X87) FPU is used by default to perform floating-point operations.
The FPU is claimed to be IEEE-754 compliant, but there is one gotcha. The operations are executed with so-called _internal precision_ and the results are rounded to the target precision in the end [[1]].
By default the precision is set to [80-bit extended precision](https://en.wikipedia.org/wiki/Extended_precision) (except VC++ runtime [[2]]).
Unfortunately, this causes problems for 64-bit double precision operations (`f64.add`, `f64.sub`, `f64.mul`, `f64.div`) — the results may be different than when computed with double precision directly.

The precision of the FPU can be dynamically modified [[1]] but this has similar issues to controlling the rounding mode and there exist no C/C++ standard way of doing so.

In the Fizzy implementation we decided it is not worth to fight with the x87 FPU quirks.
Floating-point operations were never our top priorities.
We decided to opt-in for using SSE2 instructions to implement WebAssembly floating-pointing instructions
also in x86 32-bit builds (this is default for 64-builds). I.e. SSE2 instructions set is required.
This option is controlled by [`-msse2 -mfpmath=sse`][x86-Options] compiler options and one can always override
this decision if wants to experiment with x87 FPU.

For the record, we also tried to use [`-mpc64`][x86-Options] GCC compiler option which is suppose to set the FPU to 64-bit double precision.
This attempt was unsuccessful for unknown reason.

[1]: http://christian-seiler.de/projekte/fpmath/
[2]: https://randomascii.wordpress.com/2012/03/21/intermediate-floating-point-precision/
[x86-Options]: https://gcc.gnu.org/onlinedocs/gcc/x86-Options.html

## Development

### Performance testing

We use the following build configuration for performance testing:

- Release build type (`-DCMAKE_BUILD_TYPE=Release`)

  This enables `-O3` optimization level and disables asserts.

- Link-Time Optimization (`-DCMAKE_INTERPROCEDURAL_OPTIMIZATION=TRUE`)

  This does not give big performance gains (v0.3: +1%, v0.4: -3%), but most importantly
  eliminates performance instabilities related to layout changes in the code section of the binaries.
  See [#510](https://github.com/wasmx/fizzy/pull/510) for example.

- Latest version of GCC or Clang compiler

  The Fizzy built with recent versions of GCC and Clang has comparable performance.
  For LTO builds of Fizzy 0.4.0 Clang 10/11 is 4% faster than GCC 10.

- libstdc++ standard library

- No "native" architecture selection

  The compiler option `-march=native` is not used, leaving default architecture to be selected.
  Building for native CPU architecture can be easily enabled with CMake option `-DNATIVE=TRUE`.
  We leave the investigation of the impact of this for the future.

## Releases

For a list of releases and changelog see the [CHANGELOG file](./CHANGELOG.md).

## Maintainers

- Alex Beregszaszi [@axic]
- Andrei Maiboroda [@gumb0]
- Paweł Bylica [@chfast]

## License

[![license badge]][Apache License, Version 2.0]

Licensed under the [Apache License, Version 2.0].

[webassembly]: https://webassembly.org
[standard readme]: https://github.com/RichardLitt/standard-readme
[circleci]: https://circleci.com/gh/wasmx/fizzy/tree/master
[codecov]: https://codecov.io/gh/wasmx/fizzy
[Apache License, Version 2.0]: LICENSE
[@axic]: https://github.com/axic
[@chfast]: https://github.com/chfast
[@gumb0]: https://github.com/gumb0

[C API]: ./include/fizzy/fizzy.h

[WASI]: https://github.com/WebAssembly/WASI
[uvwasi]: https://github.com/nodejs/uvwasi
[wasi_snapshot_preview1]: https://github.com/WebAssembly/WASI/blob/master/phases/snapshot/witx/wasi_snapshot_preview1.witx

[webassembly badge]: https://img.shields.io/badge/WebAssembly-Engine-informational.svg?logo=webassembly
[readme style standard badge]: https://img.shields.io/badge/readme%20style-standard-brightgreen.svg
[circleci badge]: https://img.shields.io/circleci/project/github/wasmx/fizzy/master.svg?logo=circleci
[codecov badge]: https://img.shields.io/codecov/c/github/wasmx/fizzy.svg?logo=codecov
[license badge]: https://img.shields.io/github/license/wasmx/fizzy.svg?logo=apache
