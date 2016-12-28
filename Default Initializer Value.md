% Default Initializer Value
%
%

This proposal creates a language mechanism for performing default initialization on objects in more circumstances than the present standard allows. It provides a way to default initialize objects created by forwarding functions like `emplace`. It also provides a way for standard library containers to be told to perform default initialization on their elements, in a way that mirrors the generalized default initialization mechanism.

It also provides a way to make constructors which do not initialize the object, similar to how default initialization through a trivial default constructor works.

# Motivation and Scope

## Default Initializing Containers

Default initialization can be dangerous. But it can also be very useful for performance. Consider the following code:

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

This has two problems. First, it relies on `genData`'s ability to work with iterators. If this were a C function, that would not be possible. Equally importantly, it is still quite inefficient.

`back_insertion_iterator` ultimately calls `vector::push_back`. And this function must check the capacity to see if it needs to expand it when inserting. But for our needs, that will never happen; we guaranteed that with the call to `reserve`. So it is just wasted performance, relative to our optimal code.

It is good that `vector` and other containers do not default to default initialization of its contents. But it is not good that there is no way to permit default container initialization at all, even when that is precisely what the user wants.

C APIs are hardly the only APIs that fill in object values in-situ. `operator>>` with iostreams are expected to be provided with live objects. Reading a number of integers from a stream and storing them in an array is a reasonable operation. With a `std::array` or an `int[]`, the array will be uninitialized. But if you use a `vector`, the values will have to be at least value-initialized. This wastes performance.

## Indirect Default Initialization

The various `emplace` functions, along with their `in_place_t`-based variations, represent an effective way to defer initialization of an object. They effectively avoid "initialize-and-copy" methodology that sometimes happens in C++.

However, even if you pass no parameters, they will value-initialize the data. Again, it is good that this is the default. But there are times when what you really want is to default initialize the data, with the expectation that other code will give it useful values.

## Uninitialized Values From Non-Default Initialization

Having a default constructor and/or default member initializers perform initialization is very much akin to making value initialization be the default form of initialization in container insertion operations.

The problem with doing so is that it becomes impossible to allow the object to be uninitialized. If the default constructor exists and is non-trivial (or the class has DMIs), then there is no way to create uninitialized instances of the class. That may be what you want in some cases. But sometimes, you want to have the default constructor initialize the object, while having some other constructor leave the object (and its subobjects) uninitialized.

It would be useful to have constructors that can construct the object without initializing its members. Obviously, this would be an opt-in feature, and the parameters to such a constructor would be irrelevant.

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

DefaultInit d = std::default_init;
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

1. The user wants to default initialize `d1` in its entirety.
2. The user wants to default initialize `d1.a`, but `d1.b` should be value-initialized, in accord with regular aggregate initialization rules.

We already have a similar ambiguity:

`````
struct Convert
{
	operator int() {return 5;}
	operator Agg() {return {20, 40};}
};

Agg c{Convert{}};
`````

This could use one of two conversion pathways: convert to a type compatible with the aggregate being initialized, or convert to a type compatible with the first member of the aggregate. C++ selects the second one, in accord with the sequence of rules in [dcl.init.list]/3. Rule 3.1 only checks for types which are the type being initialized or a type derived from it. Since `Convert` is neither, the initialization is sent to [dcl.init.aggr], where the first value tries to map to the first aggregate member.

Therefore, we resolve the ambiguity with `default_init_t` in the same way: list initialization treats the type as applying to the first argument of the aggregate, not the aggregate itself.

Of course, as a consequence of this rule, if the type being initialized is not an 	aggregate, using `{std::default_init}` will do something unusual. It will attempt to select constructors based on overload resolution. Given the above rules for overload resolution of `std::default_init_t` then if there is more than one constructor which can be called with one parameter, then the program is ill-formed due to overload resolution conflict.

The one exception is if the first parameter of one of the constructors is of the type `std::default_init_t` itself. In which case, it will be called correctly.

We could of course change list initialization rules to check for initialization from a single `default_init_t` value, but after passing over to aggregate initialization. However, this could be considered quite incongruous.

List initialization conceptually means to initialize an object from a potentially-heterogeneous sequence of values. This is part of the reason why we want `{std::default_init}` to default initialize the first member rather than the object itself. Calling a constructor is simply a way to provide those values to an object's initialization.

Therefore, we should apply the default initialization to the constructor's parameter, not to the object itself.

## Uninitialized Constructor

A constructor can be declared to be an uninitialized constructor with the following syntax:

`````
struct A
{
	A(some_type) = <<uninit_syntax>>;
};
`````

Where `<<uninit_syntax>>` is a keyword, special identifier, or other unique symbol.

An uninitialized constructor is a trivial constructor that will not initialize any of its subobjects. Any default member initializers of non-static data members will not be used.

Declaration of an uninitialized constructor for a type is ill-formed if any of the following is true:

1. The type has virtual member functions or base classes.
2. Any subobjects with non-trivial default constructors do not also have at least one uninitialized constructor.

The idea with rule #2 is that, by declaring a type to have an uninitialized constructor, you are saying that it is OK for this object to be constructed in an uninitialized way. So if a subobject class has default member initializers (and thus a non-trivial default constructor), a user can declare that bypassing those initializers is OK through the use of an uninitialized constructor.

We may want the uninitialized constructor to have to be accessible to the type in question. That way, if a user wants only private code to be able to create an uninitialized instance of that type, then they can do that.

This syntax also allows users to force a default constructor to be trivial, even if the type has default member initializers (they would be used for any non-uninitializing constructor). Such a declaration would also error out if the type cannot be uninitialized. Also, such a declaration would still count as a `defaulted` default constructor, so the type could still be an aggregate.

Note that the lifetime rules in [basic.life] should be updated, such that initializing a class via a call to an uninitialized constructor counts as vacuous initialization.

## Library Changes

Containers should be changed to have specific overloads for `default_init_t`-based initialization. Specifically, the following should have an overload that takes `default_init_t`:

* All `push_x`-style functions. These add a single uninitialized value. This includes the container adapter types.
* All sized constructors, `resize`, and sized `insert` functions.
* Single-element `insert` functions for sequence containers. Adding `default_init_t` overloads for associative or unordered containers is problematic.

There is no need to add overloads for `emplace`-style construction, as forwarding the value ought to handle that adequately. Similarly, there is no need for overloads for in-place constructors (for `any`, `optional`, or `variant`).

Additionally, output iterators ought to have an overload for taking `default_init_t` directly, which can apply default initialization to the created element in-situ. This can be more efficient than default initializing an element, then copying it into the output.

# Alternatives Considered

## 



# Impact on the Standard

This adds a new type and a new value to the `std` namespace. It adds a number of rules surrounding object initialization, but only with regard to the new type. Depending on how we want `<<uninit_syntax>>` to work, we may need a keyword or a special identifier.

In any case, all such changes should be backwards compatible.

# Acknowledgments

* Sean Middleditch, for his alternative idea and some impetus for cleanup. Also, for the uninitialized constructor idea.


