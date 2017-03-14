Disclaimer:  LiteCore is not a directly supported / delivered product, but rather a product that is consumed by Couchbase Lite.  That being said, it is tested extensively as part of the Couchbase Lite build and testing process.  This document describes the basics of using the library.  For arbitrary reasons, I have chosen x86 Linux as an example but the concepts will apply to any platform.  

## Dependencies

Here is a list of all dependencies for LiteCore:

Compilation or Download needed
[libsqlite3](https://www.sqlite.org/download.html)
[libc++](https://libcxx.llvm.org/)
[libbsd](https://libbsd.freedesktop.org/releases/)
[libcrypto](https://github.com/openssl/openssl)

Likely already installed:
libm.so
libgcc_s.so
libc.so
libpthread.so
libdl.so
librt.so
ld-linux.so

## Building

LiteCore makes use of CMake, which is a project generation tool that supports many backends (Makefiles, Ninja, Visual Studio projects, etc).  The supported environment for compilation is using the [clang](https://clang.llvm.org/) compiler in conjunction with the [libc++](https://libcxx.llvm.org/) standard library (both from the LLVM project).  Linux tends to ship with GCC, and CMake tends to try to use it by default, but it will respect the environment variables `CC` and `CXX` to specify alternative compilers for C and C++ respectively.  So to set up the project, ensure that the above deps, clang, and CMake are installed and in the path on the build system and run this command:

`CC=clang CXX=clang++ cmake -DCMAKE_BUILD_TYPE=RelWithDebInfo <path/to/CMakeLists.txt>`

This will generate a `Makefile` that you can then build by simply running `make -j8 LiteCore` (or just `make LiteCore` but the former is faster with the appropriate value after `j`).

The end product of this will be `libLiteCore.so` in the same folder.

## Deployment

Deployment is the same as any other shared library.  Just ensure that the program using it can find and link to it (i.e. it is in the same folder as the executable, or in the system path) as well as its dependencies.

## Testing

There are two test suites in the repo:  C4Tests and CppTests.  The former tests the C API / dynamic library interface and the latter tests the internals by linking statically.  You can build them by completing the CMake setup and then running `make C4Tests` and/or `make CppTests`.  The executables will be located in LiteCore/tests/CppTests and C/tests/C4Tests.  You can run either directly.  The recommended way is to run `C4Tests -r list` and `CppTests -r list` with the current working directory set to the location of the test executable.

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