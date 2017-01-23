% Towards a Fix for LWG 2089
%
% 

[LWG issue 2089][1] outlines a problem with using `emplace`-style construction of objects (such object construction shall heretofore be referred to as "indirect initialization"). The user passes such functions a list of values, and the function forwards these values to the constructor of a particular type.

The problem is that aggregates, by definition, do not have a constructor. Therefore, indirect initialization functions cannot invoke aggregate initialization on them. However, we do have list-initialization, which can call constructors or perform aggregate initialization, as is appropriate to the type being constructed.

Unfortunately, we cannot simply declare that indirect initialization functions will always use list-initialization. This would not be backwards compatible with existing code. Even if that were not true, it would hide access to certain constructors.

This paper starts with examining a potential problem in the current solution, along with a fix for it. Then it evaluates ways to resolve this problem through the language and the library. No particular solution is actively being suggested here; this is primarily to example the scope of such construct.

# Flaws in the Proposed Solution

LWG 2089 outlines a proposed library-only solution. `allocator::construct` would perform as follows:

> Effects: `::new((void *)p) U(std::forward<Args>(args)...)` if `is_constructible<U, Args...>::value` is `true`, else `::new((void *)p) U{std::forward<Args>(args)...}`

Obviously, such a solution should be applied equally to such functions which do not rely on `allocator::construct` (`make_unique/shared`, in-place object constructors, and the like). But this has a significant flaw beyond that.

`is_constructible` is true if the type can be constructed with the given argument types. This seems like the right question, but it may not be the question the user actually expects.

Under the current standard, indirect initialization calls will fail if the type is not constructible with the given arguments. This could be for two reasons: 1) the type is an aggregate, or 2) the type doesn't have an appropriate constructor for the parameter list. Consider the following type:

	class Something
	{
	public:
		Something(int);
		Something(std::initializer_list<int>);
	};

If the user passes a single integer to `vector<Something>::emplace_back`, then the first constructor will be called. If the user passes a pair of integers, then (under the current standard) a compile error will result, since there is no matching constructor.

However, with the above fix, `is_constructible` will return `false`. Therefore, `emplace_back` will use list-initialization with the parameters. And since they match up against an `initializer_list` constructor, that will be called.

The question is this: is that what the user wanted or expected to happen?

This behavior is very incongruous with current list-initialization rules, which prefer `initializer_list` constructors over regular ones. If you did `Something{10}`, this would call the `initializer_list` constructor. Whereas calling `emplace(10)` will call the regular constructor. And yet, `Something{10, 20}` will have the same effect as `emplace(10, 20)`: calling the `initializer_list` constructor.

I suspect that users would find this to be surprising. A user would probably expect indirect initialization for a non-aggregate to either act exactly like list-initialization or exactly like constructor initialization, with list-initialization used only for actual aggregates. They would not expect it to have behavior that is distinct from both of the existing language-based cases.

At the same time, the following is not an adequate solution either:

> Effects: `::new((void *)p) U(std::forward<Args>(args)...)` if `is_aggregate<U>::value` is `false`, else `::new((void *)p) U{std::forward<Args>(args)...}`

Ignoring that `is_aggregate` is not an actual type trait, the problem here is dealing with this unusual circumstance<a name="agg-problem"/>:

	struct Agg
	{
		int i;
	};

	class Value
	{
	public:
		operator Agg() const {...}
	};

	Agg a1(Value{});
	Agg a2{Value{}};

The important fact here is that, while aggregates do not have user-provided constructors, users can still call default constructors and copy/move constructors. As such, the initialization of `a1` works, because `Value` is implicitly convertible to `Agg`, and thus it can invoke the copy constructor.

The initialization of `a2` does *not* work, because it attempts to use aggregate initialization. However, the `is_aggregate` version of indirect initialization will *always* use aggregate initialization on aggregates, regardless of what types the user passes. Thus, the `is_aggregate`-based fix represents a backwards-incompatible change, because right now `vector<Agg>::emplace(Value{})` is perfectly valid code.

<a name="proposed"/>Therefore, a complete fix which is backwards-compatible while making sure not to invoke `initializer_list` constructors improperly would be the following:

> Effects: `::new((void *)p) U(std::forward<Args>(args)...)` if `is_constructible<U, Args...>::value` is `true`, else `::new((void *)p) U{std::forward<Args>(args)...}` if `is_aggregate<U>::value` is true, else ill-formed.

