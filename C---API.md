>**NOTE:** This change is currently still only on the `dev` branch. (9 March 2021)

After some refactoring, LiteCore has added a C++ API. The existing C API is still there, but it's now implemented as a thin wrapper over the C++ one.

The opaque types declared c4Base.h, like `C4Database` and `C4Document`, are now bona fide C++ classes (well, structs) whose definitions are in the corresponding headers in `C/Cpp_include/`. So while c4Base.h declares `typedef struct C4Database C4Database`, in `c4Database.hh` you'll find the actual `struct C4Database { ... }`, containing all the database methods.

This means that the APIs are pretty interchangeable. The C functions just call into the C++ methods. The significant differences are:

* The C++ methods **throw exceptions** (of type `litecore::error`), while the C functions catch those exceptions and return them as `C4Error`.
* The C++ methods return ref-counted objects as `Retained<T>`, not raw pointers.
* The C++ methods use the C++ slice classes, `fleece::slice` and `fleece::alloc_slice`.

## Using The API

It's pretty easy. You just need to 

1. Add these directories to the compiler's header search path:
   * `couchbase-lite-core/C/Cpp_include/`
   * `couchbase-lite-core/vendor/fleece/Fleece/Support/`
2. Include LiteCOre headers as `.hh` files instead of `.h`, i.e. `c4Database.hh`

## Storing References To Ref-Counted Objects

Some of the C++ classes are ref-counted (inherit from `fleece::RefCounted`.) These should be stored in `Retained<T>` values, not as raw pointers. (It's OK to pass a raw pointer _to_ a function/method, though.) For example:

```c++
Retained<C4Document> doc = db->getDocument("foo");
```

If you're using the C++ API, it's best to avoid the C functions that create objects; call the C++ equivalents instead. Otherwise it's easy to make a mistake like:

```c++
Retained<C4Document> doc = c4db_getDoc(db, "foo"_sl, true, kDocGetCurrentRev, &error);  // BAD!
```
which will leak the document object. `Retained` will retain the document when it's assigned, and release it when it's done with it ... but nothing will release the reference created by `c4db_getDoc` itself. Don't do this, use the above form instead.