## Disclaimer (READ THIS)

**LiteCore is not a supported / delivered product**, rather a component of Couchbase Lite. We don't recommend you use it directly. [Couchbase Lite for C](https://github.com/couchbaselabs/couchbase-lite-C) also has a C API but is easier to use and better supported.

That being said, you may need to build LiteCore if you are contributing fixes or improvements, or as part of building your own Couchbase Lite. So here's how.

For arbitrary reasons, we have chosen x86 Linux as an example but the concepts will apply to any platform.  

## Dependencies

* A C++ compiler and standard library. [Clang](https://clang.llvm.org/) and [libc++](https://libcxx.llvm.org/) are preferred because that's what we use for building Couchbase Lite, but GCC 7 or later will work too.
* The [CMake](https://cmake.org) build tool.
* [ICU](http://site.icu-project.org) (International Components for Unicode) libraries -- usually called `libicu-dev` in package managers like apt.
* zlib

## Building

The typical CMake build process is:

1. (First time) Create a directory for the build products and intermediate files (can be anywhere)
2. `cd` to that directory
3. Run `cmake` giving the path to the root project directory -- this creates or updates the makefiles
4. Run `make` (usually with arguments `-j` and the number of parallel threads to run)

The end product of this will be `libLiteCore.so` (or on macOS, `libLiteCore.dylib`) in the same folder.

There are some simple shell scripts that build LiteCore, in the `build_cmake/scripts/` directory. Just run the appropriate one for your platform. The build output will appear in `build_cmake/mac/` or `build_cmake/unix/`.

To use Clang when the default compiler is GCC, put `CC=clang CXX=clang++` at the start of the `cmake` command.

To create a debug build, add the `cmake` argument -DCMAKE_BUILD_TYPE=Debug`.

For example:

    $ mkdir -p /tmp/build
    $ cd /tmp/build
    $ CC=clang CXX=clang++ cmake -DCMAKE_BUILD_TYPE=Debug /path/to/couchbase-lite-core
    $ make -j 8

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