This version will only use list-initialization on actual aggregates, and only when constructor initialization is not possible. It therefore represents a more targeted fix.

## Unfixable Flaws { #unfixable }

The fundamental problem which is the basis of LWG 2089 is this: we want to be able to initialize objects of some known type `T`, given *only* an ordered sequence of values. C++ has many forms of initialization from a sequence of values, each with its own syntax. But in our case, the selection of syntax has to be made for us; the only thing we can control is exactly what types/values are provided.

The goal of all of these indirect initialization fixes is to find a way of initializing objects that will allow you to initialize them in any way that you could have directly. But without a way to explicitly specify a means of initialization, all we have to work with is the sequence of initialization parameters.

Our solution above is quite a good solution. It provides a way to access most of the ways of initializing a type. You can pass initializer_list objects directly if you want to access initializer_list constructors. You can pass constructor parameters that will be used exactly as such. And if those don't match, and the type is an aggregate, then it will attempt aggregate initialization with the parameters.

This solution may be more verbose in some cases, but it has fewer corner cases than list-initialization. Even so, it still has some corner cases that cannot be easily resolved. Consider this:

	struct OuterAgg;

	struct InnerAgg
	{
		int i;
		operator OuterAgg() const;
	};

	struct OuterAgg
	{
		InnerAgg agg;
		float f = 5.0f;
	};

	InnerAgg::operator OuterAgg() const { return {{i}, 2.0f}; }

	OuterAgg a1(InnerAgg{});
	OuterAgg a2{InnerAgg{}};

`a1` is being initialized via constructor syntax. As such, it will attempt to call one of the special member constructors of the type. Since `InnerAgg{}` is implicitly convertible to `OuterAgg`, `a1` will be initialized by a call to the copy constructor. As such, `a1.f` will be 2.0f.

`a2` is being initialized via list-initialization. `InnerAgg` is not related to `OuterAgg`, so clause [dcl.init.list]/3.1 (invoking copy/move) is skipped. So clause 3.3 kicks in: invoke aggregate initialization. `InnerAgg{}` can initialize `OuterAgg::agg`, so it will do so. The rest will get their default values. So `a2.f` will be 5.0f.

Given the proposed solution of construct-then-aggregate-initialize, `emplace(InnerAgg{})` will always produce `a1`, since `InnerAgg` is implicitly convertible to `OuterAgg`. Therefore, the only way to construct things in `a2`'s style is via `emplace(OuterAgg{InnerAgg{}})`.

Unfortunately, there is no way to *avoid* invoking a copy/move constructor through indirect initialization of a single parameter of the type to be initialized. Whereas `emplace(InnerAgg{})` will not need to copy/move to do its initialization.

There is no single syntax that can possibly allow both kinds of initialization through just `emplace(InnerAgg{})`. This is not a real criticism of the resolution, so much as a recognition that a perfect solution is fundamentally impossible, given only parameters to be passed to the constructor/braced-init-list. There will always be some corner case that has a sub-optimal solution.

So long as such cases are a rarity, the syntax should be adequate.

<a name="disuniform"/>There is also the issue of non-uniformity in initialization. In order to use these initialization forms, you really need to know *what* you're initializing.

For example, using `emplace` for an aggregate type like `std::array` will not look like `emplace` for a user-defined type like `std::vector`. While both would respond to uniform initialization syntax in the same way, `emplace` initialization cannot do so:

	vector<array<int, 2>> va;
	vector<vector<int>> vv;
	va.emplace_back(1, 2); //Initializes the aggregate.
	vv.emplace_back(initializer_list<int>{1, 2}); //Calls an initializer list constructor.
	
The syntax for `va` cannot be used with `vv`, and vice-versa.

LWG 2089, as a solution, cannot resolve this, but some language solutions can.

## A New Form of Initialization

It is vital to understand that, by implementing *any* fix for LWG 2089, we will be introducing a *new form* of initialization into C++. This form is a means of initializing objects which uses a unique set of initialization rules, reliant on other forms yet the collection is distinct from them. And we will be standardizing this initialization form, employing it throughout the standard library.

By applying this new initialization form to all indirect initialization APIs, users will need to be aware of its behavior and quirks. As Ville Voutilainen outlined in [N4462][ville], it is very important for users to be able to not merely understand this form of initialization, but to write it into their own APIs. When a user goes to implement their own indirect initialization APIs, we want them to have the same behavior.

