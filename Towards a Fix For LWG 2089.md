% Towards a Fix for LWG 2089
%
% 

[LWG issue 2089][1] outlines a problem with using `emplace`-style construction of objects. The user passes such functions a list of values, and the function forwards these values to the constructor of a particular type.

The problem is that aggregates, by definition, do not have a constructor. Therefore, `emplace`-style functions cannot be used to initialize them. However, we do have list initialization, which can call constructors or perform aggregate initialization, as is appropriate to the type being constructed.

Unfortunately, we cannot simply declare that `emplace`-style functions will always use list initialization. There are actually several reasons for this. One of the principle reasons is that such lists have a preference for `initializer_list` constructors where available, and this preference can break existing code that relied on `emplace`-style functions to call a non-`initializer_list` constructor.

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

> Effects: `::new((void *)p) U(std::forward<Args>(args)...)` if `is_constructible<U, Args...>::value` is `true`, else `::new((void *)p) U{std::forward<Args>(args)...}` if `is_aggregate<U>` is true, else il-formed.

This version will only use list-initialization on actual aggregates, and only when constructor initialization is not possible. It therefore represents a more targeted fix.

## Unfixable Flaws

The fundamental problem we have is this: we want to be able to initialize objects of some known type `T`, given an ordered sequence of values. C++ has many forms of initialization from a sequence of values, each with its own syntax. But in our case, the selection of syntax is made for us; the only thing we can control is exactly what types/values are provided.

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

# A New Form of Initialization

It is vital to understand that, by implementing *any* backwards-compatible form of fix for LWG 2089, we will be inventing a *new form* of initialization. It is a means of initializing objects which is a unique combination of initialization, reliant on other forms yet the collection is distinct from them. And we will be standardizing this initialization form, employing it throughout the standard library.

By applying this new initialization form to all `emplace`-style APIs, users will need to be aware of its behavior. As Ville Voutilainen outlined in [N4462][ville], it is very important for users to be able to use this form of initialization as well. When a user goes to implement their own `emplace`-style APIs, we want them to have the same behavior.

## Standard Library Function

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

## Possible Language Features

The most important reason to prefer a language feature over a library function is that we are creating a new form of initialization, one which will be used throughout the standard library. As such, users will need to be familiar with it. By giving it C++ syntax, it is much more discoverable than a utility function buried in the C++ standard library. As such, users will be better able to make it work for them.

If we create a good language feature for using this kind of initialization, users will instinctively gravitate to it when writing their own `emplace`-style functions. This will promote a consistent feel for this kind of initialization.

We shall analyze two syntactic approaches to providing this form of initialization:

### 1: Constructor Syntax Extension

As it currently stands, constructor syntax cannot invoke aggregate initialization. We could change that. If we do, then the [rules proposed here](proposed) naturally fall out. Constructor syntax will use the existing rules, and any syntax which would be il-formed when applied to an aggregate will instead invoke aggregate initialization if the type is an aggregate.

One downside here is that [dcl.init.aggr]/3 forbids narrowing conversions. So if we invoke aggregate initialization, then we would cause `()` constructor syntax to prevent narrowing conversions. This would probably be unexpected from users, but it would likely be acceptable.

Another downside has to do with default member initializers:

	struct Outer
	{
		Inner i(stuff);
	};

To avoid parsing ambiguity issues with member functions, a brace-or-equal-initializer cannot directly use constructor syntax. This requires users to do this instead:

	struct Outer
	{
		Inner i = Inner(stuff);
	};

This needlessly repeats the typename. This could be alleviated by permitting `auto` to be used for members which have a equals-style brace-or-equal-initializer.

### 2: Qualified List Initialization

