% Trivial Operations
%
%

The C++ concepts of trivially copyable and trivial types are very useful for low-level code. However, employing them in a number of circumstances often require heavy use of casting, the use of more opaque functions like `memcpy`, or even relying on undefined behavior.

This proposal includes a number of standard library functions that improve upon such constructs, making them well-defined C++ without violating the C++ object model.

# Motivation

This proposal offers to solve a number of cases in a C++-compliant way.

## Construction In-Place

At present, placement-`new` can be employed to begin the lifetime of an object in a piece of memory:

````
auto mem = std::malloc(sizeof(Type));
auto ptr = new(mem) Type;
````

This performs default-initialization on `Type`. If `Type` is trivially default constructible, then the values within `Type` will be undefined.

But sometimes, having such values be undefined is not what you want:

````
auto mem = std::malloc(sizeof(Type));
fill_in_memory(mem);
````

`fill_in_memory`, in this case, is a function which put something into that memory. This may have read data from a file-stream, it may be a packet from a TCP/IP socket, whatever. However, we as the user know exactly what `fill_in_memory` wrote. And using standard layout rules, we have constructed `Type` (for this platform) to match the value representation of that data. So logically, `mem` now contains `Type`.

But as far as the C++ object model is concerned, `mem` does not contain `Type`. Therefore:

````
auto ptr = static_cast<Type*>(mem);
````

The use of `ptr` represents undefined behavior. This is frequently done in low-level code anyway, because `Type` is trivially default constructible and therefore logically any value representation is legitimate. But that doesn't make it any less UB.

Note that `new(mem) Type` is not guaranteed to preserve the contents of `mem`. Debug builds often write arbitrary values, even during placement new operations, and this behavior is permitted by the standard.

What we need is a function which will begin the lifetime of `Type`, but preserve the contents of the memory it is being built within. This operation is only legitimate for types that are trivially default constructible, since such types can reasonably assume any value representation (since they can be legally constructed with uninitialized data).

## Copy Into New Object



## Trivial Copying

The use of `memcpy` on trivially copyable types is permitted by the standard. However, it is a bit cumbersome to write:

````
std::memcpy(&dstObj, &srcObj, sizeof(Typename));
````

Some repetition is being done here, since `sizeof(Typename)` is deducible. But more importantly, there are types which are trivially copyable yet are not CopyAssignable:

````
struct Type
{
	const int i;
};
````

To perform trivial copy assignment on `Type` violates the prohibition on changing a `const` object. As such, `Type` has `delete`d copy and move assignment operators. But `memcpy` will work just fine.

If we had a type-based trivial copying function, we could explicitly verify that the type was assignable, and if it is not issue a compile error.

# Design

The standard library will include the following functions. They should be part of the language support library (chapter 18).


----

Trivial default construction in-place, where the object's initial object representation is the data currently residing in that memory:

````
template<typename T>
T *trivial_construct_in_place(void *ptr);
````

Requires: `ptr` points to storage that contains `sizeof(T)` bytes of storage. `ptr` must be aligned to at least `alignof(T)`. `ptr` shall store the value representation of an object of type `T`.

Returns: A pointer to a `T` object which reuses the storage referenced by `ptr`. The object representation of `T` shall be exactly the same values that were stored in `ptr`, up to `sizeof(T)` bytes.

Remarks: This function does not participate in overload resolution unless `T` is trivially default constructible. The storage starting at `ptr` and ending `sizeof(T)` bytes after this address are reused for `T`, so any such objects in that storage have their lifetimes ended. [note: This means all proscriptions about using pointers/references/names to the old object(s) apply [basic.life] .]

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

Remarks: This function does not participate in overload resolution unless `T` is trivially default constructible.

----

Copy assignment, both with individual values and arrays:

````
template<typename T>
void trivial_copy_assign(T &dst, const T &src);
````

Equivalent to `memcpy(&dst, &src, sizeof(T));`.

Requires: `dst` and `src` shall be distinct objects. `dst` must not be a base class subobject.

Remarks: This function does not participate in overload resolution unless `T` is a trivially-copyable type and is Assignable.

````
template<typename T>
void trivial_copy_assign(T &dst, const void *src);
````

Equivalent to `memcpy(&dst, src, sizeof(T));`.

Requires: `dst` shall be a distinct range of memory from `src` + `sizeof(T)`. `src` shall contain at least `sizeof(T)` byptes. `dst` must not be a base class subobject.

Remarks: This function does not participate in overload resolution unless `T` is a trivially-copyable type and is Assignable.

````
template<typename T>
void trivial_copy_assign(T *dst, const void *src, size_t count);
````

Equivalent to `std::memmove(dst, src, sizeof(T) * count)`. [note: `memmove` lacks the overlap restrictions of `std::memcpy`.]

Requires: `dst` shall not be a base class subobject.

Remarks: This function does not participate in overload resolution unless `T` is a trivially-copyable type and is Assignable.

````
template<typename T>
void trivial_copy_assign_strict(T *dst, const void *src, size_t count);
````

Equivalent to `memcpy(dst, src, sizeof(T) * count);`. [note: It therefore has the same overlapping restrictions of `std::memcpy`. Hence the "_strict" suffix.]

Requires: `dst` shall not be a base class subobject.

Remarks: This function does not participate in overload resolution unless `T` is a trivially-copyable type and is Assignable.

----

Direct construction via a trivial copy operation:

````
template<typename T>
T trivial_copy_construct(void *ptr, size_t sz);
````

Requires: `ptr` points to storage that contains at least `sz` bytes of memory. `sz` shall be at least as large as `sizeof(T)`.

Returns: A prvalue of type `T` who's object representation is equivalent to the storage currently in `ptr`.

Remarks: This function does not participate in overload resolution unless `T` is a trivially-copyable type and is CopyConstructible.





# Impact on the Standard

These functions should not break compatibility, nor should they be constructs that require the compiler to violate its object model.



# Acknowledgments:


