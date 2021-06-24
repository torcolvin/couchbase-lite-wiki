# Intro To LiteCore

**Couchbase Lite Core**, or **LiteCore**, is the shared cross-platform implementation of Couchbase Lite’s functionality. LiteCore itself is written in C++ and exposes C and [[C++|CPlusPlus API]] APIs. Each Couchbase Lite implementation wraps around LiteCore’s API to provide an idiomatic API in its platform’s language.

> Important: LiteCore itself is not Couchbase Lite! It is not a product, we do not support 3rd parties using it directly, and its “public” API is public only to Couchbase Lite. We do have an [official Couchbase Lite implementation with a C/C++ API](https://github.com/couchbaselabs/couchbase-lite-C) for developers to use.

LiteCore’s API is _similar_ to Couchbase Lite’s, but not identical. Some features are exposed at a lower level, and some features in LiteCore don’t exist in the public API for one reason or another.

# Concepts

I’m assuming you understand Couchbase Lite and its API (on any platform.) LiteCore is pretty similar, at a high level. It’s an object-oriented API, even when expressed in C.

## General

LiteCore’s C API can be found in `C/include/`, and its C++ API in `C/Cpp_include/`. The header files are your friends! Not only are they the source of truth for the API details, they also have documentation comments. There are also HTML docs generated from the headers, but they might not be up to date.

LiteCore uses the prefix “`C4`” or “`c4_`” for most of its identifiers. This doesn’t really stand for anything anymore, but we could say it means “Couchbase Client Core Classes”.



## Classes and Objects

### In C

Most LiteCore classes are represented in C as opaque structs that can only be referred to through pointers. For example:

```cpp
typedef struct C4Database C4Database;
```

This declares that there is a struct named `C4Database`, without actually defining what it contains. The type `C4Database*` is used to refer to an instance. This is a common technique in C APIs.

“Methods” are functions that take a class instance pointer as the first parameter. Their names are prefixed with the name of the class (or an abbreviation.) For example:

```cpp
uint64_t c4db_getDocumentCount(C4Database*);
```

Most classes are reference-counted. These classes have two special methods, `retain` and `release`, that increment and decrement the reference count. When the count reaches zero the object is freed.

Some functions that return an object pointer create a new reference. This includes any function that creates a new object. When you call such a method, you are responsible for releasing the object when you’re done with it; otherwise it’ll be leaked. (The API documentation will call this out.)

Classes that aren’t reference-counted have a `free` method that frees the instance.

### In C++

The C++ headers actually contain the full definitions of the structs. For example, in `c4Database.hh` you’ll find (slightly simplified):

```cpp
struct C4Database : public fleece::RefCounted {
    ...
    uint64_t getDocumentCount() const;
    ...
};
```

> Note: Yes, this is a C++ class, even though it says `struct`. For historical reasons, the C++ keywords `struct` and `class` have almost identical meaning; the only difference is that struct members are public by default and class members are private by default.

The C++ API uses smart pointers to manage ref-counting automatically. The template `fleece::Retained<T>` represents a pointer to class `T`, with an implicit reference. Assigning an object pointer to a `Retained` retains it, and when it stops being stored the `Retained` will release it. API methods that return retained references to objects will return a `Retained` to transfer the reference.

The rule of thumb is that any time you have a variable (local or in a struct/class) that points to a ref-counted object, that variable should be a `Retained`. This doesn’t apply to function parameters: when you pass an object to a function/method you can just pass it as a regular pointer.


## Slices and Strings

LiteCore makes heavy use of “slices”, a simple data structure defined in the Fleece library (a submodule.) The C type is `FLSlice`, also known by the typedefs `FLString`, `C4Slice` and `C4String`. The C++ type is `fleece::slice`.
```cpp
typedef struct FLSlice {
    const void* buf;
    size_t      size;
} FLSlice;
```

A slice is just a pointer to a range of bytes in memory. LiteCore uses it to represent both strings and binary data. [Slices have their own documentation](https://github.com/couchbaselabs/fleece/wiki/Slices), which you should definitely read.

A slice itself has no ownership; it’s just a pointer. The `FLSliceResult` type, `alloc_slice` in C++, _does_ own the memory it points to. It’s reference counted, so that the heap block can be freed. In C, any time a `FLSliceResult` is returned to you from a function, you will need to call `FLSliceResult_Release` when you’re done with it. In C++ the `alloc_slice` already acts like a smart pointer that takes care of ref-counting for you.



## Error Handling

### C4Error

The struct `C4Error` (defined in `c4Error.h`) contains a description of an error. It has two fields: `domain` is an enum that identifies what subsystem created the error, and `code` is an integer defined by that subsystem identifying the specific error. 

**The error code 0 always means “no error”**, no matter what the domain.

The error domains are:

- `LiteCoreDomain` — LiteCore-specific errors, like `kC4ErrorInvalidQuery`. (Defined in the `C4ErrorCode` enum in `"c4Error.h"`)
- `POSIXDomain` — POSIX error codes like `EADDRINUSE`. (Defined in `<errno.h>`)
- `SQLiteDomain` — SQLite error codes like `SQLITE_BUSY` (Defined in `<sqlite3.h>`)
- `FleeceDomain` — Fleece storage library errors like `kFLJSONError` (Defined in `"fleece/Fleece.h"`)
- `NetworkDomain` — LiteCore-defined networking errors, many involving TLS, like `kC4NetErrDNSFailure`. (Defined in the `C4NetworkErrorCode` enum)
- `WebSocketDomain` — WebSocket or HTTP status codes, like `404 Not Found`. Small numbers (under 600) are HTTP, large numbers (over 10000) are WebSocket. (Defined in various Internet RFCs)
- `MbedTLSDomain` — Low-level crypto or TLS errors. (Defined in `"mbedtls/error.h"`)

A `C4Error` also has an error message. This isn’t stored in the struct; you can get it by calling `c4error_getMessage` or `c4error_getDescription`.

### In C

Not all C API functions return `C4Error`s. Some just can’t fail, and some are simple enough that you don’t need an error code, so they just return `false` or `NULL`. The function’s documentation will indicate this.

The C API functions that do return `C4Error`s use a convention inspired by Apple’s Objective-C frameworks. They behave like this:
* The last parameter to the function is of type `C4Error*`, i.e. the caller passes a pointer to a `C4Error` struct (or `NULL` if they don’t care.)
* The function has a special return value, usually `false` or `NULL`, that indicates an error.

Here’s an example:
```cpp
bool c4db_purgeDoc(C4Database *database, C4String docID, C4Error* outError);
```

If the function fails, it:
1. stores the error information in the pointed-to `C4Error`, _unless_ the pointer is `NULL`;
 2. returns the special failure value.

When calling such a function, you should:
1. Declare a `C4Error` local variable, usually called `error` or `err`
2. Pass the address of this variable as the last function argument
3. Check if the function’s returned value is the special failure value
4. If so, handle the failure, looking at the contents of the `C4Error` for details.

> Warning: In step 3, **do not** check for failure by testing `error.code`! If the function succeeds, it doesn’t store anything there, so when the function returns it may still be uninitialized, i.e. garbage. Always check the function’s return value first.

Example:
```cpp
C4Error error;
if (!c4db_purgeDoc(db, docID, &error) {    // returns false on error
    my_error_handler(error);
    return;
}
```

> Note: If for some reason you don’t care about the specifics of the error (maybe you’re just cleaning up after something else already went wrong), you can pass `NULL` instead of a pointer to a `C4Error`.

It’s pretty common for errors to be propagated up through a call chain. The usual way to do this is for your own function to follow the same convention: it also takes a final parameter of type `C4Error*` (usually named `outError`.) Then when you call a LiteCore function you pass that same pointer to it. If the function’s return value indicates it failed, you clean up and then return your own failure value. The caller can look at the `C4Error` to figure out what happened.

Example:
```cpp
bool my_purge(slice docID, C4Error *outError) {
    if (!c4db_purgeDoc(my_db, docID, outError) {
        return false;
    }
    ...
    return true;
}
```

### In C++

The C++ API uses exceptions instead of returning errors through function parameters. This makes calls and error handling simpler, because exceptions automatically propagate up to the nearest `catch` block. Usually you’ll put `catch` blocks in the top level methods of your public API, and handle the exception by converting it to a platform exception/error.

> Note: You should assume that any LiteCore method other than a destructor can throw an exception, unless its declaration includes the keyword `noexcept`.

The first example in the previous section looks like this in C++:
```cpp
db->purgeDoc(docID);
```

In your `catch` block you call `C4Error:: fromCurrentException()` to get a `C4Error` value corresponding to the exception that was caught. For example, if we’re writing a C++ support method for a binding for the popular Freon language:

```cpp
freon_error MyDatabaseWrapper::purge(freon_string docID) {
    try {
        _db->purgeDoc(asSlice(docID));
		return freon_ok;
    } catch (...) {
        C4Error error = C4Error:: fromCurrentException();
        return bridge_to_freon_error(error);
    }
}
```

Except for these top-level bridges, you shouldn’t often need to use `try...catch`. If you find you need to run cleanup code, consider using “RAII” idioms — this is a C++ term for helper classes that own resources and clean them up in their destructors. An example is `std::unique_ptr`, which owns a memory allocation and deletes it automatically.

# What Next?

The [[Home]] page of the wiki has a lot of links to other documentation.