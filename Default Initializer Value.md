% Default Initializer Value
%
%

This proposal creates a language mechanism for performing default initialization on objects in more circumstances than the present standard allows. It provides a way to default initialize objects created by forwarding functions like `emplace`. It also provides a way for standard library containers to be told to perform default initialization on their elements, in a way that mirrors the generalized default initialization mechanism.

# Motivation and Scope

## Default Initializing Containers

Default initialization can be dangerous, since it can cause trivial types to have uninitialized values. But this can also be very useful for performance. Consider the following code:

`````
int anArray[256];

genData(anArray, 256);
`````

The `genData` function generates data for the array range it is given. This code overall is quite efficient. The compiler will allocate storage for 256 `int`s on the stack. But it will not initialize that storage. This is because it uses default initialization on the array, and `int` is a type for which performing default initialization means that the object goes uninitialized.

However, there is a problem when it comes to using standard library containers:

`````
std::vector<int> aVec(256);

genData(aVec.data(), aVec.size());
`````

This is different. It performs value initialization on each element of the `vector`. Sometimes, this is exactly what you want, but not in this case. Here, we are value-initializing those elements, only to immediately overwrite them.

For our use case, this is decidedly non-optimal. And the more array elements there are, the less optimal it gets.

One could argue that we could achieve the same effect by proper use of iterators:

`````
std::vector<int> aVec;
aVec.reserve(256);

genData(std::back_inserter(aVec), 256);
`````

This has two problems. First, it relies on `genData`'s ability to work with iterators. If this were a C function, that would not be possible. And there are many C++ functions that simply cannot work with iterators: `std::to_chars` being a prime example.

Equally importantly, it is still quite inefficient. `back_insertion_iterator` ultimately calls `vector::push_back`. And this function must check the capacity to see if it needs to expand it when inserting. But for this particular use case, that will never happen; we guaranteed this at a higher level, thanks to the call to `reserve`. So the check is just wasted performance, relative to the optimal code.

It is good that `vector` and other containers do not default to default initialization of its contents. But it is not good that there is no way to permit default container initialization at all, even when that is precisely what the user wants.

C APIs are hardly the only APIs that fill in object values in-situ. The iostream overloads for `operator>>` are expected to be provided with live objects. `std::to_chars` works similarly, expecting a live array of `char`s. Reading a number of integers from a stream and storing them in an array is a reasonable operation. With a `std::array` or an `int[]`, the array can be uninitialized. But if you need to use a `vector` with a dynamic length, the values will have to be at least value-initialized. This wastes performance needlessly.

## Indirect Default Initialization

The various `emplace` functions, along with their `in_place_t`-based variations, represent an effective way to defer initialization of an object. They effectively avoid "initialize-and-copy" methodology that sometimes happens in C++.

However, even if you pass no parameters, they will still value-initialize the data. Again, it is good that this is the default. But there are times when what you really want is to default initialize the data, with the expectation that other code will give it useful values.

# Design

We first declare a new type in the standard library: `std::default_init_t`. This is a special type which the language looks for in certain circumstances. This is similar to how `std::nullptr_t`'s behavior in the standard. For example, it can be converted to an integer type or a pointer type, but `nullptr_t` does not have `operator int` or `operator T*` overloads. The conversion happens because the language says so, and it is not considered an overloaded conversion operator.

`std::default_init_t` will be:

1. Trivial type
2. `==` and `!=` with itself. All instances are considered equal to each other.

`std::default_init` shall be a `const`-qualified value of type `std::default_init_t`. Aside from copying or moving, it is the only way to get a value of `std::default_init_t` type. This is much like `std::in_place_t`'s relationship to `std::in_place`.

A value of type `std::default_init_t` works like a normal C++ object, except in ways that will be discussed below.

The list in [dcl.init]/17 shall be changed to add a new entry, probably after 17.2. If the initializer is a single value of type `std::default_init_t`, then the destination object will be default initialized (in accord with [dcl.init]/7). Also, if the type being initialized is a reference, then a default-initialized prvalue of the referenced type will be initialized and assigned to the reference.

As such, all of the following will perform default initialization:

`````
T t = std::default_init;
T t(std::default_init);
T t[23] = std::default_init;
new T(std::default_init);
`````

Note that this special initialization *overrides* constructor calls. So if you have this:

`````
struct DefaultInit
{
	DefaultInit() = default;
	DefaultInit(std::default_init_t);
};

DefaultInit d(std::default_init);
`````

`d` will be default initialized. It *will not* call the constructor taking a value of type `default_init_t`.

