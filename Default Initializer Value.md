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

The `genData` function generates data for the array range it is given. This code overall is both perfectly safe and quite efficient. The compiler will allocate storage for 256 `int`s on the stack. But it will not initialize that storage. This is because it uses default initialization on the array, and `int` is a type for which performing default initialization means that the object goes uninitialized.

However, this efficiency cannot be replicated when it comes to using standard library containers:

`````
std::vector<int> aVec(256);

genData(aVec.data(), aVec.size());
`````

This is different. It performs value initialization on each element of `aVec`. Sometimes, this is exactly what you want, but not in this case. Here, we are value-initializing those elements, only to immediately overwrite them with new values.

For our use case, this is decidedly non-optimal. And the more array elements there are, the less optimal it gets.

One could argue that we could achieve the same effect by proper use of iterators:

`````
std::vector<int> aVec;
aVec.reserve(256);

genData(std::back_inserter(aVec), 256);
`````

This has two problems. First, it relies on `genData`'s ability to work with iterators. If this were a C function, that would not be possible. And there are many C++ functions that simply cannot work with iterators: `std::to_chars` being a prime example.

Equally importantly, it is still less efficient than the original code. `back_insertion_iterator` ultimately calls `vector::push_back`. And this function must check the capacity to see if it needs to expand it when inserting. But for this particular use case, that will never happen; we guaranteed this at a higher level, thanks to the call to `reserve`. So the check is just wasted performance, relative to the optimal code.

It is good that `vector` and other containers do not default to default initialization of its contents. But it is not good that there is no way to permit default container initialization at all, even when that is precisely what the user wants and the user has ensured that it will be safe.

C APIs are hardly the only APIs that fill in object values in-situ. The iostream overloads for `operator>>` are expected to be provided with live objects. `std::to_chars` works similarly, expecting a live array of `char`s. Reading a number of integers from a stream and storing them in an array is a reasonable operation. With a `std::array` or an `int[]`, the array can be uninitialized. But if you need to use a `vector` with a dynamic length, the values will have to be at least value-initialized. This wastes performance needlessly.

## Indirect Default Initialization

The various `emplace` functions, along with their `in_place_t`-based variations, represent an effective way to defer initialization of an object. They effectively avoid "initialize-and-copy" methodology that sometimes happens in C++.

However, even if you pass no parameters, they will still value-initialize the data. Again, it is good that this is the default. But there are times when what you really want is to default initialize the data, with the expectation that other code will give it useful values.

## Prvalue Default Initialization

Creating a prvalue temporary can be done presently. However, there is no way to do it through default initialization. You have to use either `()` or `{}` to initialize a prvalue, and neither of these will perform default initialization.

This is particularly important in light of guaranteed elision. Prvalues represent initializers for objects, rather than being temporaries. As such, people are going to be more willing to use them for initialization purposes. So being able to explicitly request default initialization in such circumstances becomes more important.

This also makes it possible to use `auto` with such declarations, while preserving default initialization:

`````
auto var = some_type(std::default_init);
`````

This removes one of the "almost" parts of "almost always auto" declaration syntax.

## Aggregate Default Initialization

Consider aggregate initialization:

`````
struct Agg
{
	int a;
	int b;
};

Agg agg2{1, 2};
Agg agg1{1};
`````

`agg2` will initialize `agg2.a` and `agg2.b` to the given values. `agg1` will initialize `agg1.a` to the given value, but `agg2.b` will be value initialized, in accord with the existing initialization rules.

This is a good default, and sometimes this is exactly what you want. But at present, your choices are limited to "initialize with a value" or "value-initialize"; there is no way to explicitly request default initialization.

## Permits Better Warnings

As it currently stands, compilers try to warn on the use of uninitialized variables. While uninitialized variables are only problematic on their use, it would generally be better to warn on the creation of uninitialized objects directly. However, such variables are declared far too frequently to warn over.