List initialization has a lot of really useful properties, syntactically speaking. Copy-list-initialization can deduce the type to be constructed based on the context in which it is used, thus [promoting DRY principles](https://en.wikipedia.org/wiki/Don't_repeat_yourself). It is already hooked into existing syntactic mechanisms like default member initializers, so we wouldn't have to change them.

The main problem with list initialization is not how it is invoked; it is [how it behaves][ui problem]. So it would be quite ideal if we had a form of initialization which keys off of list initialization *syntax*, but the actual initialization behavior is in accord with our rules here. Indeed, users may even see widespread use as the default means of initializing objects.

A possible syntax for this would be as follows:

	{c: stuff};

This would be called a "constructor-qualified braced-init-list". Regular braced-init-lists are "unqualified braced-init-lists". Applying a constructor-qualified braced-init-list to an object will invoke "constructor-qualified list-initialization".

Constructor-qualified list-initialization must behave exactly [as we proposed above](proposed). As such, it completely ignores the *entire suite* of list-initialization rules in [dcl.init.list]/3. The rules it uses to initialize `T` shall be as follows:

1. `T{c: stuff}` will attempt to invoke `T(stuff)`. If there are no constructors that `T(stuff)` would call, then proceed ahead. Otherwise, invoke `T(stuff)` (and if that leads to multiple constructors, this `T{c: stuff}` shall be il-formed). If copy-list-initialization is being performed, and an `explicit` constructor is selected, the program is il-formed.

	For all intents and purposes, this invokes [over.match.list] to find a constructor, except 1.1 is skipped.
	
2. If `T` is an aggregate type, then it will initialize `T` via [dcl.init.aggr]. This may not be equivalent to `T{stuff}`, since that might invoke parts of [dcl.init.list]/3 that don't provoke aggregate initialization. If `T` cannot be aggregate initialized with `stuff`, then `T{c: stuff}` is il-formed.
3. Otherwise (ie: no matching constructor and `T` is not an aggregate) the code is il-formed.

Note that this means that narrowing conversions on the elements of a constructor-qualified braced-init-list are perfectly fine when invoking a constructor (though they will still affect aggregate initialization, in accord with [dcl.init.aggr]/3). Without this caveat, this would not be a backwards compatible fix to LWG #2089.

The principle downside of this fix is that there is a very large behavioral difference between the behavior of `{stuff}` and `{c: stuff}`.

**Designated Initializers**

Constructor-qualified braced-init-lists should not be able to use designated initializers. This makes sense, as the purpose of this initialization syntax is not to solely initialize aggregates. If you want designated initializers, then you know you're dealing with an aggregate, so you don't need `c:` syntax at all.

# Companion: The initializer_list Problem

Regardless of how we expose this new form of initialization, whether as a language feature or a library function, users will still have one problem. That problem is how to cleanly provide an `initializer_list` for `initializer_list` constructors.

Consider the following:

    vector<array<int, 4>> va = {{1, 2, 3, 4}, {10, 20, 30, 40}}
    vector<vector<int>> vv = {{1, 2, 3, 4}, {10, 20, 30, 40}};

Both of these are equally short and clean pieces of code that make it clear exactly what is going on. There is no extra redundancy, yet users can easily see how this works. However, using `emplace_back` to insert shows a sharp difference:

	va.emplace_back(1, 2, 3, 4); //Uses aggregate initialization.
	vv.emplace_back(1, 2, 3, 4); //Compile error

Our [suggested resolution to LWG 2089](proposed) explicitly disallowed the second one, on the grounds that `vv.emplace_back(1, 2)` would have radically different behavior from `vv.emplace_back(1, 2, 3)`. As such, the natural solution is to have the user explicitly provide an `initializer_list`:

	vv.emplace_back({1, 2, 3, 4}); //Compile error

This does not work, since a bare braced-init-list cannot be deduced as a type when provided as a function argument to a template function.

It should be noted that the `in_place_t` functions added to `any`, `optional`, and `variant` actually have a solution to this problem. They have a second overload explicitly for `initializer_list<T>`, so long as the target type in question can be constructed from one (and the other provided arguments).

Even so, this is a problem that could better be solved in the language itself. The main problem is that braced-init-lists cannot be deduced into `initializer_list<T>`s by template parameters. And this is a good thing; it is not clear if the user meant to get an `initializer_list<T>` or some other type. So it's best to let the user make this explicit.

The problem is that making it explicit is *really* long-winded:

	vv.emplace_back(std::initializer_list<int>{1, 2, 3, 4}); //Valid code.

This actually cannot be done with a language feature. Consider this potential function:

	template<typename T>
	std::initializer_list<T> il(std::initializer_list<T> val)
	{
		return val;
	}

While this would allow is to (theoretically) do `vv.emplace_back(std::il({1, 2, 3, 4}))`, it plays havok with the lifetime rules for `initializer_list`s.

When a braced-init-list is transformed into an `initializer_list`, a static array is created along with it. The linked of this temporary is linked to the lifetime of the object that was initialized with the `braced-init-list`. Which, in the case of a call to `il` will be the parameter to that function.

Thus, the array may be destroyed at the end of this function call. Or the end of the expression that called `il`. As such, if we want to make this feature safely, we need to do it in a way that works. And that requires a language feature.

## List Converter

There are two cases that we want to cover: one where the compiler deduces the type of `initializer_list`, and one where the user provides an explicit type to use. Case 1 is equivalent to `auto x = {more-than-one-value};` rules. Case 2 is when you want the compiler to use a specific `T` as well, in accord with `intializer_list<T> x = {values};` rules.

A possible syntax for this would be as follows:

	{l: stuff} //Converts to an `initializer_list`, but deduces `T` from the values.
	{l Typename: stuff} //Converts to `initializer_list<Typename>`.
	
This would be called a "list-qualified braced-init-list", which shall provoke "list-qualified list-initialization". The first syntax will behave in accord with our existing `auto x = {more-than-one-value};` syntax, though `stuff` is permitted to have exactly one value. The second syntax will behave in accord with `intializer_list<Typename> x = {values};`.

Note that narrowing conversions will be forbidden, just as with the behavior of the equivalent syntaxes. Also, just as with `c:` syntax, designated initializers are forbidden.

## List to Aggregate?

The above focuses on the use of list-qualified lists for building `initializer_list`s, usually via deduction. But what about initializing other things?

Should list-qualified list initialization be able to invoke aggregate initialization, or even call `initializer_list` constructors of objects? It sounds like a very useful idea. Conceptually, these all seem to mean the same thing:

	auto il = {l: 1, 2, 3}; //initializer_list<int>
	int arr[] = {l: 1, 2, 3};
	std::array<int, 3> arr2 = {l: 1, 2, 3};
	std::vector<int> v = {l: 1, 2, 3};
	
	struct IVec3 { int x; int y; int z; };
	IVec3 agg = {l: 1, 2, 3};

This code seems to express the intent of the user very effectively, without ambiguity.

There are a couple of problems with this. First, such lists can be applied to aggregates that are not arrays. `IVec3` above is an aggregate of 3 variables, but is conceptually array-like in its meaning. It is a 3D vector of integers, not unlike an `array<int, 3>`. So conceptually, it makes sense to be able to use list-qualified list-initialization on it.

So how do we differentiate between aggregates that are conceptually array-like and those that are not? Would we allow any arbitrary values within the braces?

It would make sense to require that list-qualified lists are il-formed if the values given cannot deduce to some kind of `initializer_list<T>`. Aggregate initialization won't use them, but the deduction test ensures that you could only use list-qualified aggregate initialization with a list of homogenous values.

The other problem deals with providing an explicit type when initializing an aggregate:

	Agg agg2 = {l float: 1, 2, 3};

Exactly how should this behave? It doesn't trigger the previous rule, since the values do not require a narrowing conversion to fit into a `float`. But `Agg` is a sequence of `int` types. Does it make sense to permit what the user has explicitly claimed is a sequence of `float` to initialize a sequence of `int`?

The rules for list-qualified list-initialization overall would be as follows. Just as with constructor-qualified initialization, we ignore all of [dcl.init.list]/3 (except for any clause explicitly invoked):

1. If the list-qualified list is type qualified, attempt to initialize an `initializer_list<Q>` with the values, where `Q` is the given type. If that fails, then the code is il-formed.
2. If the list-qualified list is not type qualified, use [temp.deduct.call] rules to deduce an `initializer_list` type `Q`. The list shall behave in later steps as though it were type qualified with `Q`.
3. If `T` is an aggregate, perform aggregate initialization with the members of the list.
4. If `T` is a user-defined type, attempt to call initializer list constructors, in accord with [over.match.list], except that paragraph 1.2 is ignored. Also, ignore the bit about empty lists using the default constructor.
5. If `T` is a placeholder (whether a variable or a return value), deduce `T` as `initializer_list<Q>`, where `Q` is the qualified type.
6. If `T` is a template argument undergoing template argument deduction, deduce `T` as `initializer_list<Q>`, where `Q` is the qualified type.

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