As such, visibility is an important aspect of a good resolution to LWG 2089. It is very easy to write an indirect initialization function. As such, user code is likely to create these in a variety of places. If users are unaware of the LWG 2089 fix, or are ignorant of even the need for it, then users are very likely to simply use constructor or list-initialization syntax, rather than the LWG 2089 fix. However, if they use the LWG 2089 fix throughout their code, even for non-indirect initialization cases, then they will simply naturally default to it.

## Library vs. Language { #lang_library }

This paper categorizes solutions into two groups: [library changes](#library) and [language changes](#language). These two groups have some benefits and downsides in common.

The prime benefit of library changes are that they are not language changes, and thus not *formally* ushering in a new kind of initialization. C++ is replete with initialization forms, so not adding another to the language can be seen as a benefit.

Library-based solutions generally have lower user visibility, particularly to less experienced C++ developers. There are so many utility functions, algorithms, and classes that lots of C++ programmers simply do not know all that their standard library offers them. Whereas the number of language features added to a new version are much smaller.

Library solutions to this problem in particular will have lower visibility than usual, since it will primarily be used inside the function performing object initialization. They are just one more utility function in a library with many such functions. Users are unlikely to wholesale switch to using `std::initialize<T>(...)` over directly invoking `T(...)` or `T{...}`, even within template code. It's just too cumbersome for day-to-day use.

The lack of visibility of these solutions makes it unlikely that novice C++ users will know to use `std::initialize` in their own indirect initialization functions. This will create a dichotomy between the behavior users get from the standard library and what they get from user-written libraries. If users have never seen `std::initialize` before, they will likely attempt to roll their own equivalent, with mixed results.

# Library-Based Solutions { #library }

## LWG 2089 Through the Library { #library_2089 }

C++17's features, particularly `if constexpr` makes the code for implementing this initialization form much easier to write. The code for the [resolution suggested here](#proposed) would look something like this:

	template<typename T, typename ...Args>
	auto initialize(Args &&...args)
	{
		if constexpr(is_constructible_v<T, Args...>)
			return T(std::forward<Args>(args)...);
		else if constexpr(is_aggregate_v<T>)
			return T{std::forward<Args>(args)...};
		else
			static_assert(false, "Cannot initialize with parameters");
		end
	}

Obviously, this requires the addition of an `is_aggreate` trait.

The standard library could simply provide this function as a common indirect initialization utility tool. LWG 2089 would be resolved by having `std::initialize<T>` be used by:

* `allocator_traits<Alloc>::construct`, via `new(static_cast<void*>(p)) auto(std::initialize<T>(std::forward<Args>(args)...))`, made possible thanks to guaranteed elision.
* The now-deprecated `allocator::construct`, through similar syntax.
* `make_shared` and `make_unique`
* The `in_place_t` constructors for `any`, `variant` and `optional`.
* Any other APIs that use indirect construction syntax.

### Pros and Cons { #library_2089_pro_con }

The principle advantage of this library-only feature is that it is self-contained. We would be introducing a new initialization form, but it would be one that lives.

One downside of this solution is narrowing. When performing list initialization directly, you can provide an integer or floating-point literal, even to parameters that are smaller than the type that the literal would otherwise deduce to. So if you pass a literal "1" to a `char`, that is fine, because the compiler can detect the constant expression and see that "1" is in the range of char.

However, once you are dealing with indirect initialization, that becomes a problem. The literal is long-gone; it is replaced by an `int`. And `int`-to-`char` conversion is narrowing. Since the LWG 2089 syntax is based on list-initialization, which forbids narrowing conversions, what would have been valid code becomes a compile error.

This feature also poorly interacts with other features that make initialization easier. The most obvious of which is template argument deduction for constructors. A user can write `vector(...)` easily enough; they can even use list-initialization if they so desire. But they cannot use `initialize<vector>(...)` so easily; they have to explicitly name the class.

In cases of indirect initialization, there is pretty much no way to avoid it. And in many cases, the typename is implicit, so there is no need. In cases like `in_place_type_t<T>`, there is little that can be done to avoid having to spell out the full typename.

## Initialization Control Through Parameters { #library_control }

Recall that the problem we have is that C++ has many forms of initialization, but we as the indirect initializer can only provide a sequence of parameters to whatever initialization form the code chooses to use.

LWG 2089 tries to solve this problem by declaring that, if the sequence of parameters for one form of initialization is invalid, then it tries another form. This gives priority to one form of initialization, but leads to a [number of unfixable problems](#unfixable). LWG 2089 essentially hopes that these are infrequently encountered corner cases and/or that users can work around them when they are encountered. While better than any other solution, LWG 2089 is ultimately not a perfect form of initialization forwarding.

However, the caller of an indirect initialization function does have one useful ability: the ability to add more parameters. As such, we could establish a set of special types, declared in the `std` namespace, which directly control the form of initialization to be used.

We could declare `std::construct_init_t` and `std::list_init_t`, which respectively mean to use constructor sytnax and list syntax. We would also have values for them, `std::construct_init` and `std::list_init`.

Our generic `std::initialize` function would look like this:

`````
template<typename T, typename ...Args>
T initialize(Args ...args)
{
	return T(std::forward<Args>(args)...);
}

template<typename T, typename ...Args>
T initialize(std::construct_init_t, Args ...args)
{
	return T(std::forward<Args>(args)...);
}

template<typename T, typename ...Args>
T initialize(std::list_init_t, Args ...args)
{
	return T{std::forward<Args>(args)...};
}
`````

If a user wants to use constructor initialization, they would pass `std::construct_init` as the first parameter to any function that uses `initialize`.

We could even apply LWG 2089's solution to the first overload of `initialize`, in addition to the qualified forms. That would permit users to use shorter syntax in many cases where the user's desires are obvious, but with the ability to qualify the initialization where needed.

### Pros and Cons { #library_control_pro_cons }

There are two advantages. The first is that this syntax visibly captures the user's intent, allowing users to directly spell out exactly what they want at the time they call the indirect function. This is generally preferable to relying on esoteric initialization rules.

The second advantage is that it fixes all of those [supposedly unfixable](#unfixable) problems. Or more to the point, it gives users the ability to resolve those conflicts themselves.

The typical visibility issues of a library solution are present. However, in this case, they are slightly mitigated. If a user expects you to have used this version of std::initialize and you have not, then the problems will be apparent when someone tries to use the control parameters and fails.

## Injection Initialization { #library_injection }

The fundamental issue behind this entire problem is that the user is forced to cede control over how initialization works to external, often generic, code. The previous solution dealt with this problem by allowing users to specify the form of initialization. This solution works by handing over initialization to the use *entirely*.

Guaranteed elision allows prvalues to be used to directly initialize any object. As such, syntax of the form `new(ptr) Typename(Function())` can defer the initialization of the object to `Function`. And so long as function returns a prvalue, no extraneous copies/moves are provoked.

Indirect initialization functions can be created to take advantage of this. Whether these are new functions or the same ones with new type-based overloads are issues to be determined. The general idea is to have functions that take a callable of some kind, possibly with a set of parameters to pass to it. The indirect object would be initialized as if by:

````
new(memory) Type(Function(std::forward<Args>(args)...));
````

This should probably bypass `allocator_traits` entirely. Such an interface would permit users to use aggregate initialization like so:

````
vector<Aggregate> v;
v.inject_back([]{return Aggregate{1, 4, "a string"};});
````

### Pros and Cons { #library_injection_pro_con }

This solves most indirect initialization problems, since it gives the user complete and direct control over object initialization. It does a better job than allocator_traits at allowing you to control access to constructor (public vs. private). And thanks to guaranteed elision, this is quite performance friendly, and it can even work with placement new.

One issue that this problem doesn't solve is dealing with template code that doesn't know or care if a type is an aggregate or non-aggregate. If the code wants to call a constructor or use list initialization if the constructor is not available, they must manually develop such an initialization function.

Another issue with injection is that it makes initialization more verbose. Consider the following:

````
vector<Aggregate> v;
v.inject_back([]{return Aggregate{1, 4, "a string"};});
v.emplace_back(1, 4, "a string");
````

Over half of the text in the injection version is dedicated to extraneous text. If you have a `vector<T>`, then you know what type is going to be created. Yet the lambda must repeat the typename, either in the return expression or as the lambda's return type. The lambda prefix and so forth only adds to the verbosity.

Even in the ideal case of expression lambdas (a feature we don't have yet, mind you), this is the best we could do:

````
v.inject_back([] => Aggregate{1, 4, "a string"});
v.emplace_back(1, 4, "a string");
````

# Language-Based Solutions { #language }

## Control Parameters in The Language { #language_control }

The use of control parameters has some good aspects. But it has all of the problems of any library-only solution: we are creating a new form of initialization via a set of rules bound up in the library. That makes it very easy for users to be unaware of this kind of initialization. And it is very important that users can rely on these parameters in indirect indirect initialization scenarios.

However, because these control parameters are based on values and types that do not yet exist in the standard, we could modify the language's initialization rules to use these types to control the form of initialization at the language level.

For direct or copy initialization, we can change the rules to ignore the first parameter in the initializer if that type is `std::construct_init_t`. If the first parameter is of type `std::list_init_t`, then it uses the rest of the parameters in list-initialization, as though they were in a braced-init-list.

For list-initialization, we do something similar. If the first element of the braced-init-list is `construct_init_t`, then the rest of the parameters are forwarded to direct or copy initialization. If the first element is `list_init_t`, then the rest of the parameters are forwarded back through list-initialization.

This syntax also makes it possible to deal with the indirect narrowing issue. While `T(std::list_init_t, ...)` could use the exact same list-initialization rules, we could also make it so that if the original initialization form was construction, then the forbearance of narrowing no longer applies.

Nothing in the standard library would have to change to make this work.

### Pros and Cons { #language_control_pro_con }

Apart from [all of the advantages of the library version](#library_control_pro_cons), a major side benefit is that we can access all of the advantages of list-initialization (use in brace-or-equals initializers, automatic deduction of the type being initialized, etc), but *without* most of the [issues of its sometimes-oddball initialization rules][ui problem]. So the old `vector<int>` problem disappears:

	vector<int> func() {return {std::construct_init, 5, 10};}

This will always create a 5-element vector, with each element storing 10.

How visible this feature will be is somewhat questionable, due to factors discussed a bit later. But because it is built into all forms of initialization, visibility is irrelevant. Users who need to write an indirect initialization function cannot help but allow this syntax to work.

One really important downside here is that this creates lots of complexity in the initialization rules, which are not known for being simple. What happens if the first two parameters to the initializer are `construct_init_t` and `list_init_t`? Should this mean list-initialization of the parameters that aren't of those types? Or does it mean constructor initialization, with the first argument to the constructor being of type `list_init_t`?

Also, how do these types behave on initialization? With a library-only solution, they behaved exactly like ordinary types. Once these types are part of general initialization, they start doing odd things:

	auto x = std::construct_init;

What this ought to do is copy from the value into a new variable of its type. However, the first parameter to the initializer is of type `std::construct_init_t`, which means that we should ignore it, right? Which means that it should perform default or value initialization.

But for what type? What type do we deduce `x` to be? If the first parameter is ignored, then clearly it shouldn't be part of that deduction.

These are hardly insurmountable problems. But they require careful thought and added complexity in an already complex part of the standard.

Another disadvantage is that it is explicit about which form of initialization to use. While that is usually a good thing, the downside is that it is impossible to treat constructor and aggregate initialization the same way. The code passing the indirect parameters *must* be aware whether the type should use constructor or list initialization.

This poses a problem with template code. Or, more to the point, it does not allow a particular degree of interface freedom. If template code were restricted by a concept to [use `std::initialize<T>`](#library_2089) for a particular set of parameters, the user would be free to pass aggregates or objects that fit that particular interface. By forcing the code to statically select `construct_init` or `list_init`, things become much more complex.

This problem could be solved by having a third form: `construct_list_init`, which works like our proposed LWG 2089 fix. The template code could then rely on that particular set of functionality.

A more minor problem is the added verbosity. A language solution, by virtue of being part of C++'s syntax, has the potential to become widely used outside of indirect initialization scenarios. But this feature is unlikely to cause that, thanks to its verbosity. A user might want to uniformly use uniform initialization syntax, but the sheer bulk of having to use `std::construct_init` to avoid calling `initializer_list` constructors will ward such users away.

It will thus likely only see use to disambiguate corner cases, not be used day-to-day.

## LWG 2089 by Constructor Extension { #language_2089_constructor }

As it currently stands, constructor syntax cannot invoke aggregate initialization. We could change that. If we do, then the [desired LWG 2089 fix rules](#proposed) naturally fall out. Constructor syntax will use the existing rules, and any constructor calls that would be ill-formed when applied to an aggregate will instead invoke aggregate initialization. But it will only do so if the type is an aggregate.

This also gives us a good chance to deal with the indirect narrowing issue. Aggregate initialization provoked through constructor syntax could permit narrowing conversions.

### Pros and Cons { #language_2089_constructor }

New initialization forms as language features have the problem of cramped syntax. Since this one overloads existing syntax, we deftly avoid that problem. Plus, since it is a syntax people commonly use, there is very little to learn. And thus, virtually all user-written indirect initialization functions will behave as the user expects them to.

A more difficult issue has to do with default member initializers:

	struct Outer
	{
		Inner i(stuff);
	};

To avoid parsing ambiguity issues with member functions, a brace-or-equal-initializer cannot directly use constructor syntax. This requires users to do this instead:

	struct Outer
	{
		Inner i = Inner(stuff);
	};

This needlessly repeats the typename (though it does not require a copy/move constructor). This could be alleviated by permitting `auto` to be used for members which have a equals-style brace-or-equal-initializer.

## LWG 2089 List-Initialization Extensions { #language_2089_list }

List-initialization has a lot of really useful properties, syntactically speaking. Copy-list-initialization can deduce the type to be constructed based on the context in which it is used, thus [promoting DRY principles](https://en.wikipedia.org/wiki/Don't_repeat_yourself). It is already hooked into existing syntactic mechanisms like brace-or-equal-initializers.

The main problem with list-initialization is not how it is invoked on types; it is how it behaves. It does not behave in a backwards-compatible fashion with constructor syntax, so it cannot be used to resolve LWG 2089.

Even so, it would be quite ideal if we had a form of initialization which keys off of list-initialization *syntax*, but the actual initialization behavior is in accord with our rules here.

A possible syntax for this would be as follows:

	{c: stuff};

This would be called a "c-qualified braced-init-list". Regular braced-init-lists are "unqualified braced-init-lists". Applying a c-qualified braced-init-list to a type will invoke "c-qualified list-initialization".

Note that syntactically, these work identically to a braced-init-list, and this kind of initialization is still list-initialization. As such, `T t = {c: stuff};` is still copy-list-initialization rather than direct-list-initialization, so explicit constructors (if any) will not be allowed.

Also, note that the `c` here is not a keyword. It is a "special identifier", under the rules for such things. As such, there should be no need to reserve it as a keyword, and it should not interfere in parsing existing braced-init-lists.

C-qualified list-initialization must behave exactly [as we proposed above](#proposed). As such, its rules are *completely separate* from the list-initialization rules in [dcl.init.list]/3. Those rules become the rules for unqualified list-initialization.

In order to properly implement LWG 2089 in a backwards-compatible way, narrowing conversions on c-qualified braced-init-list members would be allowed. We could choose only to allow them when not performing aggregate initialization, but it would be simpler to just allow them period.

Qualified braced-init-lists should not be able to use designated initializers. This makes sense, as the purpose of this initialization syntax is not to solely initialize aggregates. If you want designated initializers, then you know you're dealing with an aggregate, so you don't need the qualification syntax at all.

## List Qualification

While the above is sufficient to match LWG 2089 (once applied to all applicable functions), it is useful to explore this syntax to see if other indirect initialization issues can be resolved.

Regardless of how we expose this new form of initialization, users will still have one problem. That problem is how to cleanly provide an `initializer_list` for `initializer_list` constructors to indirect initialization functions.

Consider the following:

    vector<array<int, 4>> va = {{1, 2, 3, 4}, {10, 20, 30, 40}}
    vector<vector<int>> vv = {{1, 2, 3, 4}, {10, 20, 30, 40}};

Both of these are equally short and clean pieces of code that make it clear exactly what is going on. There is no extra redundancy, yet users can easily see how this works. However, using `emplace_back` to insert shows a sharp difference:

	va.emplace_back(1, 2, 3, 4); //Uses aggregate initialization.
	vv.emplace_back(1, 2, 3, 4); //Compile error

Our [suggested resolution to LWG 2089](#proposed) explicitly disallowed the second one, on the grounds that `vv.emplace_back(1, 2)` would have radically different behavior from `vv.emplace_back(1, 2, 3)`. As such, the natural solution is to have the user explicitly provide an `initializer_list`:

	vv.emplace_back({1, 2, 3, 4});

This does not work, since a bare braced-init-list cannot undergo template argument deduction in this way. As well as other similar restrictions.

It should be noted that the `in_place_t` functions added to `any`, `optional`, and `variant` actually have a solution to this problem. They have a second overload explicitly for `initializer_list<T>`, so long as the target type in question can be constructed from one (and the later provided arguments).

Even so, this is a problem that could better be solved in the language itself. The main problem is that braced-init-lists cannot be deduced into `initializer_list<T>`s by template parameters. And this is a good thing; it is not clear if the user meant to get an `initializer_list<T>` or some other type. So it's best to let the user make this explicit.

The problem is that making it explicit is *really* long-winded:

	vv.emplace_back(std::initializer_list<int>{1, 2, 3, 4}); //Valid code.
	
Our goal is therefore to make this shorter. There are two cases that we want to cover: one where the compiler deduces the `T` in `initializer_list<T>`, and one where the user provides an explicit type to use. Case 1 is equivalent to `auto x = {more-than-one-value};` rules. Case 2 is equivalent to `intializer_list<T> x = {values};`, for a given `T`.

Given that we already have one form of qualified list, we can add another:

	{l: stuff} //Converts to an `initializer_list`, but deduces `T` from the values.
	{l Typename: stuff} //Converts to `initializer_list<Typename>`.
	
This would be called an "l-qualified braced-init-list", which shall invoke "l-qualified list-initialization". The second syntax a "type-qualified list" in addition to be an "l-qualified" list. The first syntax will behave in accord with our existing `auto x = {more-than-one-value};` syntax, though `stuff` is permitted to have exactly one value. The second syntax will behave in accord with `intializer_list<Typename> x = {values};`.

As with `c` above, `l` is a special identifier, not a keyword.

Note that narrowing conversions will be forbidden. Also designated initializers are forbidden.

L-qualified lists can be used in deduced contexts. Exactly what they deduce to will be discussed below. They can also be applied to types with `initializer_list` constructors; they will only attempt to call such constructors.

The question is whether they should be able to be applied to aggregates. It sounds like a very useful idea. Conceptually, these all seem to mean the same thing:

	auto il = {l: 1, 2, 3};
	int arr[] = {l: 1, 2, 3};
	std::array<int, 3> arr2 = {l: 1, 2, 3};
	std::vector<int> v = {l: 1, 2, 3};
	
	struct IVec3 { int x; int y; int z; };
	IVec3 agg = {l: 1, 2, 3};

This code seems to express the intent of the user very effectively, without ambiguity.

The concern is with aggregates which are not arrays. `IVec3` above is an aggregate of 3 variables, but is conceptually array-like in its meaning. It is a 3D vector of integers, not unlike an `array<int, 3>`. So conceptually, it makes sense to be able to use l-qualified list-initialization on it.

Rather than trying to differentiate which aggregates are array-like "enough", it is easier to differentiate between the actual initializer values themselves.

The expressions of an untyped l-qualified list must be able to deduce to some `initializer_list<T>`. The expressions of a typed l-qualified list must fit without narrowing into an `initializer_list<type>`, where `type` is the provided type. Both of these are regardless of whether they are used to initialize an aggregate.

This is the sum total of what we check. As such, this allows unusual or incoherent initialization constructs:

	Agg agg2 = {l float: 1, 2, 3};

The values do not require a narrowing conversion to fit into a `float`. And the narrowing check on the actual aggregate initialization passes as well. Therefore, despite `Agg` being a sequence of integers, this initialization is allowed.

Similarly, this will work:

	std::vector<float> vf = {l int: 1, 2, 3};

So while an l-qualified list will check to see if it can be used to generate an appropriate `initializer_list`, it will not check to see if it is used in initialization that matches the type in question.

### Aggregate Uniformity

In a previous section, [this example](#disuniform) was given to represent a distinction between initializing an aggregate and initializing a non-aggregate with an `initializer_list` constructor. Adjusted for our new syntax, this is the result:

	vector<array<int, 2>> va;
	vector<vector<int>> vv;
	va.emplace_back(1, 2); //Initializes the aggregate.
	vv.emplace_back({l: 1, 2}); //Calls an initializer_list constructor.

This functions well enough. However, it would be nice if there was a way to have a single syntax that could initialize both. We know that making `vv.emplace_back(1, 2)` call the `initializer_list` constructor is a complete non-starter, as it breaks backwards compatibility (this example uses 2 elements specifically to demonstrate this). Therefore, we could entertain the second possibility: somehow make `va.emplace_back({l: 1, 2})` invoke aggregate initialization.

It is generally unworkable to make a runtime-sized type like `initializer_list` directly interface with a compiler construct like an aggregate. The interaction of runtime-sized initialization along with default member initializers in aggregates makes the resulting code rather sub-optimal.

We do have one advantage: `{l: stuff}` is a new piece of syntax. As such, we are free to give it new rules.

So instead of deducing to an `initializer_list<T>`, `{l: stuff}` deduces instead to `std::sized_init_list<T, N>`. This should be a class publicly derived from `initializer_list<T>`. As such, it should be able to be used in such interfaces (calling `initializer_list` constructors). However, functions that takes variadic parameters like `emplace` will deduce a `sized_init_list<T, N>`.

Note that this would affect *all uses* of the syntax: `auto x = {l: stuff}` will deduce a `sized_init_list<T, N>` and so forth.

With such an object in hand, we can *statically* perform aggregate initialization from `sized_init_list<T, N>`. The compiler can statically generate code for any particular `N`, and thus not have to generate code that does runtime checks. If `N` is greater than the number of aggregate members, you get a compile error.

This allows us to permit these:

	Agg agg{l: stuff};
	vector<Agg> va;
	va.emplace_back({l: stuff});

The only difference that `emplace_back` will provoke a copy from the values in the `sized_init_list`, while `agg` will not. But that's a distinction which cannot be avoided, since we're using parameters of a function to remotely initialize an object.

### Pros and Cons { #language_2089_list_pro_con }

This feature is a big change. While it is backwards compatible, adding new syntax rather than taking any away, it is still quite sizable.

It is good to look at a solution of such complexity and ask if it results in something conceptually grounded, or if it's just a hodge-podge of useful stuff slapped into a convenient syntax. Or to put it another way, what does all of this syntax *mean*?

Unqualified list-initialization has a meaning. It treats an object as a collection of heterogeneous values. This is what an aggregate is, after all; we simply extend that initialization to other types. This is part of the reason for prioritizing `initializer_list` constructors; homogeneous sequences of values are a subset of heterogeneous values. Such constructors therefore initialize the type as a collection of values, like a user-defined aggregate. We permit non-`initializer_list` constructors to be called, but they are conceptually treating constructor parameters as equivalent to internal object state.

C-qualified list-initialization means to treat an object's initialization like a constructor call. Constructor parameters are constructor parameters; whether they match internal object state is up to the particular object. Aggregates are permitted, but the user is conceptually treating the aggregate as if it had a constructor that initialized its members from the parameters. That is, the fact that there is a 1:1 mapping between the object members and the parameters to `{c:}` is treated as a fortunate coincidence, a design decision that may later be changed in a backwards-compatible way.

L-qualified list-initialization means to treat an object as a collection of homogeneous values. Not every object can be initialized in such a way. Such lists directly verify that the values are homogeneous in terms of their types.

The principle downside of this fix is that there is a very large difference in initialization behavior between `{stuff}` and `{c: stuff}`. While that is the main point, it does create a few elements of unexpected incongruity. Unqualified list-initialization forbids narrowing in all cases, while c-qualified initialization permits narrowing. Furthermore, `{c: stuff}` will never create an `initializer_list`, even directly:

	initializer_list<int> il{c: 1, 2, 3, 4}; //Ill-formed

`initializer_list` has no constructor that take 4 integer parameters. And none of the other cases apply. So it cannot be constructed in this way.




# Acknowledgments

* Joël Lamotte: For coming up with the idea to use {:} syntax for differentiating forms of list initialization.
* Malte Skarupke: For bringing up the issue of using uniform initialization in generic code.
* 

# References

* LWG issue [#2089][1]
* [N4462][ville]: LWG 2089, Towards more perfect forwarding, Ville Voutilainen
* [The Problems With Uniform Initialization][ui problem], Malte Skarupke

[1]: http://cplusplus.github.io/LWG/lwg-active.html#2089
[ville]: http://wg21.link/N4462
[ui problem]: http://probablydance.com/2013/02/02/the-problems-with-uniform-initialization/
[4]: http://ideone.com/n5q0CW