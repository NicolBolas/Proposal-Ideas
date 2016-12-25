% Towards a Fix for LWG 2089
%
% 

[LWG issue 2089][1] outlines a problem with using `emplace`-style construction of objects. The user passes such functions a list of values, and the function forwards these values to the constructor of a particular type.

The problem is that aggregates, by definition, do not have a constructor. Therefore, `emplace`-style functions cannot be used to initialize them. However, we do have list initialization, which can call constructors or perform aggregate initialization, as is appropriate to the type being constructed.

Unfortunately, we cannot simply declare that `emplace`-style functions will always use list initialization. This would not be backwards compatible with existing code. Even if that were not true, it would hide access to certain constructors.

This paper evaluates the current solution, presents a hopefully better one, and suggests a number of ways to expose this solution to users.

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

I suspect that users would find this to be surprising. A user would probably expect `emplace` for a non-aggregate to either act exactly like list-initialization or act like constructor initialization, with aggregate initialization used only for actual aggregates. They would not expect it to have behavior that is distinct from both of the existing language-based cases.

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

The initialization of `a2` does *not* work, because it attempts to use aggregate initialization. However, the `is_aggregate` version of `emplace` will *always* use aggregate initialization on aggregates, regardless of what types the user passes. Thus, the `is_aggregate`-based fix represents a backwards-incompatible change, because right now `vector<Agg>::emplace(Value{})` is perfectly valid code.

<a name="proposed"/>Therefore, a complete fix which is backwards-compatible while making sure not to invoke `initializer_list` constructors improperly would be the following:

> Effects: `::new((void *)p) U(std::forward<Args>(args)...)` if `is_constructible<U, Args...>::value` is `true`, else `::new((void *)p) U{std::forward<Args>(args)...}` if `is_aggregate<U>::value` is true, else ill-formed.

This version will only use list-initialization on actual aggregates, and only when constructor initialization is not possible. It therefore represents a more targeted fix.

## Unfixable Flaws { #unfixable }

The fundamental problem which is the basis of LWG 2089 is this: we want to be able to initialize objects of some known type `T`, given *only* an ordered sequence of values. C++ has many forms of initialization from a sequence of values, each with its own syntax. But in our case, the selection of syntax has to be made for us; the only thing we can control is exactly what types/values are provided.

The goal is to find a create a way of initializing objects that will allow you to initialize them in any way that you could have without . 

But there are some corner cases that simply can't be resolved. Consider this:

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

`a2` is being initialized via list initialization. `InnerAgg` is not related to `OuterAgg`, so clause [dcl.init.list]/3.1 (invoking copy/move) is skipped. So clause 3.3 kicks in: invoke aggregate initialization. `InnerAgg{}` can initialize `OuterAgg::agg`, so it will do so. The rest will get their default values. So `a2.f` will be 5.0f.

Given the proposed solution of construct-then-aggregate-initialize, `emplace(InnerAgg{})` will always produce `a1`, since `InnerAgg` is implicitly convertible to `OuterAgg`. Therefore, the only way to construct things in `a2`'s style is via `emplace(OuterAgg{InnerAgg{}})`.

Unfortunately, there is no way to avoid invoking a copy/move constructor from `emplace`'s parameter into the final object. Whereas `emplace(InnerAgg{})` will not need this from either type.

There is no single syntax that can possibly allow both via an `emplace(InnerAgg{})` call. This is not a real criticism of the resolution, so much as a recognition that a perfect solution is fundamentally impossible (or at least, without a much bigger language fix). There will always be some sub-optimal corner case.

So long as such cases are a rarity, the syntax should be adequate.

<a name="disuniform"/>There is also the issue of non-uniformity in initialization. In order to use these initialization forms, you really need to know *what* you're initializing.

For example, using `emplace` for an aggregate type like `std::array` will not look like `emplace` for a user-defined type like `std::vector`. While both would respond to uniform initialization syntax in the same way, `emplace` initialization cannot do so:

	vector<array<int, 2>> va;
	vector<vector<int>> vv;
	va.emplace_back(1, 2); //Initializes the aggregate.
	vv.emplace_back(initializer_list<int>{1, 2}); //Calls an initializer list constructor.
	
The syntax for `va` cannot be used with `vv`, and vice-versa.

Much like the prior issue, there's not much of a way around it.

## A New Form of Initialization