Once we have a way to explicitly declare a variable to be uninitialized via default initialization, it makes more sense for compilers to warn on declarations that are not explicitly uninitialized. Of course, this will take time, but tools like Clang-Tidy make it possible to automate the process.

# Design

We first declare a new type in the standard library: `std::default_init_t`. This is a special type which the language looks for in certain circumstances. This is similar to `std::nullptr_t`'s behavior in the standard. For example, it can be converted to an integer type or a pointer type, but `nullptr_t` does not have `operator int` or `operator T*` overloads. The conversion happens because the language says so, and it is not considered an overloaded conversion operator.

`std::default_init_t` will be:

1. A TrivialType
2. Not DefaultConstructible
2. EqualityComparable, though instances are considered equal to each other.

`std::default_init` shall be a `const`-qualified value of type `std::default_init_t`. Aside from copying or moving, it is the only way to get a value of `std::default_init_t` type. This is much like `std::in_place_t`'s relationship to `std::in_place`.

A value of type `std::default_init_t` works like a normal C++ object, except in ways that will be discussed below.

The list of rules in [dcl.init]/17 shall be changed to add a new entry, immediately after 17.4. If the initializer is a single value of type `std::default_init_t`, then the destination object will be default initialized (in accord with [dcl.init]/7). Also, if the type being initialized is a reference, then a default-initialized prvalue of the referenced type will be created, a temporary manifested, and the temporary will be bound to the reference (observing the usual restrictions of temporaries bound to references).

As such, all of the following will perform default initialization:

`````
T t = std::default_init;
T t(std::default_init);
T t[23] = std::default_init; //Default initializes the array.
new T(std::default_init);
`````

Note that, because the above rule is inserted in [dcl.init]/17 before constructor calls, that rule takes priority. This is deliberate. So, if you have this:

`````
struct DefaultInit
{
	DefaultInit() = default;
	DefaultInit(std::default_init_t);
};

DefaultInit d(std::default_init);
`````

`d` will be default initialized. It *will not* call the constructor taking a value of type `default_init_t`.

Note that if the default constructor is explicit, then copy-initialization of the type through `default_init_t` is ill-formed:

`````
struct E { explicit E() = default; };

E e1(std::default_init); //Fine.
E e2 = std::default_init; //Ill-formed.
`````

## Overload Resolution

This is a sticky point with `std::default_init_t`.

The simplest way to handle this is to say that, while the invocation of the default initialization of the parameter is not an implicit conversion, for overload resolution purposes, we will treat it as if it was.

This gives us the following behavior:

`````
void func1(int i);
func1(std::default_init); //Default initializes `i`.

void func2(int j);
void func2(float k);
func2(std::default_init); //ill-formed. Equivalent "conversions".

void func3(int a);
void func3(float b);
void func3(std::default_init_t);
func3(std::default_init); //Calls the third overload directly, since it requires no conversion.

void funcA(int i, float j);
void funcA(int i, std::string str);
funcA(std::default_init, 5.3f); //Calls first overload with a default-initialized `i`.
funcA(std::default_init, "5.3"); //Calls second overload with a default-initialized `i`.

void funcB(std::default_init, float j);
void funcB(int i, std::string str);
funcB(std::default_init, "foo"); //Calls the second one with a default initialized `i`, as the only viable overload.
`````

## List Initialization

The change above notably does not affect list initialization; [dcl.init]/17.1 sends all braced-init-list-initialized objects to [dcl.init.list] for resolution. And this determination happens well in advance of our new [dcl.init]17 statement.

This is *deliberate*. Consider the following:

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

Given this precedent, we resolve the ambiguity with `default_init_t` in the same way: `{std::default_init}` treats the type as applying to the first member of the aggregate, not the aggregate itself.

Of course, as a *consequence* of this rule, if the type being initialized is not an 	aggregate, using `{std::default_init}` will do something unusual. It will attempt to select constructors based on overload resolution. Given the above rules for overload resolution of `std::default_init_t`, if there is more than one constructor which can be called with one parameter, then the program is ill-formed due to overload resolution conflict.

