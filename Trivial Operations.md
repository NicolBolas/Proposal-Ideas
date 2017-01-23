% Trivial Operations
%
%

The C++ concepts of trivially copyable and trivial types are very useful for low-level code. However, employing them in a number of circumstances often require heavy use of casting, the use of more opaque functions like `memcpy`, or even relying on undefined behavior.

This proposal includes a number of standard library functions that improve upon such constructs, making them well-defined C++ without violating the C++ object model.

# Motivation

This proposal offers to solve a number of cases in a C++-compliant way.

## Trivial Construction In-Place

At present, placement-`new` can be employed to begin the lifetime of an object in a piece of memory:

````
auto mem = std::malloc(sizeof(Type));
auto ptr = new(mem) Type;
````

This performs default-initialization on `Type`. If `Type` is trivially default constructible, then the values within `ptr` will be undefined.

But sometimes, having such values be undefined is not what you want:

````
auto mem = std::malloc(sizeof(Type));
fill_in_memory(mem);
````

`fill_in_memory`, in this case, is a function which put something into that memory. This may have read data from a file-stream, it may be a packet from a TCP/IP socket, whatever. However, we as the user know exactly what `fill_in_memory` wrote. And using standard layout rules, we are able to construct `Type` (for this platform) to match the value representation of that data. So logically, `mem` now contains `Type`.

But as far as the C++ object model is concerned, `mem` does not contain `Type`. Therefore:

````
auto ptr = static_cast<Type*>(mem);
````

The use of `ptr` represents undefined behavior. This kind of casting is frequently done in low-level code anyway, because `Type` is trivially default constructible and so long as the value representation stored in `ptr` matches `Type`'s value representation, it ought to work. But that doesn't make it any less UB.

Note that `new(mem) Type` is not guaranteed to *preserve* the contents of `mem`. Debug builds often write arbitrary values, even during placement new operations, and this behavior is permitted by the standard.

What we need is a function which will begin the lifetime of `Type`, but preserve the contents of the memory it is being built within.

## Trivial Copy Construct

When reading memory from an arbitrary source whose value representation matches a C++ object, the typical idiom for accessing it is to do this:

````
T t;
memcpy(&t, src, sizeof(T));
````

While the aforementioned ability to construct an object within that memory would allow us to handle this in-situ, perhaps we need to reuse `src` or maybe it does not belong to us. Regardless of why, we want to construct an object from the contents of that memory.

The above idiom imposes a restriction on `T`; it requires not only that it is trivially-copyable but that it is Assignable. For example:

````
struct NoAssign
{
	const int i;
};
````

The aforementioned code will compile with `NoAssign`, but it will invoke UB due to changing the value of a `const` object.

What we really need is the equivalent of a copy constructor that constructs `NoAssign` as a copy of some memory.

## Trivial Copy Assignment

The use of `memcpy` on trivially copyable types is permitted by the standard. However, it is a bit cumbersome to write:

````
std::memcpy(&dstObj, &srcObj, sizeof(Typename));
````

Some repetition has to be done here, since `sizeof(Typename)` is deducible from the actual types of the memory.

But more importantly, there are types which are trivially copyable yet are not Assignable:

````
struct Type
{
	const int i;
};
````

To perform trivial copy assignment on `Type` violates the prohibition on changing a `const` object. As such, `Type` has `delete`d copy and move assignment operators. But `memcpy` will work just fine.

If we had a type-based trivial copying function, we could explicitly verify that the type was assignable, and if it is not issue a compile error. It could also be used to trivially copy arrays of data. `std::copy` could do so as well, but having a function specifically for trivial copies makes it clear to the reader that the code is intended to take advantage of doing just a `memcpy`.

# Design

The standard library will include the following functions. They should be part of the language support library (chapter 18).

## Trivial Construction In-Place

These functions construct an object in place, but they do so in such a way that the object's initial value representation shall be the data currently residing in that memory. These functions require `T` to be either:

1. Trivially default constructible.
2. Trivially copyable and CopyConstructible.

`T` will be initialized as if by a trivial constructor call.

Note that while these functions could in theory be used to achieve type-punning (conversion of `int` to `float` by bits), such an operation would still be UB. The value representation of an `int` is different from that of a `float`. And the function can only work if the value representation stored in the memory matches that required for the `Type`.

It could however permit you to perform punning if source and destination types are layout compatible (so long as the destination is trivially default constructible too). Strict aliasing rules are preserved, as this reuses the storage and therefore ends the lifetime of the prior objects in that storage.

````
template<typename T>
T *trivial_construct_in_place(void *ptr);
````

