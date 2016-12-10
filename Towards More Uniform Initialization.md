% Towards Fix LWG 2089
%
% 

[LWG issue 2089][1] outlines a problem with using `emplace`-style construction of objects. The user passes such functions a list of values, and the function forwards these values to the constructor of a particular type.

The problem is that aggregates, by definition, do not have a constructor. Therefore, `emplace`-style functions cannot be used to initialize them. However, we do have so-called "uniform initialization syntax", which can call constructors or perform aggregate initialization, as is appropriate to the type being constructed.

Unfortunately, we cannot simply declare that `emplace`-style functions will always use braced-init-lists. There are actually several reasons for this (which will be discussed below). One of the principle reasons is that such lists have a preference for `initializer_list` constructors where available, and this preference can break existing code that relied on `emplace`-style functions to call a non-`initializer_list` constructor.

This paper evaluates the problem and current proposed solution. It also goes deeper than that, evaluating the fundamental problem that LWG 2089 represents.

# Flaws in the Proposed Solution

LWG 2089 outlines a proposed library-only solution. `allocator::construct` would perform as follows:

> Effects: `::new((void *)p) U(std::forward<Args>(args)...)` if `is_constructible<U, Args...>::value` is `true`, else `::new((void *)p) U{std::forward<Args>(args)...}`

Obviously, such a solution should be applied equally to such functions which do not rely on `allocator::construct` (`make_unique/shared`, in-place object constructors, and the like). But this has a significant flaw beyond that.

`is_constructible` is true if the type can be constructed with the given argument types. This seems like the right question, but it may not be the question the user actually expects.

Under the current standard, the `emplace` call will fail if the type is not constructible with the given arguments. This could be for two reasons: 1) the type is an aggregate, or 2) the type doesn't have an appropriate constructor for the parameter list. Consider the following type:

	class Something
	{
	public:
		Something(int);
		Something(std::initializer_list<int>);
	};

If the user passes a single integer to `emplace`, then the first constructor will be called. If the user passes a pair of integers, then (under the current standard) a compile error will result, since there is no matching constructor.

However, with the above fix, `is_constructible` will return `false`. Therefore, `emplace` will use list-initialization with the parameters. And since they match up against an `initializer_list` constructor, that will be called.

The question is this: is that what the user wanted or expected to happen?

This behavior is very incongruent with current list-initialization rules, which prefer `initializer_list` constructors over regular ones. If you did `Something{10}`, this would call the `initializer_list` constructor. Whereas calling `emplace(10)` will call the regular constructor. And yet, `Something{10, 20}` will have the same effect as `emplace(10, 20)`: calling the `initializer_list` constructor.

I suspect that users would find this to be surprising. A user would probably expect `emplace` for a non-aggregate to either act exactly like list-initialization or act like constructor initialization, with aggregate initialization used only for actual aggregates. They would not expect it to have behavior that is distinct from both language-based cases.

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

The important fact here is that, while aggregates do not have user-provided constructors, users can still call default constructors and copy/move constructors. As such,the initialization of `a1` works, because `Value` is implicitly convertible to `Agg`, and thus it can invoke the copy constructor.

The initialization of `a2` does *not* work, because it attempts to use aggregate initialization. However, the `is_aggregate` version of `emplace` will *always* use aggregate initialization on aggregates, regardless of what types the user passes. Thus, the `is_aggregate`-based fix represents a backwards-incompatible change, because right now `vector<Agg>::emplace(Value{})` is perfectly valid code.

Therefore, a complete fix which is backwards-compatible without invoking `initializer_list` constructors improperly would be the following<a name="proposed"/>:

> Effects: `::new((void *)p) U(std::forward<Args>(args)...)` if `is_constructible<U, Args...>::value` is `true`, else `::new((void *)p) U{std::forward<Args>(args)...}` if `is_aggregate<U>` is true, else il-formed.

This version will only use list-initialization on actual aggregates, and only when constructor initialization is not possible. It therefore represents a more targeted fix.

# The Case for a Language Solution

The above outlines a purely library-based approach. But, as Ville Voutilainen outlined in [N4462][ville], it is difficult for the user to build their own equivalent to `allocator::construct`, one which exhibits this exact behavior. And that was for the LWG 2089 solution; the one outlined here requires even more coding.

