Title: Trivial Operations

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

But as far as the C++ object model is concerned, `mem` does not contain `Type`. So let us consider the common next statement:

````
auto ptr = static_cast<Type*>(mem);
````

The use of `ptr` represents undefined behavior, since `mem` does not in fact contain `Type`. This kind of casting is frequently done in low-level code anyway, because `Type` is trivially default constructible and so long as the value representation stored in `ptr` matches `Type`'s value representation, it ought to work.

But that doesn't make it any less UB, as far as the standard is concerned.

Note that `new(mem) Type` is not guaranteed to *preserve* the contents of `mem`. Debug builds often write arbitrary values, even during placement new operations, and this behavior is permitted by the standard.

What we need is a function which will begin the lifetime of `Type`, but preserve the contents of the memory it is being built within. However, it also needs to do this only for types where it is reasonable to be able to do so.

P0593 has some interaction with this part of the proposal. However, it does not make this part unnecessary in all cases, such as when converting from a `U` to a `T` in-situ. P0593 can only convert from an array of bytes into a `T`. Similarly, `std::bitcast` cannot perform an in-situ object conversion while preserving the value representation.

## Trivial Copy Construct

When reading memory from an arbitrary source whose value representation matches a C++ object, the typical idiom for accessing it is to do this:

````
T t;
memcpy(&t, src, sizeof(T));
````

While the aforementioned ability to construct an object within that memory would allow us to handle this in-situ, perhaps we need to reuse `src` or maybe it is memory that does not belong to us. Regardless of why, we want to construct an object by copying its object representation from the contents of that memory.

The above idiom imposes a restriction on `T`; it requires not only that it is trivially-copyable but that it is Assignable. For example:

````
struct NoAssign
{
	const int i;
};
````

The aforementioned code will compile with `NoAssign`, but it will invoke UB due to changing the value of a `const` object.

What we really need is the equivalent of a copy constructor that performs copy-construction on `NoAssign`, but from a piece of memory rather than a live `NoAssign` instance.

This behavior is similar from `std::bitcast`. The difference is that `bitcast` copies from a live object `T` to a prvalue of type `U`. This conversion is explicitly from a byte array to a prvalue of type `U`.

## Trivial Copy Assignment

The use of `memcpy` on trivially copyable types is permitted by the standard. However, it is a bit cumbersome to write:

````
std::memcpy(&dstObj, &srcObj, sizeof(Typename));
````

Some repetition has to be done here, since `sizeof(Typename)` is deducible from the actual types of the memory.

But more importantly, there are types which are trivially copyable yet are not Assignable, like the aforementioned `NoAssign`:

````
struct NoAssign
{
	const int i;
};
````

To perform trivial copy assignment on `NoAssign` violates the prohibition on changing a `const` object. As such, `NoAssign` has `delete`d copy and move assignment operators. But `memcpy` will compile just fine, allowing us to encounter undefined behavior, even though the error is statically detectable.

If we had a type-based trivial copying function, we could explicitly verify that the type was assignable, and if it is not issue a compile error. It could also be used to trivially copy arrays of data. `std::copy` could do so as well, but having a function specifically for trivial copies makes it clear to the reader that the code is going to merely call `memcpy` or `memmove`.

# Design

## Common

At present, trivial copying (as defined in [basic.types]/2-3) is only permitted in two cases:

1. Copying from a `T` to an arbitrary byte array and back to a `T`.
2. Copying from one `T` to another `T`.

All of the functions here will instead work in a fashion similar to `std::bitcast`. Namely, the functions which take a sequence of bytes and create/copy to one or more `T`'s have undefined behavior if the input sequence of bytes does not represent legal object representation for the destination object type `T`. And if those bytes could have multiple object representations in `T`, then which one is returned is unspecified.

## Trivial Construction In-Place

These functions construct an object in place, but they do so in such a way that the object's initial value representation shall be the data currently residing in that memory. These functions require `T` to be either:

* Trivially default constructible.
* Trivially copyable and CopyConstructible.

As such, these functions rely on the following concept:

````
template<typename T>
concept TrivialInPlaceConstruct<T> =
	(is_trivially_copyable_v<T> && is_copy_constructible_v<T>) ||
	is_trivially_default_constructible_v<T>
````

`T` will be initialized as if by a trivial constructor call: either the trivial default constructor or the trivial copy constructor.

````
template<typename T>
	requires TrivialInPlaceConstruct<T>
T *trivial_construct_in_place(span<byte, sizeof(T)> data);

template<typename T>
	requires TrivialInPlaceConstruct<T>
const T *trivial_construct_in_place(span<const byte, sizeof(T)> data);
````

