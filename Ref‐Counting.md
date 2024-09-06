LiteCore and Fleece heavily use reference counting for memory management.

# Basics

A ref-counted object has an internal counter in it, which reflects how many valid references there are to that object. When the object is first created and initialized it has one reference, so the counter is initialized to 1.

You never explicitly delete a ref-counted object; instead when you're done with it you **release** it, which decrements the counter. When the counter reaches zero, we know nothing is using the object anymore, so the object is automatically freed. Since release may free the object, _never_ use that pointer again afterwards. It's dead to you.

If you need to store another reference to the same object somewhere, you **retain** the object, which increments the counter. That second reference also requires a release call when it's no longer used.

> Note: As a convenience, it's always safe to retain or release a NULL pointer; it's a no-op.

> Note: Retain and release are atomic operations, so they are thread-safe. It's OK to release a reference on a different thread than the one where you first obtained or retained it.


# What Types Are Ref-Counted?

We have several categories of ref-counted objects:

* In the C API:
  - Types like `C4Database*` are opaque references that have associated functions like `c4db_retain()` and `c4db_release()`.
  - Some Fleece types like `FLDoc`, `FLMutableDict` `FLMutableArray`, are ref-counted and have their associated retain/release functions.
  - `FLSliceResult` is a ref-counted block of memory, often used as a string, with associated functions `FLSliceResult_Retain()` and `FLSliceResult_Release()`.
* In the C++ API:
  - Any subclass of `fleece::RefCounted`, which includes the LiteCore classes `C4Database` etc. These are always stored in a `Retained<>`.
  - Fleece classes like `Doc`, `MutableArray`, `MutableDict` -- these are all smart-pointer wrappers around their C equivalents.
  - `alloc_slice`, which is a smart pointer wrapping a `FLSliceResult`.
  - In the innards of LiteCore you'll find a few uses of `std::shared_ptr`, the standard C++ equivalent of `Retained<>`.


# Language-Specific Details

## In C

In C APIs, retain and release are functions that take a pointer. There are separate functions for each type of object; in fact, the existence of functions with "retain" and "release" in their names is a clue that the type they're passed is ref-counted.

The most common way you obtain a reference is by an API call that returns one, like `c4db_openNamed()`, whose doc-comment points out that you're responsible for releasing the result. Since the call is _passing ownership_ of the reference to you, you're responsible for calling release when done:

```C
// Temporarily using a ref-counted result:
C4Foo *foo = c4foo_new();
if (foo)
    do_something_with(foo);
c4foo_release(foo);
```

### Updating a stored reference

A more complicated case is where you have an existing reference variable and want to make it point to a different object. In this case you need to:

1. Retain the new value
2. Release the old value
3. Store the new value in the variable

The order is important: if you swap steps 1 and 2, then if the new value happens to be the same as the old value, the initial release could delete the object, which would be Very Bad.

```C
// Updating a stored ref-counted reference:
typedef struct { C4Foo* foo; } mystruct;

void updateFoo(mystruct *s, C4Foo* newFoo) {
  c4foo_retain(newFoo);
  c4foo_release(s->foo);
  s->foo = newFoo;
}
```

This works even if either of `newFoo` and/or `s->foo` are NULL, or if they point to the same object.

> Note: This three-step tango is _not_ thread-safe (and making it thread-safe would be more difficult than it appears.) Never mutate a reference variable when another thread might have access to it!


## In C++

In C++ APIs, we don't call retain/release directly. Instead we use "smart pointer" types that do it automatically: these are `Retained<>` and `RetainedConst<>`, as well as `alloc_slice`, `MutableDict`, `MutableArray`. These all do the 'three-step tango' described above when you assign to them, and they release their value in their destructor.

```c++
// Temporarily using a ref-counted result:
Retained<C4Foo> foo = C4Foo::newFoo();
if (foo)
  do_something_with(foo);
// No need to release; `foo` will release when it goes out of scope.
```

```c++
// Updating a stored ref-counted reference:
class mystruct { Retained<C4Foo> _foo; };

void mystruct::updateFoo(C4Foo* newFoo) {
    _foo = newFoo;
}
```

> Note: Another benefit of `Retained<>` (and the other smart pointers) is that they're automatically initialized to NULL, not garbage like a raw pointer.

### Passing parameters

A question that sometimes comes up is whether you should pass a ref-counted type to a function as a plain pointer or a smart pointer (`Foo*` or `Retained<Foo>`.) Opinions vary.

- Passing a smart pointer is guaranteed to be safe, but it's expensive (at the micro-level.) The implicit retain/release calls involve atomic operations that take hundreds of CPU cycles. [Citation Needed]
- Passing a raw pointer is very fast ... but there is an edge case where it can bite you: if the function passed the pointer makes a call that releases the caller's reference. 

We generally use the fast option: pass a raw pointer.

### Using `new`

Inside LiteCore you may find yourself directly creating ref-counted objects using `new`. A new RefCounted object **must** immediately be assigned to a `Retained<>` value, otherwise it has a dangling ref-count of zero.

```c++
Foo* foo = new Foo(42);             // Bad!
auto foo = new Foo(42);             // Bad! (foo is type Foo*, just like above)
Retained<Foo> foo = new Foo(42);    // Good
auto foo = make_retained<foo>(42);  // Best (make_retained<T> returns a Retained<T>.)
```