## Overload Resolution

This is a sticky point with `std::default_init_t`. We do not want it to be implicitly convertible to anything. It can however initialize any default initializable object.

So we want the following:

`````
void func1(int i);
void func2(int j);
void func2(float k);
void func3(int a);
void func3(float b);
void func3(std::default_init_t);

func1(std::default_init);
func2(std::default_init);
func3(std::default_init);
`````

We want the call to `func1` to work. But we want the call to `func2` to fail. And we want the call to `func3` to call the overload specifically taking `default_init_t`.

## List Initialization

The change above notably does not affect list initialization; [dcl.init]/17.1 sends all braced-init-list-bound objects to [dcl.init.list] for resolution.

This is a *deliberate* omission. Consider the following:

`````
struct Agg
{
	int a;
	int b;
};

Agg agg2{1, 2};
Agg agg1{1};
Agg agg0{};
`````

The rules of aggregate initialization say that `agg1.b` will be value initialized, as are all aggregate members which are not listed in the braced-init-list. So `agg0` will value initialize the whole object.

Given that, consider this:

`````
Agg d2{std::default_init, std::default_init};
Agg d1{std::default_init};
`````

For `d2`, the user's expectation is obvious: the user wants to default initialize the two *members* of the aggregate. But what about `d1`? There are two interpretations of this code:

1. The user wants to default initialize the object `d1` in its entirety.
2. The user wants to default initialize `d1.a`, but `d1.b` should be value-initialized, in accord with regular aggregate initialization rules.

As this is similar to an already existing ambiguity, this design uses the same resolution as that one. Consider the following code:

`````
struct Convert
{
	operator int() {return 5;}
	operator Agg() {return {20, 40};}
};

Agg c(Convert{});
Agg d{Convert{}};
`````

`c` will be initialized by copy construction, via an implicit conversion from `Convert` to `Agg`. `d` is a seemingly more complex case.

`d` could follow `c`'s pathway. But because the first member of `Agg` is an `int`, it could instead implicitly convert `Convert{}` into an `int` and perform aggregate initialization. C++ selects the aggregate initialization option, in accord with the sequence of rules in [dcl.init.list]/3. Rule 3.1 only checks for types which are the type being initialized or a type derived from it. Since `Convert` is neither, the initialization is sent to [dcl.init.aggr], where the first value tries to map to the first aggregate member.

Given this precedent, we resolve the ambiguity with `default_init_t` in the same way: `{std::default_init}` treats the type as applying to the first argument of the aggregate, not the aggregate itself.

Of course, as a *consequence* of this rule, if the type being initialized is not an 	aggregate, using `{std::default_init}` will do something unusual. It will attempt to select constructors based on overload resolution. Given the above rules for overload resolution of `std::default_init_t`, if there is more than one constructor which can be called with one parameter, then the program is ill-formed due to overload resolution conflict.

The one exception is if the first parameter of one of the constructors is of the type `std::default_init_t` itself. In which case, it will be called correctly.

We could of course change list initialization rules to check for initialization from a single `default_init_t` value, but only after passing over to aggregate initialization. However, this could be considered quite incongruous. The current way it works seems to work within an interpretation of what list initialization means.

List initialization conceptually means to initialize an object from a potentially-heterogeneous sequence of values. Calling a constructor is simply a way to provide those values to an object's initialization. This helps explain list-initialization's preference for `initializer_list` constructors; it isn't about calling constructors, it is about treating the object as a sequence of values.

Given that understanding of list-initialization, it makes sense to have `{std::default_init}` to default initialize the first member rather than the object itself. Which means default initializing the first parameter of a constructor.

## Library Changes

The language itself takes care of `allocator` and `allocator_traits` issues. Calling `construct(std::default_init_t)` will perform default initialization automatically. So all of the `emplace`-style APIs are automatically functional with `std::default_init`. So there are fewer forward-facing library APIs that need explicit overloads.

Containers (including `basic_string`) should be changed to have specific overloads for `default_init_t`-based initialization. Specifically, the following should have an overload that takes `default_init_t`:

* Single-element `insert` functions for sequence containers. Associative/unordered container inserts always insert a specific value.
* All sized constructors, `resize`, and sized `insert` functions.

Note that we do not need overloads for `push_x` functions, as they all have `emplace_x` alternatives that work quite well with `std::default_init`.

# Impact on the Standard

This adds a new type and a new value to the `std` namespace. It adds a number of rules surrounding object initialization, but only with regard to the new type.

In any case, all such changes should be backwards compatible.

# Acknowledgments