Requires: `data.data()` shall be aligned to at least `alignof(T)`. `data` shall store the value representation of a valid instance of the type `T`.

Returns: A pointer to a `T` object which reuses the storage referenced by `data`. The object representation of `T` shall be exactly the same bytes that were stored in `data`. `T` will be initialized as if by a trivial constructor call.

Complexity: O(1) with respect to `sizeof(T)`. [note: Implementations are not allowed to implement these functions by doing a pair of `memcpy`s.]

Remarks: If `T` is trivially copyable and CopyConstructible, then it will be created as if by its copy constructor. The storage designated by `data` reused for `T`, so any such objects in that storage have their lifetimes ended. [note: This means all proscriptions about using pointers/references/names to the old object(s) apply [basic.life].]

````
template<typename T, ptrdiff_t Extent = dynamic_extent>
	requires TrivialInPlaceConstruct<T>
span<T, Extent> trivial_construct_array_in_place(span<byte,
	Extent == dynamic_extent ? dynamic_extent : (sizeof(T) * Extent)> data);
	
template<typename T, ptrdiff_t Extent = dynamic_extent>
	requires TrivialInPlaceConstruct<T>
span<const T, Extent> trivial_construct_array_in_place(span<const byte,
	Extent == dynamic_extent ? dynamic_extent : (sizeof(T) * Extent)> data);
````

Equivalent to:

````
auto mem = reinterpret_cast<byte*>(ptr); //Or `const byte*`
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

Requires: `data.size()` shall not be zero. `data.size()` shall be a multiple of `sizeof(T)`. `data` shall contiguously store `count` value representations of valid instances of the type `T`. `data.data()` must be aligned to at least `alignof(T)`.

Returns: A pointer to an array of `data.size() / sizeof(T)` objects of type `T`.

Complexity: O(1) with respect to `sizeof(T)` and with respect to `data.size()`. [note: Implementations are not allowed to implement these functions by doing a pair of `memcpy`s.]

Remarks: If `T` is trivially copyable, then its default constructor will not be called.

## Trivial Copy Construct

This function constructs a prvalue via trivial copy construction, from a region of storage containing `T`'s value representation. `T` is required to be trivially copyable and CopyConstructible.

````
template<typename T>
	requires is_trivially_copayble_v<T> && is_copy_constructible_v<T>
constexpr T trivial_copy_construct(span<const byte, sizeof(T)> data);
````

Requires: `data` shall store the value representation of a valid instance of the type `T`.

Returns: A prvalue of type `T` whose value representation is equivalent to the storage currently in `data`.

Note that the implementation of this function must take into account the following possibility:

````
new(ptr) auto(trivial_copy_construct<T>(data.data()));
````

That would effectively be the equivalent of `trivial_construct_in_place`.

## Trivial Copy Assignment

These functions perform trivial copy assignment to and/or from live instances of `T`. These functions require that `T` is trivially copyable. When copying to existing `T` objects, the functions also require that `T` is Assignable.

As such, they rely on the following concept definition:

````
template<typename T>
concept TrivialCopyAssignable =
	is_trivially_copyable_v<T> && is_copy_assignable_v<T>
````

These are effectively convenience functions, wrappers around `memcpy` and `memmove` that do some basic type-checking to ensure that they are only used on types that support the operations.

````
template<typename T>
	requires TrivialCopyAssignable<T>
constexpr void trivial_copy_assign(T &dst, const T &src);
````

Requires: `dst` and `src` shall be distinct objects. Neither `dst` nor `src` shall be base class subobjects.

Effects: Equivalent to `memcpy(&dst, &src, sizeof(T));`.

````
template<typename T>
	requires TrivialCopyAssignable<T>
constexpr void trivial_copy_assign(T &dst, span<const byte, sizeof(T)> src);
````

Requires: The object `dst` shall be a distinct range of memory from `src`. `src` shall store the value representation of a valid instance of the type `T`. `dst` shall not be a base class subobject.

Effects: Equivalent to `memcpy(&dst, src, sizeof(T));`.

````
template<typename T, ptrdiff_t Extent = dynamic_extent>
	requires TrivialCopyAssignable<T>
constexpr void trivial_copy_assign(span<T, Extent> dst,
	span<const byte, Extent == dynamic_extent ? dynamic_extent : (sizeof(T) * Extent)> src);
````

Requires: `dst` shall not point to a base class subobject of type `T`. `dst.size()` shall not be zero. `src` shall contiguously store `dst.size() * sizeof(T)` value representations of valid instances of the type `T`. [note: Per the use of `memcpy`, this function implicitly requires that the address range of `dst` to `dst + count` must not overlap the range of `src` to `src + count * sizeof(T)`.]