Given C++17 features, the code is not so difficult to write:

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

The standard library could provide this template function, so that users do not have to write this every time they want to do `emplace`-style construction on their own.

One of the problems with this is that users will have to be *taught* to use this function. By default, users will do something like this:

	template<typename ...Args>
	auto emplace(Args &&...args)
	{
		return T(std::forward<Args>(args)...);
	}

A slightly-more clever user might even think that using `T{std::forward<Args>(args)...};` would be even better, since it could initialize aggregates. Such a user would be decidedly unaware of the [pitfalls of such an approach][ui problem], but this is true of many users.

Providing an `initialize` function will solve the problem for savvy C++ users, but it will not solve the problem for the more rank-and-file users, who learn primarily through experience.

Furthermore, even if `initialize` were to become widely used, we still have a problem. If a user wants to use `emplace` construction with a list of values that should represent a call to an `initializer_list` constructor, they have to spell this out very verbosely at the call site: `emplace(initializer_list<int>{1, 3, 5})`. By contrast, if they were doing it manually, they could do `T t = {1, 3, 5};`, without having to explicitly provide either the `initializer_list` typename or the template parameter given to `initalizer_list`.

Even if we manage to find a way to deduce the `initializer_list` template parameter from the arguments (via C++17 template constructor mapping syntax, perhaps), it still requires the user to spell out a long typename.

# A Survey of Initialization

`emplace`-style functions have an interface that takes two things:

1. A type to be constructed.

2. A list of values to construct it with.

The core problem we have is trying to take "a list of values" and map that to the correct form of initialization for some type `T`. The problem is this: what exactly is "the correct form of initialization"?

C++ has numerous forms of initialization for various objects. These forms of initialization use different syntax.

When working directly with `T`, a user has the ability to select the "correct form" explicitly. But when working through `emplace`-style functions, the user lacks such tools. The *only* thing they can provide is a sequence of values of distinct types.

Thus, the core problem is this: we must define a single initialization form which, from *a sequence of values alone*, will allow the user to select the "correct form of initialization" for them. If a type is a class type, the user should be able to call every legal constructor on that type. If the type is an aggregate, the user should be able to initialize that aggregate in every way that they could if they were directly creating the object with those values.

The goal is not necessarily to use the same syntax as if one were initializing it directly. The goal is to have one form that provides access to all other forms.

So let us look at the various currently-existing forms of initialization. The purpose here is to see which initialization syntaxes provide coverage of various forms of initialization. With such a survey, we may be able to determine some commonalities that will allow us to resolve this issue more effectively.

## Uniform initialization for aggregates

List initialization was the first attempt to provide a way to say that an any object could be initialized from a sequence of values alone. Of course, the name "uniform initialization" is a misnomer precisely because it failed to achieve this goal. But let us look at precisely where it failed.

List initialization attempts to combine several forms of initialization that existed in C++98/03: constructor initialization and aggregate initialization. From a high-level perspective, there seems to be no overlap here: aggregates don't have constructors, and types with constructors aren't aggregates.