Requires: `ptr` points to storage that contains `sizeof(T)` bytes of storage. `ptr` must be aligned to at least `alignof(T)`. `ptr` shall store the value representation of an object of type `T`.

Returns: A pointer to a `T` object which reuses the storage referenced by `ptr`. The object representation of `T` shall be exactly the same values that were stored in `ptr`, up to `sizeof(T)` bytes. `T` will be initialized as if by a trivial constructor call.

Remarks: This function does not participate in overload resolution unless `T` is either trivially default constructible or both trivially copyable and CopyConstructible. If `T` is trivially copyable, then its default constructor will not be called. The storage starting at `ptr` and ending `sizeof(T)` bytes after this address are reused for `T`, so any such objects in that storage have their lifetimes ended. [note: This means all proscriptions about using pointers/references/names to the old object(s) apply [basic.life].]

````
template<typename T>
T *trivial_construct_in_place(void *ptr, size_t count);
````

Equivalent to:

````
auto mem = reinterpret_cast<unsigned char*>(ptr);
T *ret = nullptr;
for(size_t i = 0; i < count; ++i)
{
  auto obj = trivial_construct_in_place(mem);
  if(!ret)
	ret = obj;
  mem += sizeof(T);
}

return ret;
````

Requires: `count` shall not be zero.

Remarks: This function does not participate in overload resolution unless `T` is either trivially default constructible or both trivially copyable and CopyConstructible. If `T` is trivially copyable, then its default constructor will not be called.

## Trivial Copy Construct

This function constructs a prvalue via trivial copy construction, from a region of storage containing `T`'s value representation. `T` is required to be trivially copyable and CopyConstructible.


````
template<typename T>
T trivial_copy_construct(void *ptr);
````

Requires: `ptr` points to storage that contains at least `sizeof(T)` bytes of memory. `ptr` shall store the value representation of an object of type `T`.

Returns: A prvalue of type `T` whose value representation is equivalent to the storage currently in `ptr`.

Remarks: This function does not participate in overload resolution unless `T` is a trivially-copyable type and is CopyConstructible.

Note that the implementation of this function must take into account the following possibility:

````
new(ptr) auto(trivial_copy_construct<T>(ptr));
````

That would effectively be the equivalent of `trivial_construct_in_place`.

## Trivial Copy Assignment

These functions perform trivial copy assignment, and fail for any types `T` which are not both trivially-copyable and Assignable.

````
template<typename T>
void trivial_copy_assign(T &dst, const T &src);
````

Equivalent to `memcpy(&dst, &src, sizeof(T));`.

Requires: `dst` and `src` shall be distinct objects. Neither `dst` nor `src` shall be base class subobjects.

Remarks: This function does not participate in overload resolution unless `T` is a trivially-copyable type and is Assignable.

````
template<typename T>
void trivial_copy_assign(T &dst, const void *src);
````

Equivalent to `memcpy(&dst, src, sizeof(T));`.

Requires: `dst` shall be a distinct range of memory from `src` + `sizeof(T)`. `src` shall contain at least `sizeof(T)` bytes. `src` shall store the value representation of an object of type `T`. `dst` shall not be a base class subobject. If `src` points to an object of type `T`, then it shall not be a base class subobject.

Remarks: This function does not participate in overload resolution unless `T` is a trivially-copyable type and is Assignable.

````
template<typename T>
void trivial_copy_assign(T *dst, const void *src, size_t count);
````

Equivalent to `memcpy(dst, src, sizeof(T) * count);`. [note: It therefore has the same overlapping restrictions of `std::memcpy`.]

Requires: `dst` shall not be a base class subobject. `src` shall contiguously store the value representation of `count` objects of type `T`. [note: Per the use of `memcpy`, this function implicitly requires that the address range of `dst` to `dst + count` must not overlap the range of `src` to `src + count * sizeof(T)`.] `count` shall be greater than 0.

Remarks: This function does not participate in overload resolution unless `T` is a trivially-copyable type and is Assignable.

````
template<typename T>
void trivial_copy_assign_relax(T *dst, const void *src, size_t count);
````

Equivalent to `std::memmove(dst, src, sizeof(T) * count)`. [note: `memmove` lacks the overlap restrictions of `std::memcpy`.]

Requires: `dst` shall not be a base class subobject. `src` shall contiguously store the value representation of `count` objects of type `T`.

Remarks: This function does not participate in overload resolution unless `T` is a trivially-copyable type and is Assignable.







# Impact on the Standard

These functions should not break compatibility, nor should they be constructs that require the compiler to violate its object model.



# Acknowledgments:


