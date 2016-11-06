% Towards More Uniform Initialization
%
%

Brace-bounded lists of values were originally confined to use with aggregate initialization. In the development of C++11, the idea was proposed to extend this to include non-aggregate types through the use of a special `initializer_list` type.

This idea was ultimately extended into what became known as uniform initialization: one could initialize any type via the use of a braced-init-list. Doing so provides a number of advantages, among them is that initialization behavior is fully uniform: it works the same everywhere. Automatic type deduction and the elimination of the most vexing parse were also benefits. In a perfect world, we would always use uniform initialization and never use direct constructor syntax again.

However, despite being called "uniform initialization", it cannot be uniformly used everywhere, due to a critical flaw that prevents its use towards these ends. This proposal intends to correct this flaw, thus allowing it to be used in all possible cases of object construction, while having well-defined results.

# Motivation and Scope

The flaw in uniform initialization is quite simple to see in the standard library. The `std::vector` type has an explicit constructor which takes a single `std::vector::size_type`, which is an integral type. `std::vector` also has a constructor that takes an `initializer_list<T>`.

For most types, these constructors can coexist with uniform initialization quite reasonably. For example:

	std::vector<A> v1{20};
	std::vector<A> v2{A{}};
	std::vector<A> v3{A{}, A{}, A{}};

In this example, `v1` is an array of 20 value-constructed values. `v2` is an array of 1 value-constructed element. `v3` is an array of 3 value-constructed elements.

This is all well and good, right up until we change the type of the `vector`:

	std::vector<int> v1{20};
	std::vector<int> v2{int{}};
	std::vector<int> v3{int{}, int{}, int{}};

This uses exactly the same pattern of code as before, but now it has fundamentally different results. `v2` and `v3` retain their old meaning. But `v1` is now very different. It is an array containing a single element, the number 20. Why?

Because uniform initialization syntax always prefers initializer list constructors if one is available that would fit the given braced-init-list. This is a consequence of uniform initialization syntax using the same syntax as initializer lists: the braced-init-list. And since `std::vector<int>::vector(std::initializer_list<int>)` matches the braced-init-list `{20}`, it will be preferred over `std::vector<int>::vector(std::vector<int>::size_type)`.

This is a problem because there is *no way* to get at the `size_type` constructor via uniform initialization syntax. There is nothing we can employ which will cause the conflicting `initializer_list` constructor to be ignored or to do anything else that would allow us to get at a different set of constructors.

Now, this may seem like a rather unimportant issue. After all, if I know that I have to use constructor initialization with `vector<int>`, then I can simply do that. But consider code that is generic on the `vector`'s type:

	template<typename T>
	std::vector<T> MakeVector()
	{
	  return std::vector<T>{20};
	}

By all rights, this code should always do the same thing: return a `vector` containing 20 value-initialized elements. But it does not. `MakeVector<int>()` will return a `vector` containing exactly one element.

This is of much greater concern when dealing with more intricate templates. Consider the following trivial template function:

	template<typename T>
	T Process()
	{
	  T t{};
	  //do stuff with t;
	  return T{t};
	}

This function creates a temporary, does some processing, and returns a copy of it. This function `Process` should have a concept requirement that `T` be DefaultConstructible and CopyConstructible, in addition to whatever "do stuff with `t`" requires.

The problem is the last line. This line may violate the concept constraints of `Process`. Why? Because there is no guarantee that `T` does not have an `initializer_list` constructor that can match with a `T`. For an example of such a class, consider `std::vector<std::function<void()>>` as our `T`.

The problem is that `std::function` has a non-explicit constructor that can take any type that is CopyConstructible. And `std::vector<std::function>` is CopyConstructible. Therefore, this will call the `initializer_list` constructor. But the template function should not be calling the initializer list constructor; it's not part of the allowed interface to `T`. Therefore, `Process` violates the concept constraint that it imposed on `T`, through no fault on the user's part.

What this means is that you cannot use uniform initialization in generic code where the exact type of the object being constructed is derived from a template parameter. This is because there is no way for the user to explicitly choose which constructors to call.

# Investigating the Flaw

Normally in C++, if the attempt to deduce the parameters for a function call would result in two or more equally-preferred functions, the compiler would complain. This would force the user to use an explicit conversion in some part of the function call, thus allowing them to explicitly resolve the ambiguity.

The flaw in C++11's uniform initialization is based on such an ambiguity. This flaw comes up because C++ forces a specific resolution of the ambiguity, rather than giving the user the opportunity to resolve the ambiguity with some specific syntax. The ambiguity is quite simple: For some type `T`, what does this mean?

	T t{5};

