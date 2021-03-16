>**NOTE:** This change is currently still only on the `dev` branch. (9 March 2021)

After some refactoring, LiteCore has added a C++ API. The existing C API is still there, but it's now implemented as a thin wrapper over the C++ one.

The opaque types declared c4Base.h, like `C4Database` and `C4Document`, are now bona fide C++ classes (well, structs) whose definitions are in the corresponding headers in `C/Cpp_include/`. So while c4Base.h declares `typedef struct C4Database C4Database`, in `c4Database.hh` you'll find the actual `struct C4Database { ... }`, containing all the database methods.

This means that the APIs are pretty interchangeable. The C functions just call into the C++ methods. The significant differences are:

* The C++ methods return ref-counted objects as `Retained<T>`, not raw pointers.
* The C++ methods throw exceptions (of type `litecore::error`), while the C functions catch those exceptions and return them as `C4Error`.
* The C++ methods use the C++ slice classes, `fleece::slice` and `fleece::alloc_slice`.

## Using The API

It's pretty easy. You just need to 

1. Add these directories to the compiler's header search path:
   * `couchbase-lite-core/C/Cpp_include/`
   * `couchbase-lite-core/vendor/fleece/Fleece/Support/`
2. Include LiteCore headers as `.hh` files instead of `.h`, i.e. `c4Database.hh`

### C4Document

`C4Document` is sort of troublesome because the C API declares it as a real struct with public fields, not just an opaque type. Making it work as a ref-counted C++ object was a bit messy. This shouldn't matter to clients, _except_ that if a compilation unit ends up `#include`ing c4Document.h before any of the C++ `.hh` headers, `struct C4Document` will get declared C-style instead of C++-style, and things will go wrong. At the moment this will probably manifest as an error about an unknown type `C4Document_C`. If this happens, look at your order of `#include`s.

In the C++ API I've chosen not to expose the public fields. Instead, there are getter methods `docID()`, `revID()`, `flags()`, `sequence()` and `selectedRev()`.

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

## Error Handling

As mentioned above, the C++ API throws exceptions. You'll notice an absence of "`C4Error *outError`" parameters. Some methods have a `noexcept` keyword; those are guaranteed not to throw anything, but all others can.

The exact exception types thrown are internal classes (`litecore::error` and `fleece::FleeceException`). In your code it's best to just catch any `std::exception`. You can call `C4ErrorFromException()` (declared in c4Base.hh) inside a `catch` block to get the current exception as a `C4Error`. For example:

```c++
try {
    Retained<C4Document> doc = db->getDocument("foo");
    do_something_with(doc);
} catch (const std::exception &x) {
    C4Error error = C4ErrorFromException(x);
    somehow_report_error(error);
}
```

In reporting the error, you may find the C4Error methods `message()`, `description()` and `backtrace()` useful; all return `std::string`.

Conversely, you might sometimes need to _throw_ a LiteCore error yourself. The `C4ThrowError()` function lets you do this.

## TBD

The method naming is not as compatible with the C API as it should be. I'll clean this up in the near future.

The following APIs are not yet available as C++:
* C4Cert
* C4KeyPair
* C4Listener
* C4PredictiveModel
* C4Socket