Effects: Equivalent to `memcpy(dst.data(), src.data(), src.size());`. [note: It therefore has the same overlapping restrictions of `std::memcpy`.]

````
template<typename T, ptrdiff_t Extent = dynamic_extent>
	requires TrivialCopyAssignable<T>
constexpr void trivial_copy_assign_relax(span<T, Extent> dst,
	span<const byte, Extent == dynamic_extent ? dynamic_extent : (sizeof(T) * Extent)> src);
````

Requires: `dst.size()` shall not be zero. `dst` shall not point to a base class subobject of type `T`. `src` shall contiguously store `count` value representations of valid instances of the type `T`.

Effects: Equivalent to `std::memmove(dst.data(), src, sizeof(T) * count)`. [note: `memmove` lacks the overlap restrictions of `std::memcpy`.]

Remarks: This function does not participate in overload resolution unless `T` is a trivially-copyable type and is Assignable.


````
template<typename T>
void trivial_copy_from(std::byte *dst, const T &src);
````

Requires: `dst` shall point to storage containing at least `sizeof(T)` bytes. `dst` + `sizeof(T)` shall be a distinct range of memory from `src`.

Effects: Equivalent to `std::memcpy(dst, &src, sizeof(T))`;

Remarks: This function does not participate in overload resolution unless `T` is a trivially-copyable type.

````
template<typename T>
void trivial_copy_from(std::byte *dst, const T *src, size_t count);
````
Equivalent to `memcpy(dst, src, sizeof(T) * count);`.

Requires: `count` shall be greater than 0. `src` shall not be a base class subobject. `dst` shall point to `count * sizeof(T)` bytes of storage. `src` shall point to a sequence of at least `count` objects of type `T`. [note: Per the use of `memcpy`, this function implicitly requires that the address range of `dst` to `dst + count` must not overlap the range of `src` to `src + count * sizeof(T)`.]

Remarks: This function does not participate in overload resolution unless `T` is a trivially-copyable type.

There is no `relax` version of `trivial_copy_from`. The reason for this is that the `trivial_copy_from` functions are specifically meant for copying to byte arrays (which is why they take `std::byte *` instead of `void*`). If you're writing to part of the same region of memory, then you must also be assigning to some `T` objects. In which case, you almost certainly meant to call `trivial_copy_assign`.


# Impact on the Standard

These functions should not break compatibility, nor should they be constructs that require the compiler to violate its object model. The language feature should also not affect compatibility.

# Possible Improvements

Use `span` and `byte` where appropriate. The new prototypes would be:

````
//trivial_construct_in_place
template<typename T>
T *trivial_construct_in_place(span<byte, sizeof(T)> data);

template<typename T>
const T *trivial_construct_in_place(span<const byte, sizeof(T)> data);

//Array forms get a new name, since the function no longer needs an explicit count.
template<typename T, ptrdiff_t Extent = dynamic_extent>
span<T, Extent> trivial_construct_array_in_place(span<byte,
	Extent == dynamic_extent ? dynamic_extent : (sizeof(T) * Extent)> data);
	
template<typename T, ptrdiff_t Extent = dynamic_extent>
span<const T, Extent> trivial_construct_array_in_place(span<const byte,
	Extent == dynamic_extent ? dynamic_extent : (sizeof(T) * Extent)> data);

//trivial_copy_construct
template<typename T>
constexpr T trivial_copy_construct(span<const byte, sizeof(T)> data);

//trivial_copy_assign
template<typename T>
constexpr void trivial_copy_assign(T &dst, span<const byte, sizeof(T)> src);

template<typename T, ptrdiff_t Extent = dynamic_extent>
constexpr void trivial_copy_assign(span<T, Extent> dst,
	span<const byte, Extent == dynamic_extent ? dynamic_extent : (sizeof(T) * Extent)> src);

template<typename T, ptrdiff_t Extent = dynamic_extent>
constexpr void trivial_copy_assign_relax(span<T, Extent> dst,
	span<const byte, Extent == dynamic_extent ? dynamic_extent : (sizeof(T) * Extent)> src);

//trivial_copy_from
template<typename T>
constexpr void trivial_copy_from(const T &src, span<byte, sizeof(T)> dst);

template<typename T, ptrdiff_t Extent = dynamic_extent>
constexpr void trivial_copy_from(span<const T, Extent> src, span<byte,
	Extent == dynamic_extent ? dynamic_extent : (sizeof(T) * Extent)> dst);
````


# Acknowledgments:

* Matthew Woehlke 
* Jens Maurer
* Thiago Macieria