It is vital to understand that, by implementing *any* fix for LWG 2089, we will be introducing a *new form* of initialization to C++. It is a means of initializing objects which is a unique set of initialization rules, reliant on other forms yet the collection is distinct from them. And we will be standardizing this initialization form, employing it throughout the standard library.

By applying this new initialization form to all `emplace`-style APIs, users will need to be aware of its behavior. As Ville Voutilainen outlined in [N4462][ville], it is very important for users to be able to not merely understand this form of initialization, but to write it into their own APIs. When a user goes to implement their own `emplace`-style APIs, we want them to have the same behavior.

What follows are a set of possible ways to resolve the problem. At the end of each section is a sub-section explaining the advantages and disadvantages of the mechanism. This paper makes no particular suggestion as to which to take at the present time; it is primarily here to explore the space for both library and language solutions.

# LWG 2089 as a Library Feature { #lib2089 }

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

## Pros and Cons { #lib2089_pro_con }

The principle advantage of this library-only feature is that it is self-contained. We would be introducing a new initialization form, but it would be one that lives in its own space, and one that is based on existing language features.

The downsides of such a solution, relative to any language feature, are:

1. Visibility. A language feature could represent a form of initialization that users could commonly use. Whereas `std::initialize` would be something tucked away in the utility portion of the standard library, where it would primarily be known about by experts rather than the hoi polloi. 

2. Consistency. Because of the lack of visibility of the feature, when users want to create their own `emplace`-style APIs, they will likely use constructor or list initialization syntax rather than `std::initialize`. As such, they are not likely to get the same behavior as the standard library APIs.

# Initialization Control Through Parameters { #control_library }

Recall that the problem we have is that C++ has many forms of initialization, but we as the indirect initializer can only provide a sequence of parameters to whatever initialization the code chooses to use.