If `T` is an aggregate, the rules say that `t` will be aggregate initialized. If `T` is a non-aggregate type, then a constructor of some kind will be called. However, in order to allow container types like `std::vector` to take sequences of uniform values, the C++11 rules do something very strange. They allow the braced-init-list to match, not only constructor signatures that take the given values, but also to match against constructors that take a single non-default value of `initializer_list` type. If the braced-init-list can be used to build an `initializer_list` that matches an initializer-list constructor, then that constructor will be called.

The problem is that this now creates ambiguity between the list of constructors who take the braced-init-list's values as parameters, and the list of initializer-list constructors who take the braced-init-list as a single parameter.

C++ normally would resolve such ambiguity with a compiler error. In function overload resolution, this would force the user to explicitly resolve the call, using some syntax. In particular, the tool of choice is an explicit cast or conversion of some kind. However, in the attempt to create completely uniform syntax, it was decided to use an ad-hoc rule to resolve the ambiguity. Thus, initializer-list constructors are preferred over non-initializer-list constructors.

This means that there are constructors that can be hidden entirely from uniform initialization syntax. And therefore, the when the user writes `T t{5};` in some template code, they can never be certain of what this will actually do. This makes uniform initialization extremely unreliable.

It also creates a serious maintenance problem. Consider a library that exposes some type that users are able to use. Users may be using uniform initialization syntax to construct these types. Now, what if the maintainer of the library wants to add an initializer-list constructor? Suddenly, perfectly functioning code is now broken.

This maintenance problem is faced by any function you might add overloads to, but there are two differences. First, the breakage in the function overloading case is noisy; the code that became ambiguous is ill-formed. While this is annoying for users who's code used to work, at least it's not silently doing the wrong thing. With the uniform initialization case, the code silently starts doing something highly unexpected. But the bigger problem is that the function case can be easily resolved via appropriate explicit casts and conversion.

That is extremely important, because the only way to resolve the uniform initialization ambiguity currently is to stop using uniform initialization syntax. This answer is functional on some level, but it does lead to problems.

For example, there is a reported defect in C++11, about the use of `emplace` functions on containers (effectively, anything that calls `std::allocator_trait::construct`). Namely, that you can't call `emplace` functions on a container of aggregates, because aggregates by definition do not have constructors (besides the default and copy/move constructors). The proposed resolution of this is to have the standard `std::allocator_traits::construct` method detect whether the type is an aggregate via the type-traits, and use constructor or aggregate initialization on it as appropriate.

Why doesn't `std::allocator_traits::construct` simply use uniform initialization? Because uniform initialization cannot be trusted to do the right thing. The right thing in this case being to use aggregate initialization for aggregates or take the elements of the braced-init-list as constructor parameters. Hence the name of the function: construct.

The proposed resolution to the defect requires template metaprogramming tricks involving `std::enable_if`-bound function calls. While C++ versions will eventually be able to use cleaner techniques like concepts, the fact remains that if uniform initialization cannot handle something like this, there clearly must be something wrong with it. This is after all what uniform initialization is for: initializing aggregates like objects. But you can't do that if you want to avoid initializer-list constructors.

The fundamental problem arises due to a conceptual error in uniform initialization: the lack of the user's intent. When a user writes `T t{5};`, they generally have some expectation about what this code is supposed to do. They have some concept of what operations they expect to be legal on `T`, and they therefore expect something specific to happen when it is called.

When initializing a value, there are two conceptual ways to want to initialize it. One could construct an object by providing a series of parameters who's values may or may not be of uniform type. These parameters will be used in some way in constructing that object's data, but there would not be a general expectation that the parameter's values would be directly accessible in the resulting object. Alternatively, the user could provide a list of values of uniform type, usually under the assumption that such a list will be stored in the object or otherwise reflected in that object's contents.

Some types can only be constructed in one way or the other; a `std::unique_ptr` can only be constructed from a series of parameters. A `std::array` can only be constructed from a sequence of values that specify its contents. However, some types can be constructed from both in different circumstances; `std::vector` could be built from parameters, but it could also be built using a series of values that it will be initialized with. Aggregates are a special case, as one can conceptually treat an aggregate as either a constructed object or a container of values, depending on how you think about it. Or sometimes both at once.

Uniform initialization syntax does not offer the user the ability to specify this intent when initializing an object. It intermingles the two. Other initialization syntaxes do let you specify which; constructor syntax is the exclusive domain of a parameter sequence, but it cannot be used with aggregates.

This is not a theoretical problem with the syntax. It is an actual, real problem which people face. [Advice is being offered based on this issue, suggesting that people avoid it.](http://probablydance.com/2013/02/02/the-problems-with-uniform-initialization/) Others suggest freely using uniform initialization, without any warning about these problems. So we have one set of users who are being told to avoid it where possible, while another set are being told to always use it.

That is not a good situation.

# Design

The problem is that the user lacks the ability to specify which kind of initialization they want. The solution... is to give the user the ability to specify which kind of initialization they want.

So we will provide the ability to qualify a braced-init-list. A braced-init-list can be qualified in this way:

	{<special identifier>: ...}

The "special identifier" is an identifier that defines how the braced-init-list is qualified. The available qualifiers are:

* `c`: The braced-init-list is "constructor-qualified".
* `l`: The braced-init-list is "list-qualified".

Note that these are special identifiers, in accord with [lex.name]; they are not keywords.

[over.match.list] specifies how a braced-init-list selects constructors from a class. It has a list of two bullet points, which specify two groups of functions it checks.

A braced-init-list which is "constructor-qualified" will skip the first bullet point. A "list-qualified" braced-init-list skips the second bullet point.

Note that the qualification only affects the behavior of types with constructors. Aggregates, and even default constructor selection (from `{l:}` or {c:}`) proceeds exactly as normal. Conceptually, the idea is that an aggregate could be considered both a value list and a live object with a convenient constructor. And therefore, it can be initialized in either way.

This also means that `{l:}` can still call the default constructor, even though it's conceptually for `initializer_list` initialization. This does represent a bit of an incongruity, but it is not a surprising one, since `{}` will also call the default constructor even if an `initializer_list` constructor exists.

With constructor-qualified lists, we can also allow all standard library `emplace`-style functions (including `make_tuple/unique/shared` and other things that don't rely on `allocator_traits::construct`) to work with aggregates.

## Additional Utility

Since this syntax provides qualifications for braced-init-lists, we also have the ability to avoid some of the issues of list initialization.

There is a distinction between copy-list-initialization and direct-list-initialization. The former is always used in cases where a braced-init-list is deducing the type being initialized. The only difference is that copy-list-initialization is il-formed if it selects an `explicit` qualified constructor.

Even so, C++ developers have been trying to cut down on typename repetition since C++11 gave us `auto`, `decltype`, and similar constructs like return type deduction in C++14. But even the latter does not stop us from putting return types in signatures. So in many cases, the typename already exists in the signature.

The presence of `explicit`-qualified constructors makes `return {...};` problematic, since many constructors really are special. Therefore, we could add an additional special identifier to qualified braced-init-lists:

* `e`: The braced-init-list is "explicit-qualified".

An explicit-qualified list uses direct-list-initialization, regardless of the context it is in.

This qualifier should be able to be combined with `c` or `l`. Therefore, `return {ce: ...};` should be both explicit-qualified and constructor-qualified. Though most initializer_list constructors are not marked `explicit`, so it shouldn't be needed much with list qualification.

## Compatibility with Designated Initializers

These should not interfere with designated initializer syntax. Since at present, such syntax can only be used to initialize an aggregate, it is legal to qualify any such braced-init-list with `c` or `l`. Though it is not very useful to do so.

# Alternative Solutions

## Library Solutions

Originally, I considered a library-based solution to the issue. This would have required adding a special opaque type taken as a parameter to various calls. Since the opaque type is different from other types, it would prevent the braced-init-list from binding to an `initializer_list` constructor, thus disambiguating the call.

This is problematic for two reasons. First, it adds some complexity to various container classes, as they now have to prevent the user from using the opaque type as the member or allocator types (to prevent constructor overload resolution problems). But the primary reason that this is not a good solution is because it only solves the problem for the standard library. Every library user would have to concoct a solution, hopefully one using the same opaque types the standard library uses.

We should not ask every user who writes an `initializer_list` constructor to go though and add a suffix parameter to various functions that could interfere. While it is true that overload resolution is something the developers have to think off when adding new overloads, they are still able to add different constructors so long as the user can use some kind of explicit conversion to resolve the ambiguity. But no such syntax exists for uniform initialization.

This is a problem introduced by the language; therefore, it is best solved by the language. `std::enable_if`, and all of the complexity around it, is what we get when we try to solve language problems (lack of concept checking/`if constexpr`) with library solutions.