The one exception is if the first parameter of one of the constructors is of the type `std::default_init_t` itself. In which case, it will be called correctly.

We could of course change list initialization rules to check for initialization from a single `default_init_t` value, but only after passing over to aggregate initialization. However, this could be considered quite incongruous. The proposed way to do this follows a particular interpretation of what list initialization means.

List initialization conceptually means to initialize an object from a potentially-heterogeneous sequence of values. Calling a constructor is simply a way to provide those values to an object's initialization. This helps explain list-initialization's preference for `initializer_list` constructors; it isn't about calling constructors, it is about treating the object as a sequence of values.

Given that understanding of list-initialization, it makes sense to for `{std::default_init}` to default initialize the first member rather than the object itself. Which means default initializing the first parameter of a constructor.

## Library Changes

The language itself takes care of `allocator` and `allocator_traits` issues. Calling `allocator_traits<...>::construct(std::default_init_t)` will perform default initialization automatically. So all of the `emplace`-style APIs are automatically functional with `std::default_init`. So there are fewer forward-facing library APIs that need explicit overloads. Further, any user-created `emplace`-style APIs will gain the benefit of this feature without needed changes.

Containers (including `basic_string`) should be changed to have specific overloads for `default_init_t`-based initialization. Specifically, the following should have an overload that takes `default_init_t`:

* Single-element `insert` functions for sequence containers & `basic_string`. Default initialization doesn't make as much sense for associative and unordered containers.
* All sized-with-`T` functions, including:
	* Constructors
	* `insert`
	* `assign`
	* `resize`

Note that we do not need overloads for `push_x` functions, as they all have `emplace_x` alternatives that work just fine with `std::default_init`. The push_x functions all copy or move a value; what the user wants when they use `std::default_init` is to have the container default initialize the value, not copy/move a default initialized value.

# Alternative Designs

The most obvious alternative is to make it a library-only fix. Essentially, the feature would be to permit containers to default-initialize their members.

Doing this requires adding a new function to `allocator_traits`: `default_construct`, which performs default initialization. From there, we would need to give the container a number of interfaces that the logic can key off of to perform default-initialization of elements.

Of course, this does nothing for user-defined containers. While they would still need to have their interfaces modified to support the language proposal, the library-only solution requires more work on their part.

Plus, the language feature allows existing types like `optional`, `any`, and `variant` to in-place default construct items without having an explicit interface for it. Which also means that similar user-defined types immediately gain the ability to do the same. As well as `make_shared/unique`, `tuple`, and many other types.

## Alternative Definition

The current definition relies about general compiler magic: the compiler detects a type and does something strange with it. We could define it a different way, by saying that `std::default_init_t` is defined as follows:

`````
struct default_init_t
{
	template<typename T>
	operator T() const;
};
`````

The conversion operator would not be available for `T`s that are not default constructible. And for `T`s with explicit default constructors, the conversion operator will also be explicit. The result of the conversion is a prvalue that is default initialized. Thanks to guaranteed elision, this would directly initialize the eventual object.

Such a definition ought to result in similar overload resolution behavior, as well as list initialization behavior. The primary issue is the case of a struct with a `std::default_init_t`-bearing parameter:

`````
struct DefaultInit
{
    DefaultInit() = default;
    DefaultInit(std::default_init_t);
};

DefaultInit d(std::default_init);
`````

In the proposal, `d` should be default initialized. But if we define `std::default_init_t` through implicit conversion, then we will call the constructor instead of performing default initialization. That is, it becomes possible for `T(std::default_init)` to result in something other than default initialization.

# Impact on the Standard

This adds a new type and a new value to the `std` namespace. It adds a number of rules surrounding object initialization, but only with regard to the new type.

In any case, all such changes should be backwards compatible.

The major impact has to do with overload resolution regarding the default initializing type. I've tried to keep the overload resolution rule simple without having to resort to modifying overload scores or anything. Ideally, this construct will not be so commonly used as to cause a problem when passed to overloads.

# Acknowledgments