Which leads to the [flaw previously mentioned](#agg-problem): the fact that while aggregates don't have user-provided constructors, they still (potentially) have default, copy, and move constructors. [dcl.list.init]/3.1 does attempt to wallpaper over this hole in the initialization rules, but it only covers types which are either the aggregate type or a type derived from it. Types which are convertible to it are not handled.

Changing 3.1 to include all types implicitly convertible to the aggregate type would provoke problems:

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

Both `a1` and `a2` work, but they both do different things. `a1` invokes implicit conversion of the `InnerAgg` to `OuterAgg`, then calls the copy constructor. Therefore, `a1.f` will be 2.0f.

By contrast, `a2` invokes aggregate initialization. `a2.agg` will be initialized by the temporary, while `a2.f` will be initialized by the default member initializer. Therefore, `a2.f` will be 5.0f.

So if [dcl.list.init]/3.1 were to include implicit conversions, we would lose the ability to invoke do `a2` *entirely*. Furthermore, if `InnerAgg` was not implicitly convertible, then `a1` would fail to compile. And if `InnerAgg` was not the first member of the aggregate, then `a2` would fail to compile.

And our [proposed fix for LWG 2089](#proposed) (whether it's that one or the original fix LWG 2089) has the same problem. `OuterAgg` is always constructible from `InnerAgg`; therefore, we will get `a1` behavior instead of `a2`. And there is no way to undo that.

By contrast, if we used the "fix" that only tests `is_aggregate`, then the caller of the `emplace` function can select the type of initialization they desire. They would do this by explicit conversion: `emplace(OuterAgg(InnerAgg{}))` would invoke convert-and-copy/move behavior, while `emplace(InnerAgg{})` will perform aggregate initialization.

Of course, that is a backwards incompatible change.



## List initialization narrowing

There is another issue with list initialization. Consider the following:

	struct T
	{
	  unsigned char c;
	};

	T t{2};
	
This is valid code. But it is only valid because it uses a constant expression; take that away:

	auto val = 2;
	T t{val};
	
And quite suddenly, this fails to compile.

List initialization in all of its forms precludes narrowing conversions. This also means that the non-aggregate equivalent that has a constructor `T(unsigned char)` also cannot be called like this.

This is one of the main reasons why we cannot 


## List initialization for non-aggregates

That brings us to the classic point-of-failure for list-initialization. The old issue



If we made the change to 3.1, such that it also looks at implicit conversions, then both lines would do the same thing. Indeed, there would be *no way* to provoke aggregate initialization with just one parameter.

An interesting question is this. Under the current 3.1 wording, is there a way to differentiate these two cases with the [above proposed fix](#proposed) to 2089? The answer is... yes.

If the user were initializing an object via `emplace` calls, and we apply our fix defined above, the user can do this: . With the current wording, the user can access the appropriate kind of initialization. But to do so, they must be explicit about what they are doing.

One might also argue that making this explicit also helps make it more clear what's going on. In the former case, we see the "convert and copy"; in the latter case, we see no conversion and copying, so a user may not expect there to be such behavior.

Notice however that performing the implicit conversion requires using a cast operation. Here, we use a constructor-style cast rather than a C-style cast or a `static_cast`. But the important part is that we cannot use list initialization to get this kind of conversion.







# A Language Solution

The above outlines a purely library-based approach. But, as Ville Voutilainen outlined in [N4462][ville], it is very difficult for the user to build their own equivalent to `allocator::construct`, one which exhibits this exact behavior. If we provide an `is_aggregate` trait, the simplest this code would get is something like this:

	template<typename T, typename ...Args>
	auto initialize(Args &&...args)
	{
		if constexpr(is_aggregate_v<T>)
			return T{std::forward<Args>(args)...};
		else
			return T(std::forward<Args>(args)...);
	}

Perhaps a targeted language solution can give us a more succinct way of resolving this issue. One that is easier to teach than the above `if constexpr` test. I do not think it would be a good idea to encourage the use of a `std::initialize` for doing construction on types.

The goals of the language solution are to:

1. Permit `emplace`-like forwarding functions in the standard library to initialize aggregates, without breaking existing code and without accidentally calling the wrong function.

2. Allow the user to easily write their own `emplace`-like forwarding functions, which will have identical behavior to the standard library ones.

3. Make it easy for the caller of the `emplace` function to specify whether they want to invoke `initializer_list` constructors or not. In particular, try to avoid having the user spell out `std::initializer_list<typename>` when they want to use an `initializer_list` constructor.

The solution here requires only one change of actual syntax. The idea is to provide a way to qualify a braced-init-list. These qualifiers affect exactly how the list is used to initialize a type. The syntax will be as follows:

	{<special identifier>: ...}

The "special identifier" is an identifier that defines how the braced-init-list is qualified. The available qualifiers are:

* `c`: The braced-init-list is "constructor-qualified".
* `l`: The braced-init-list is "list-qualified".

Note that these are special identifiers, in accord with [lex.name]; they are not keywords.

[over.match.list] specifies how list-initialization selects constructors for a class. This section has two bullet points, which specify two groups of constructors list-initialization checks, in the given order.

An unqualified braced-init-list uses both bullet points, in the given order. A constructor-qualified braced-init-list will skip paragraph 1.1. A list-qualified braced-init-list will skip paragraph 1.2.

This permits us to resolve LWG 2089 as follows:

> Effects: `::new((void *)p) U{c: std::forward<Args>(args)...}`

This has nearly identical effects to the `if constexpr` version suggested above, but is far more concise. The "nearly" part has to do with narrowing conversions:

	struct T
	{
	  T(unsigned char);
	};

	vector<T> t;
	t.emplace_back(2);
	
This requires an `int`->`unsigned char` conversion. Such a conversion is permitted to happen in direct initialization. However, list-initialization forbids such conversions due to being considered "narrowing". Thus, this would be a backwards-incompatible change.

To resolve this, we declare that constructor-qualified lists will ignore the rules forbidding narrowing conversions (list-qualified lists will still follow them). This would allow the use of `{c:}` to be identical to the `if constexpr` version. This satisfies goal #1.

The outstanding question is this: should we ignore the narrowing conversion rules only if the list is going to call a constructor, or should we ignore the rules all the time on constructor-qualified initialization? That is, is it OK to allow the above to work, but not the following:

	struct T2
	{
	  unsigned char c;
	};

	vector<T2> t;
	t.emplace_back(2);

I do not think it is good for the constructor version to work where the aggregate version fails. So instead, we should permit narrowing on all forms of constructor-qualified list initialization.

The qualified syntax is pretty easy to learn, and is easy to teach. So that satisfies goal #2.

To reach the last goal, the ease of passing `initializer_list`s to functions, we make one more change. One that is rather obvious, given the meaning behind our new syntax.

As it currently stands, the statement `auto x = {...};` will always deduce to an `initializer_list`, based on the types of expressions provided. However, template argument deduction cannot deduce `{...}` into an `initializer_list`. This was done for good reasons; namely, that you don't know exactly what the user meant when they wrote `{...}`. So you force them to state their intent explicitly.

But with list-qualified lists, the writer has already *expressed* their intent. Therefore, we can decide that they can be deduced in template arguments in exactly the same way as for `auto x = {...};`. Furthermore, we should also say that constructor-qualified lists cannot be deduced as `initializer_list`s at all. As such, `auto x = {c: ...};` should fail to compile.

Therefore, users who want to pass an `initializer_list` to an `emplace` function need only do `emplace({l: ...})`.

This also means that `auto x{l: 10};` will always deduce to a single-element `initializer_list<int>`, while `auto x{10};` will become an `int` (per C++17 rules). So this uniformity is another advantage of the syntax.

Additionally, it would allow more complex chains of deduction:

	auto x = {l: {l: 4, 3}, {l: 22, 1}};

Such an initializer list can also be used as the argument to an `emplace` function.

## Ambiguous Solutions

A long outstanding problem with braced-init-lists is the issue of initializer-list constructors hiding other constructors. This is problematic to the point where [some people suggest avoiding such syntax entirely to avoid an unexpected "gotcha"][ui problem], or at least avoiding it in template code.

The most famous ambiguous example being `vector<T>{5}`. For most `T`s, that create 5 default-constructed elements. But if `T` happens to be an integer type, it will create a one-element `vector` that contains the number "5". If `T` is a template parameter to a function/class, then the writer of the code does not know which will be called.

With this new syntax, we can resolve the ambiguity. If the user wanted 5 default-constructed elements, they can say so: `vector<T>{c: 5}`. If the user wanted a single element containing 5, they can also say so: `vector<T>{l: 5}`. The latter will fail if there is no appropriate `initializer_list` constructor, thus ensuring that the user's wishes are not confused.

This permits everyone to use braced-init-list syntax, with no ambiguity about what will be called or what it will mean. As such, one could say that this feature makes "uniform initialization" truly uniform, though obviously it requires two slightly different syntaxes so that the user can express their desires adequately.

## Oddities of the Syntax

This syntax does create some circumstances that a user might not expect.

The semantic meaning behind a list-qualified initializer list, particularly in light of the `initializer_list` deduction rules, is that it is a bundle of values intended to create an `initializer_list`. So one might expect that `{l:}` syntax can only initialize things that take an `initializer_list` parameter. But that is not the actual feature described above.

List-qualified lists are capable of performing initialization that does not involve `initializer_list`s. Because the only initialization difference between a list-qualified initialization and unqualified initialization is the selection of constructors, list-qualified initialization is capable of initializing things in non-list-like fashions.

[dcl.list.init]/3 lists many circumstances that are checked well before a the constructor behavior that list-qualified initialization changes. Perhaps the most unexpected circumstance is paragraph 3.2: the invocation of aggregate initialization, exactly as if the braced-init-list were not qualified.

It makes perfect sense to be able to use list-qualified list-initialization on many aggregates. Array aggregates are the most obvious example. But at the same time, some struct aggregate contain arrays or have members that are conceptually *like* arrays. For example a 3D vector type may store its members as `x`, `y`, and `z` instead of an array, but initializing them as a list seems perfectly reasonable.

The question is whether it reasonable to ever see something like this in code:

	Typename t = {l: "a string"s, 10, std::vector<int>{10, 20, 30}}`;

Given a struct aggregate that has members of varying types, would a user expect to be able to initialize it with a list-qualified init-list? Qualified initialization, as proposed above, would do nothing to prevent this. And while it seems semantically incorrect, we wouldn't be actually breaking any code by permitting this.

The thing is, we don't simply want to say that list-qualified initialization will fail with non-array aggregates, because of the case described above. So it seems that the semantically problemantic case is not the invocation of aggregate initialization. It's the use of a list of heterogeneous types.

So if this is deemed to be unwanted behavior, we could declare that a non-empty list-qualified list must be able to deduce an `initializer_list` of some kind (as if by `auto x = {l: ...};`). This must be true even if the list is used for aggregate initialization. Most important of all, this must be true even if the `initailizer_list` type is of a different type than the one we deduce here. The reason the last one matters is that we still want non-narrowing conversions to work:

	std::vector<float> f{l: 1, 2, 3, 4};

The list `{l: 1, 2, 3, 4}` would naturally deduce to `initalizer_list<int>`. But `vector<float>` only takes an `initializer_list<float>` And we still want that code to work. So what we should do is ensure that deduction is possible, not that the deduced type is what ultimately gets used.

Then again, such a rule may be more complex than it is worth.

# Typed List Qualified Initialization

When deducing `initalizer_list`s from list-qualified braced-init-lists, a problem sometimes can arise. Consider the `vector<vector<float>>` type. If you attempt to use `emplace_back({l: 1, 2, 3, 4})`, that will deduce to the wrong type of list. And `vector`'s `initializer_list` constructor takes exactly and only one type of `initializer_list`, so this will result in a compile failure.

Now, given that we're working with completely new syntax, we could also add syntax to cover this. For example, we could permit the use of `{l typename: ...}`, which would always deduce to `initializer_list<typename>`.

It is less clear exactly how this should behave. Consider this:

	std::vector<float> f{l int: 1, 2, 3, 4};

On the surface, this seems decidedly incoherent, as if the author didn't know what they really wanted. And yet, it could be allowed.

If the rule is that the `<typename>` will only be used in deduced contexts, then this will function just fine, since the init-list is not being deduced. If instead we say that `{l typename: ...}` will always generate an `initializer_list<typename>{...}`, then this would not work. But this would mean that `{l typename:} syntax should never be able to be used on an aggregate, which means that even the following would not be allowed:

    float arr[] = {l float: 1.0f, 2.0f, 3.0f, 4.0f};

While one might question the repetition of using  `float` twice (rather than `auto`), consider a struct aggregate that stored an array of floats. The use of `l float` here would simply be making it clear what's being initialized.

A way to get around this would be to say that if you used a typed-list-qualified list, and that initialization considers `initializer_list` constructors (per [over.match.list]'s bullet point 1), then the only constructors which take `initializer_list<typename>` are considered. So the above `vector<float>` example will fail to compile, while still preserving the ability of such lists to initialize aggregates.

# Acknowledgments

* Joël Lamotte: For coming up with the idea to use {:} syntax.
* Malte Skarupke: For bringing up the issue of using uniform initialization in generic code.

# References

* LWG issue [#2089][1]
* [N4462][ville]: LWG 2089, Towards more perfect forwarding, Ville Voutilainen
* [The Problems With Uniform Initialization][ui problem], Malte Skarupke

[1]: http://cplusplus.github.io/LWG/lwg-active.html#2089
[ville]: http://wg21.link/N4462
[ui problem]: http://probablydance.com/2013/02/02/the-problems-with-uniform-initialization/
[4]: http://ideone.com/n5q0CW