LWG 2089 solves this problem by declaring that, if the sequence of parameters for one form of initialization is invalid, then it tries another form. This gives priority to one form of initialization, but leads to a [number of unfixable problems](#unfixable). LWG 2089 essentially hopes that these are infrequently encountered corner cases and/or that users can work around them when they are encountered. But otherwise, LWG 2089 offers no complete solution.

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

We could apply LWG 2089's solution to the first overload of `initialize`, in addition to having these alternate overloads.

## Pros and Cons

There are two advantages. The first is that this syntax visibly captures the users intent, allowing users to directly spell out *exactly* what they want at the time they call the indirect function. This is preferable to relying on esoteric initialization forms.

The second advantage is that it fixes all of those [supposedly unfixable](#unfixable) problems. Or more to the point, it gives users the ability to resolve those conflicts themselves.

Since it is a library-only solution, it has all of the [disadvantages of the prior solution](#lib2089_pro_con). Though in this case, if you fail to use `std::initialize` in your indirect initialization functions, you'll have problems almost immediately when someone uses one of the specific constructor forms.

# Control Parameters in The Language { #control_language }

The use of control parameters has some good aspects. But it has all of the problems of any library-only solution: we are creating a new form of initialization via a set of rules bound up in the library. That makes it very easy for users to be unaware of this kind of initialization. And it is very important that this form of initialization be used in indirect functions at the very least.

However, because these control parameters are based on values and types that do not yet exist in the standard, we could modify the initialization rules to explicitly recognize them and abide by them.

For direct or copy initialization, we can change the rules to ignore the first parameter to any constructors if that type is `std::construct_init_t`. If the first parameter is of type `std::list_init_t`, then it uses the rest of the parameters in list initialization as though they were in a braced-init-list.

For list initialization, we do something similar. If the first element of the braced-init-list is `construct_init_t`, then the rest of the parameters are forwarded to direct or copy initialization. If the first element is `list_init_t`, then the rest of the parameters are forwarded back through list initialization.

## Pros and Cons

One of the main benefits here is that we can access all of the advantages of list initialization (use in brace-or-equals initializers, automatic deduction of the type being initialized, etc), but *without* the [issues of its sometimes-oddball initialization rules][ui problem].

	vector<int> func() {return {std::construct_init, 5, 10};}

This will always create a 5-element vector, with each element storing 10.

As a language feature, it becomes more visible to users in different contexts. Indeed, given the above, users will likely want to use it frequently, which means users will be more likely to be trained to expect it. Also, it would require deliberate effort to prevent an indirect initializing function from allowing this functionality.

One really important downside here is that this creates lots of complexity in the initialization rules. What happens if the first two parameters to the initializer are `construct_init_t` and `list_init_t`? Do you want this to mean use list initialization or constructor initialization, with the first element being of type `list_init_t`?

Also, how do these types behave on initialization? With a library-only solution, they behaved exactly like ordinary types. Once these types are part of general initialization, they start doing odd things:

	auto x = std::construct_init;

What it ought to do is copy from the value into a new variable. However, the first parameter to the initializer is of type `std::construct_init_t`, which means that we should ignore it, right? Which means that it should perform default or value initialization.

But for what type? What type do we deduce `x` to be? If the first parameter is ignored, then clearly it shouldn't be part of that deduction.

These are hardly unfixable problems. But they require a very great deal of thought and care in an already complex part of the standard.

# LWG 2089 by Constructor Extension { #lang2089_constructor }

As it currently stands, constructor syntax cannot invoke aggregate initialization. We could change that. If we do, then the [rules proposed here](#proposed) naturally fall out. Constructor syntax will use the existing rules, and any constructor calls that would be ill-formed when applied to an aggregate will instead invoke aggregate initialization if the type is an aggregate.

## Pros and Cons

This has the advantage of 

One downside here is that [dcl.init.aggr]/3 forbids narrowing conversions. So if we invoke aggregate initialization, then we would cause `()` constructor syntax to prevent narrowing conversions. This would probably be unexpected from users, but it could be acceptable. Of course, we could also declare that if aggreagte initialization happens via constructor syntax, narrowing conversions would be allowed.

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

# LWG 2089 Language by List Initialization Extensions { #lang2089_list }

List initialization has a lot of really useful properties, syntactically speaking. Copy-list-initialization can deduce the type to be constructed based on the context in which it is used, thus [promoting DRY principles](https://en.wikipedia.org/wiki/Don't_repeat_yourself). It is already hooked into existing syntactic mechanisms like brace-or-equal-initializers, and the type it creates can be deduced in various contexts.

The main problem with list initialization is not how it is invoked on types; it is how it behaves. Ignoring the [various problems with its behavior][ui problem], for our needs here, it does not behave in a backwards-compatible fashion with constructor syntax.

So it would be quite ideal if we had a form of initialization which keys off of list initialization *syntax*, but the actual initialization behavior is in accord with our rules here.

A possible syntax for this would be as follows:

	{c: stuff};

This would be called a "c-qualified braced-init-list". Regular braced-init-lists are "unqualified braced-init-lists". Applying a c-qualified braced-init-list to a type will invoke "c-qualified list-initialization".

Note that syntactically, these work identically to a braced-init-list, and this kind of initialization is still list-initialization. As such, `T t = {c: stuff};` is still copy-list-initialization rather than direct-list-initialization.

Also, note that the `c` here is not a keyword. It is a "special identifier", under the rules for such things. As such, there should be no need to reserve it as a keyword, and it should not interfere in parsing existing braced-init-lists.

C-qualified list-initialization must behave exactly [as we proposed above](#proposed). As such, its rules are *completely separate* from the list-initialization rules in [dcl.init.list]/3. Those rules become the rules for unqualified list-initialization.

The rules for c-qualified list-initialization for a type `T` shall be as follows:

1. If `T` is a character array and the initializer list has a single element that is an appropriately-typed string literal (8.6.2), initialization is performed as described in that section.

2. If `T` is a class type, constructors are considered. The applicable constructors are enumerated and the best one is chosen through overload resolution ([over.match], [over.match.list]). Paragraph [over.match.list]/1.1 is ignored. If more than one constructor is considered equally viable, the program is ill-formed. If no matching constructor is found, proceed.

3. If the initializer list has a single element of type `E` and either `T` is not a reference type or its referenced type is reference-related to `E`, the object or reference is initialized from that element (by copy-initialization for copy-list-initialization, or by direct-initialization for direct-list-initialization).

4. If `T` is a reference type, a prvalue of the type referenced by `T` is generated. The prvalue initializes its result object by c-qualified copy-list-initialization or c-qualified direct-list-initialization, depending on the kind of initialization for the reference. The prvalue is then used to direct-initialize the reference.

5. If `T` is an aggregate, aggregate initialization is performed ([dcl.init.aggr]).

6. If the list contains no elements, the object is value-initialized.

7. Otherwise the program is ill-formed.

The order of rules place aggregate initialization last (well, before value-initialization). 

Note that this means that narrowing conversions on the elements of a c-qualified braced-init-list are perfectly fine when invoking a constructor (though they will still affect aggregate initialization, in accord with [dcl.init.aggr]/3). Without permitting narrowing conversions on constructor calls, this would not be a backwards compatible fix to LWG #2089.

The different narrowing behavior between constructor initialization and aggregate initialization with c-qualified initialization could be avoided by permitting aggregate initialization from a c-qualified list to not prohibit narrowing.

The principle downside of this fix is that there is a very large difference in initialization behavior between `{stuff}` and `{c: stuff}`. While that is the main point, it does create a few elements of unexpected incongruity. Unqualified list-initialization forbids narrowing in all cases, while c-qualified initialization only forbids it for aggregate initialization (if even then). Furthermore, `{c: stuff}` will never create an `initializer_list`, even directly:

	initializer_list<int> il{c: 1, 2, 3, 4};

`initializer_list` has no constructor that take 4 integer parameters. And none of the other cases apply. So it cannot be constructed in this way.

Qualified braced-init-lists should not be able to use designated initializers. This makes sense, as the purpose of this initialization syntax is not to solely initialize aggregates. If you want designated initializers, then you know you're dealing with an aggregate, so you don't need the qualification syntax at all.

## The initializer_list Problem

Regardless of how we expose this new form of initialization, whether as a language feature or a library function, users will still have one problem. That problem is how to cleanly provide an `initializer_list` for `initializer_list` constructors.

Consider the following:

    vector<array<int, 4>> va = {{1, 2, 3, 4}, {10, 20, 30, 40}}
    vector<vector<int>> vv = {{1, 2, 3, 4}, {10, 20, 30, 40}};

Both of these are equally short and clean pieces of code that make it clear exactly what is going on. There is no extra redundancy, yet users can easily see how this works. However, using `emplace_back` to insert shows a sharp difference:

	va.emplace_back(1, 2, 3, 4); //Uses aggregate initialization.
	vv.emplace_back(1, 2, 3, 4); //Compile error

Our [suggested resolution to LWG 2089](#proposed) explicitly disallowed the second one, on the grounds that `vv.emplace_back(1, 2)` would have radically different behavior from `vv.emplace_back(1, 2, 3)`. As such, the natural solution is to have the user explicitly provide an `initializer_list`:

	vv.emplace_back({1, 2, 3, 4});

This does not work, since a bare braced-init-list cannot undergo template argument deduction in this way. As well as other similar restrictions.

It should be noted that the `in_place_t` functions added to `any`, `optional`, and `variant` actually have a solution to this problem. They have a second overload explicitly for `initializer_list<T>`, so long as the target type in question can be constructed from one (and the other provided arguments).

Even so, this is a problem that could better be solved in the language itself. The main problem is that braced-init-lists cannot be deduced into `initializer_list<T>`s by template parameters. And this is a good thing; it is not clear if the user meant to get an `initializer_list<T>` or some other type. So it's best to let the user make this explicit.

The problem is that making it explicit is *really* long-winded:

	vv.emplace_back(std::initializer_list<int>{1, 2, 3, 4}); //Valid code.

One might consider adding a standard library function that takes an `initializer_list<T>` as a parameter and just returns it:

	template<typename T>
	std::initializer_list<T> il(std::initializer_list<T> val)
	{
		return val;
	}

While this would (theoretically) allow us to do `vv.emplace_back(std::il({1, 2, 3, 4}))`, it plays havok with the lifetime rules for `initializer_list`s.

When a braced-init-list is transformed into an `initializer_list`, a temporary array is created along with it. The lifetime of this temporary is linked to the lifetime of the object that was initialized with the braced-init-list. Which, in the case of a call to `il` will be the parameter to that function.

Thus, the array may be destroyed at the end of this function call. Or the end of the expression that called `il`. As such, if we want to make this feature safely, we need to do it in a way that works. And that requires a language feature.

## List Qualification

There are two cases that we want to cover: one where the compiler deduces the `T` in `initializer_list<T>`, and one where the user provides an explicit type to use. Case 1 is equivalent to `auto x = {more-than-one-value};` rules. Case 2 is when you want the compiler to use a specific `T` as well, in accord with `intializer_list<T> x = {values};` rules.

A possible syntax for this would be as follows:

	{l: stuff} //Converts to an `initializer_list`, but deduces `T` from the values.
	{l Typename: stuff} //Converts to `initializer_list<Typename>`.
	
This would be called an "l-qualified braced-init-list", which shall invoke "l-qualified list-initialization". The second syntax a "type-qualified list" in addition to be an "l-qualified" list. The first syntax will behave in accord with our existing `auto x = {more-than-one-value};` syntax, though `stuff` is permitted to have exactly one value. The second syntax will behave in accord with `intializer_list<Typename> x = {values};`.

As with `c` above, `l` is a special identifier, not a keyword.

Note that narrowing conversions will be forbidden, just as with the behavior of the equivalent syntaxes. Also, just as with `c:` syntax, designated initializers are forbidden.

## List to Aggregate

The above focuses on the use of l-qualified lists for building `initializer_list`s, usually via deduction. But what about initializing other things?

Should l-qualified list initialization be able to invoke aggregate initialization, or even call `initializer_list` constructors of objects? It sounds like a very useful idea. Conceptually, these all seem to mean the same thing:

	auto il = {l: 1, 2, 3}; //initializer_list<int>
	int arr[] = {l: 1, 2, 3};
	std::array<int, 3> arr2 = {l: 1, 2, 3};
	std::vector<int> v = {l: 1, 2, 3};
	
	struct IVec3 { int x; int y; int z; };
	IVec3 agg = {l: 1, 2, 3};

This code seems to express the intent of the user very effectively, without ambiguity.

There are a couple of problems with this. First, such lists can be applied to aggregates that are not arrays. `IVec3` above is an aggregate of 3 variables, but is conceptually array-like in its meaning. It is a 3D vector of integers, not unlike an `array<int, 3>`. So conceptually, it makes sense to be able to use l-qualified list-initialization on it.

So how do we differentiate between aggregates that are conceptually array-like and those that are not? Would we allow any arbitrary values within the braces?

It would make sense to declare that l-qualified lists are ill-formed if the values given cannot deduce to some kind of `initializer_list<T>`. Aggregate initialization won't use the result of that deduction, but the deduction test ensures that you could only use l-qualified aggregate initialization with a list of homogeneous values.

The other problem deals with providing an explicit type when initializing an aggregate:

	Agg agg2 = {l float: 1, 2, 3};

Exactly how should this behave? It doesn't trigger the previous rule, since the values do not require a narrowing conversion to fit into a `float`. But `Agg` is a sequence of `int` types. Does it make sense to permit what the user has explicitly claimed is a sequence of `float` to initialize a sequence of `int`?

Similarly, should this work:

	std::vector<float> vf = {l int: 1, 2, 3};
	
`vector<float>` only has an `initializer_list` constructor for `float`. So, should this invoke it, even though it is qualified to use `int`?

The rules for l-qualified list-initialization overall would be as follows. Just as with c-qualified initialization, we ignore all of [dcl.init.list]/3 (except for any clause explicitly invoked):

1. If the list is type-qualified, attempt to initialize an `initializer_list<Q>` with the values, where `Q` is the given type. If that fails, then the program is ill-formed.
2. If the list is not type-qualified, use [temp.deduct.call] rules to deduce an `initializer_list` type `Q`. The list shall behave in later steps as though it were type qualified with `Q`.
3. If `T` is an aggregate, perform aggregate initialization with the members of the list.
4. If `T` is an `initializer_list<D>`, then initialize it with the values in the list, in accord with the usual `initializer_list` creation rules.
4. If `T` is a class type, attempt to call constructors, in accord with [over.match.list], except that paragraph 1.2 is ignored. Also, ignore the bit about empty lists using the default constructor.
5. If `T` is a placeholder (whether a variable or a return value), deduce `T` as `initializer_list<Q>`, where `Q` is the qualified type.
6. If `T` is a template argument undergoing template argument deduction, deduce `T` as `initializer_list<Q>`, where `Q` is the qualified type.

This will permit all of the above cases to work, even the ones which seem odd.

## Aggregate Uniformity

In a previous section, [this example](#disuniform) was given to represent a distinction between initializing an aggregate and initializing a non-aggregate with an `initializer_list` constructor. Adjusted for our new syntax, this is the result:

	vector<array<int, 2>> va;
	vector<vector<int>> vv;
	va.emplace_back(1, 2); //Initializes the aggregate.
	vv.emplace_back({l: 1, 2}); //Calls an initializer_list constructor.

This functions well enough. However, it would be nice if there was a way to have a single syntax that could initialize both. We know that making `vv.emplace_back(1, 2)` call the `initializer_list` constructor is a complete non-starter, as it breaks backwards compatibility (the example uses 2 elements to demonstrate this). Therefore, we could entertain the second possibility: somehow make `va.emplace_back({l: 1, 2})` invoke aggregate initialization.

We could make it possible to initialize an aggregate with an `initializer_list`. Not merely with a braced-init-list, but with a runtime instance of an `initializer_list` value.

Given any particular aggregate, a compiler could generate the code to runtime initialize an aggregate from an `initializer_list<T>`. It would effectively give an aggregate an `initializer_list<T>` constructor, but one that will only be called when using constructor syntax. So `Agg({...})` could call it (and therefore `Agg{c:{...}}` will too), but `Agg{...}` will use unqualified list-initialization rules.

There are a number of cases to consider:

1. The `initializer_list` has fewer values than the aggregate. This should do what compile-time aggregate initialization would do: initialize what it can with the given values, then value initialize the rest (or use default member initializers).

2. The `initializer_list` has more values than the aggregate. When using a braced-init-list, this is ill-formed. Therefore, we should probably throw an exception of some form.

3. Each member of the aggregate must be able to be initialized by `T`. And such initialization should not require a narrowing conversion. Note that this applies even to aggregate members that are not being initialized by that particular `initializer_list`.

Static reflection would be difficult to use to perform this operation, since the size of an `initializer_list` is a runtime quantity.

Indeed, that is the biggest problem with this idea: code generation for aggregate initialization from `initializer_list`. Despite `initializer_list::size` being `constexpr`, once you pass an `initializer_list` it as a parameter, you lose the ability to handle it at compile-time. This is why step 2 above requires a runtime exception. Even ignoring exceptions, performing step 1 requires a lot of conditional initialization based on a runtime property.

We do have one advantage: `{l: stuff}` is a new piece of syntax. As such, we are free to give it new rules.

So we could have `{l: stuff}` lead to the generation of `std::sized_init_list<T, N>`. This should be a class publicly derived from `initializer_list<T>`. As such, it should be able to be used in such interfaces (calling `initializer_list` constructors). However, functions that takes variadic parameters like `emplace` will deduce a `sized_init_list<T, N>`.

Note that this would affect *all uses* of the syntax: `auto x = {l: stuff}` will deduce a `sized_init_list<T, N>` and so forth.

With such an object in hand, we can *statically* perform aggregate initialization from the type. The compiler can statically generate code for each particular N, and thus not have to generate code that does runtime checks. Throwing exceptions in step 2 would be completely unnecessary; if `N` is larger than the number of elements in the aggregate, you get a compile error.

And therefore, the distinction between these will mostly disappear:

	Agg agg{l: stuff};
	vector<Agg> va;
	va.emplace_back({l: stuff});

The only difference that `emplace_back` will provoke a copy from values in the `sized_init_list`, while `agg` will not. But that's a distinction which cannot be avoided, since we're using parameters of a function to remotely initialize an object.

## Conceptual Foundation

When designing a solution that is as complex as this, it is useful to look back on the solution to see what it means conceptually. If it does fit into a logical, conceptual framework, then it helps show that the solution is reasonable.

This language feature suggests adding 2 list initialization forms: c-qualified and l-qualified. What do all of these forms of initialization mean?

Unqualified list initialization means to treat an object as a collection of heterogeneous values. This is what an aggregate is, after all; we simply extend that initialization to other types. This is part of the reason for prioritizing `initializer_list` constructors; homogeneous sequences of values are a subset of heterogeneous values. Such constructors therefore initialize the type as a collection of values, like a user-defined aggregate. We permit non-`initializer_list` constructors to be called, but they are conceptually treating constructor parameters as equivalent to internal object state.

C-qualified list initialization means to treat an object's initialization like a constructor call. Constructor parameters are constructor parameters; whether they match internal object state is irrelevant. Aggregates are permitted, but the user is conceptually treating the aggregate as if it had a constructor that initialized its members from the parameters.

L-qualified list initialization means to treat an object as a collection of homogeneous values. Not every object can be initialized in such a way; not even every aggregate.

## Pros and Cons


# Acknowledgments

* Joël Lamotte: For coming up with the idea to use {:} syntax.
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