Disclaimer:  LiteCore is not a directly supported / delivered product, but rather a product that is consumed by Couchbase Lite.  That being said, it is tested extensively as part of the Couchbase Lite build and testing process.  This document describes the basics of using the library.

For arbitrary reasons, I have chosen x86 Linux as an example but the concepts will apply to any platform.  

## Dependencies

Here is a list of all dependencies for LiteCore on Linux:

Compilation or Download needed<br>
[libsqlite3](https://www.sqlite.org/download.html)<br>
[libc++](https://libcxx.llvm.org/)<br>
[libbsd](https://libbsd.freedesktop.org/releases/)<br>
[libcrypto](https://github.com/openssl/openssl)<br>

Likely already installed:
libm.so<br>
libgcc_s.so<br>
libc.so<br>
libpthread.so<br>
libdl.so<br>
librt.so<br>
ld-linux.so

## Building

LiteCore makes use of CMake, which is a project generation tool that supports many backends (Makefiles, Ninja, Visual Studio projects, etc).  The supported environment for compilation is using the [clang](https://clang.llvm.org/) compiler in conjunction with the [libc++](https://libcxx.llvm.org/) standard library (both from the LLVM project).  Linux tends to ship with GCC, and CMake tends to try to use it by default, but it will respect the environment variables `CC` and `CXX` to specify alternative compilers for C and C++ respectively. 

Before building, ensure that the above deps, clang, and CMake are installed.

There are some simple shell scripts that build LiteCore, in the `build_cmake/scripts/` directory. Just run the appropriate one for your platform. The build output will appear in `build_cmake/mac/` or `build_cmake/unix/`.

Or if you want to run cmake yourself, enter:

`CC=clang CXX=clang++ cmake -DCMAKE_BUILD_TYPE=RelWithDebInfo <path/to/CMakeLists.txt>`

which will generate a `Makefile` that you can then build by simply running `make -j8 LiteCore` (or just `make LiteCore` but the former is faster with the appropriate value after `j`).

The end product of this will be `libLiteCore.so` (or on macOS, `libLiteCore.dylib`) in the same folder.

## Deployment

Deployment is the same as any other shared library.  Just ensure that the program using it can find and link to it (i.e. it is in the same folder as the executable, or in the system path) as well as its dependencies.

## Testing

There are two test suites in the repo:  C4Tests and CppTests.  The former tests the C API / dynamic library interface and the latter tests the internals by linking statically. To fully test LiteCore, run them both.

You can build them by completing the CMake setup and then running `make C4Tests` and/or `make CppTests`.  The executables will be located in LiteCore/tests/CppTests and C/tests/C4Tests.  You can run either directly.  The recommended way is to run `C4Tests -r list` and `CppTests -r list` with the current working directory set to the root of the repository.

The tests will log a lot of output by default; to turn that off, set the environment variable `LiteCoreTestsQuiet`.

To move these tests to another system, the following directory structure should be used:

> |<br>
| - C<br>
| - | - tests<br>
| - | - | - data<br>
| - | - | - | - geoblocks.json<br>
| - | - | - | - iTunesMusicLibrary.json<br>
| - | - | - | - names_100.json<br>
| - | - | - | - names_300000.json<br>
| - | - | - | - nested.json<br>
| - C4Tests<br>
| - CppTests<br>
| - libLiteCore.so

The mentioned JSON files are checked into the repo, with the exception of geoblocks.json and names_300000.json which can be obtained from [this repo](https://github.com/arangodb/example-datasets/)