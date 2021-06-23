After some refactoring, LiteCore has added a C++ API, found in `C/Cpp_include/`. The existing C API is still there and works the same, but it's now implemented as a thin wrapper over the C++ one.

## About The API

The opaque types in the C API, like `C4Database` and `C4Query`, are now bona fide C++ classes (well, structs) whose definitions are in the corresponding C++ headers. For example, while `c4Base.h` declares `typedef struct C4Database C4Database;`, in `c4Database.hh` you'll find the actual `struct C4Database { ... };`, containing all the database methods.

This means that the APIs are pretty interchangeable. The types are the same, and the C functions just call into the C++ methods. But the C++ code can be a lot simpler! For example:

#### C code:
```c
C4Error error;
C4Database *db = c4db_openNamed(C4Str(name), &config, &error);
if (!db) {
    return error;
}
uint64_t count = c4db_getDocumentCount(db);
c4db_release(db);
```

#### Equivalent C++ code:

```c++
Retained<C4Database> db = C4Database::openNamed(name, config);
uint64_t count = db->getDocumentCount();
```

#### Even shorter equivalent C++ Code:

```c++
uint64_t count = C4Database::openNamed(name, config)->getDocumentCount();
```

If you'd like to see some real-world code that uses the C++ API, look at the [Couchbase Lite For C implementation][CBL_C_SAMPLE.

## High-Level API Differences

* C++ has higher-level semantics like constructors, reference parameters, STL collections...
  - 👍 Makes your code simpler and clearer.
  - 👎 C++ ABI doesn't play well with shared libraries/DLLs; requires linking with static lib. Not callable from .NET.
* The C++ methods return ref-counted objects as smart pointers (`Retained<T>`), not raw pointers.
  - 👍 You don't have to write "release" calls. 
  - 👍 Less chance of leaks.
* The C++ methods throw exceptions (of type `litecore::error`), while the C functions catch those exceptions and return them as `C4Error`.
  - 👍 Less error handling code for you to write. 
  - 👍 You can't forget to check an error result.
  - 👎 You need to add your own C++ `catch` blocks in top-level code that returns to the platform.
* The C++ methods use the C++ slice classes, `fleece::slice` and `fleece::alloc_slice`, instead of the C structs `C4Slice` and `C4SliceResult`.
  - 👍 You don't have to remember to release `C4SliceResults`. 
  - 👍 Slice classes have lots of utility methods and conversions that simplify your code.


## Adopting The C++ API

### Preparations

1. Add the following directories to the compiler's C++ header search path:
   * `couchbase-lite-core/C/Cpp_include/`
   * `couchbase-lite-core/vendor/fleece/Fleece/Support/`
2. In your source and header files, change all your LiteCore "`#include`"s to `.hh` instead of `.h`, i.e. `c4Database.hh`
3. Switch to linking with a LiteCore static library, not a dynamic library

### Converting Your Code

Some of these topics, like smart pointers and exceptions, are covered in more detail in later sections.

* **Adopt smart pointers.** When you assign a ref-counted object like a `C4Database*` to a variable, change that variable's type from "`T`" to "`Retained<T>`". (You _don't_ need to change function parameters. Passing a `T*` is fine.) 
* **Change LiteCore function calls to method calls.** Look through the corresponding `.hh` header to find the equivalent method, and then change your function call into a method call. The general pattern is to change `c4widget_spin(widget)` to `widget->spin()`.
* **Remove "`&error`" parameters.** If a function took a final `C4Error*` parameter, remove the corresponding argument. You can probably remove your local `C4Error` variable too.
* **Replace error handling code with `try...catch` blocks.** In almost all cases you can rip out the code following a call that checks for an error return and deals with it. But in its place, you do have to ensure you have a C++ `try...catch` block somewhere up the call chain, probably in your top-level code that implements a public platform API. Those `catch` blocks can usually contain boilerplate that translates the caught exception into a platform exception or error.
* **Adopt `alloc_slice`.** The C++ API uses this instead of `C4SliceResult`. It's very much like `Retained<>` for slices: it automatically handles the ref-counting for you.
* **Simplify use of slices.** The C++ `slice` class and its subclass `alloc_slice` have a pretty rich API that can simplify your code: conversions to and from other types, searching, subranges, comparisons... Take a look at `fleece/slice.hh`.

## A Note On C4Document

`C4Document` is sort of troublesome because the C API declares it as a real struct with public fields, not just an opaque type. Making this work as a ref-counted C++ object was a bit messy. There are two ways you might notice this:

If a compilation unit ends up `#include`ing `c4Document.h` before any of the C++ `.hh` headers, `struct C4Document` will get declared C-style instead of C++-style, and things will go wrong. This will probably manifest as an error about an unknown type `C4Document_C`. If this happens, double-check that you've converted your `#include`s to `.hh`. If you can't convert all of them for compatibility reasons, look at your order of `#include`s.

In the C++ API I've chosen not to expose the public fields of `C4Document`. Instead, there are getter methods `docID()`, `revID()`, `flags()`, `sequence()` and `selectedRev()`.


## Storing References To Ref-Counted Objects

Many of the C++ classes are ref-counted (inherit from `fleece::RefCounted`.) These should be stored in `Retained<T>` values, not as raw pointers. (It's OK to pass a raw pointer _to_ a function/method, though.) For example:

```c++
Retained<C4Document> doc = db->getDocument("foo");
```

If you're using the C++ API, it's best to avoid the C functions that create objects; call the C++ equivalents instead. Otherwise it's easy to make a mistake like:

```c++
Retained<C4Document> doc = c4db_getDoc(db, "foo"_sl, true, kDocGetCurrentRev, &error);  // ☠️
```
which will leak the document object. The `Retained` smart-pointer will retain the document when it's assigned, and release it when it's done with it ... but nothing will release the reference created by `c4db_getDoc` itself. Don't do this, use the above form instead.


## Error/Exception Handling

As mentioned above, the C++ API throws exceptions. You'll notice an absence of "`C4Error *outError`" parameters. 

>Important: Any LiteCore method (including a constructor!) can throw an exception, except for those marked as `noexcept`, and destructors. Make sure you have your own `try...catch` block in place farther up the call stack.

The C++ API still uses `C4Error` to describe errors, however. (But it's got methods now.)

The exact exception types thrown by LiteCore are internal classes (`litecore::error`, `fleece::FleeceException`, and sometimes `SQLite::Exception`). In your code it's best to just catch all exceptions, then call `C4Error::fromCurrentException()` (declared in c4Error.h) inside your `catch` block to get the current exception as a `C4Error`. For example:

```c++
try {
    Retained<C4Document> doc = db->getDocument("foo");
    do_something_with(doc);
} catch (...) {
    C4Error error = C4Error::fromCurrentException(x);
    somehow_report_error(error);
}
```

In reporting the error, you may find the C4Error methods `message()`, `description()` and `backtrace()` useful; all return `std::string`.

Conversely, you might sometimes find it useful to _throw_ a LiteCore error yourself, so your `catch` block can deal with it. `C4Error::raise()` lets you do this.

## TBD

The method naming is not as compatible with the C API as it should be. I'll clean this up in the near future.

The following APIs are not yet available as C++:
* C4PredictiveModel
* C4Socket



CBL_C_SAMPLE: https://github.com/couchbaselabs/couchbase-lite-C/blob/master/src/CBLDatabase_Internal.hh