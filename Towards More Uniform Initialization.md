% Towards More Uniform Initialization, to fix LWG 2089
%
% 

[LWG issue 2089][1] outlines a problem with using `emplace`-style construction of objects. The user passes such functions a list of values, and the function forwards these values to the constructor of a particular type.

The problem is that aggregates, by definition, do not have a constructor. Therefore, `emplace`-style functions cannot be used to initialize them. However, we do have so-called "uniform initialization syntax", which can call constructors or perform aggregate initialization, as is appropriate to the type being constructed.

Unfortunately, we cannot simply declare that `emplace`-style functions will use braced-init-lists. This is because such lists have a preference for `initializer_list` constructors where available, and this preference can break existing code that relied on `emplace`-style functions to call a non-`initializer_list` constructor.

This paper evaluates the problem and current proposed solution, and presents a better solution to this issue.

# Flaws in the Proposed Solution

LWG 2089 outlines a proposed library-only solution. `allocator::construct` would perform as follows:

> Effects: `::new((void *)p) U(std::forward<Args>(args)...)` if `is_constructible<U, Args...>::value` is `true`, else `::new((void *)p) U{std::forward<Args>(args)...}`

Ignoring the fact that this solution does not solve the problem for `emplace`-like functions that do not rely on `allocator::construct` (`make_unique/shared` and the like), this has a significant flaw.

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

The question is this: is that what the user *wanted to happen*?

Well, this behavior is very incongruent with current list-initialization rules, which prefer `initializer_list` constructors over regular ones. If you did `Something{10}`, this would call the `initializer_list` constructor. Whereas calling `emplace(10)` will call the regular constructor. And yet, Something{10, 20} will have the same effect as `emplace(10, 20)`: calling the `initializer_list` constructor.

I suspect that users would find this to be surprising. A user would probably expect `emplace` for a non-aggregate to either act exactly like list-initialization or act like constructor initialization. They would not expect it to have behavior that is distinct from both language-based cases.

A more correct fix for this problem would be this:

> Effects: `::new((void *)p) U(std::forward<Args>(args)...)` if `is_aggregate<U>::value` is `false`, else `::new((void *)p) U{std::forward<Args>(args)...}`

Of course, `is_aggregate` is not currently a type trait. But the general idea is clear: such an emplace function will either call the constructor for the arguments you passed, or it will use aggregate initialization.

I submit that this aggregate-based form is more likely to give users the behavior that they expect and *desire* from `emplace`-style functions. It allows users the freedom to select exactly how they want to construct the type. If they want to call an `initializer_list` constructor, they simply spell it out at the call site: `emplace(initializer_list<int>{...})`. If they do not, then they don't spell it out.

# A Language Solution

The above outlines a purely library-based approach. But, as Ville Voutilainen outlined in N4462, it is very difficult for the user to make their own `emplace`-style functions that exhibit this behavior. It requires using `enable_if` gymnastics; even with concepts added to the core language, it still requires a lot of effort from the user.

Therefore, a language solution seems like a good way to give the user the power to handle this.

The goals of the solution are to:

1. Permit `emplace`-like forwarding functions in the standard library to initialize aggregates, without breaking existing code and without accidentally calling the wrong function.

2. Allow the user to easily write their own `emplace`-like forwarding functions, which will have identical behavior to the standard library ones.

3. Make it easy for the caller of the `emplace` function to specify whether they want to invoke `initializer_list` constructors or not. In particular, try to avoid having the user spell out `std::initializer_list<typename>` when they want to use an `initializer_list` constructor.

Surprisingly, a solution to all of these can be done that requires only one syntactic change (along with the appropriate language for elements around this change). The solution here is to provide a way to qualify a braced-init-list; qualified lists affect exactly how the list is used to initialize a type. The syntax will be as follows:

	{<special identifier>: ...}

The "special identifier" is an identifier that defines how the braced-init-list is qualified. The available qualifiers are:

* `c`: The braced-init-list is "constructor-qualified".
* `l`: The braced-init-list is "list-qualified".

Note that these are special identifiers, in accord with [lex.name]; they are not keywords.

[over.match.list] specifies how list-initialization selects constructors for a class. This section has two bullet points, which specify two groups of constructors list-initialization checks, in the given order.

An unqualified braced-init-list uses both bullet points, in the given order. A braced-init-list which is constructor-qualified will skip the first bullet point. A list-qualified braced-init-list will skip the second bullet point.

This permits us to resolve LWG 2089 as follows:

> Effects: `::new((void *)p) U{c: std::forward<Args>(args)...}`

This has identical effects to the `is_aggregate` version suggested above, but is far more concise. Of course, this would be used in any forwarding-to-constructor type function in the standard library (in-place construction on types like `any`, calls to `make_shared/unique`, etc).

The syntax is very easy to use and learn. Indeed, people may begin to default to using `{c:}` syntax when constructing all types. This would permit the general use of list initialization for all types, without having to worry about `initializer_list` constructors getting in the way. Users would only switching to `{l: }` when they explicitly need to call an `initializer_list` constructor.

As such, one could say that this feature makes "uniform initialization" truly uniform.

To resolve the other problem, the ease of passing `initializer_list`s to functions, we make one more change. One that is rather obvious, given qualified braced-init-lists.

As it currently stands, the statement `auto x = {...};` will always deduce to an `initializer_list`, based on the types provided. However, template argument deduction cannot deduce `{...}` into an `initializer_list`. This was done for good reasons; namely, that you don't know exactly what the user meant when they wrote `{...}`. So you force them to state their intent explicitly.

But since list-qualified lists are specially qualified, the person who wrote them has already *expressed* their intent. Therefore, we can decide that they can be deduced in template arguments in exactly the same way as for `auto x = {...};`. Furthermore, we should also say that constructor-qualified lists cannot be deduced as `initializer_list`s at all. As such, `auto x = {c: ...};` should fail to compile.

Therefore, users who want to pass an `initializer_list` to an `emplace` function need only do `emplace({l: ...})`.

This also means that `auto x{l: 10};` will always deduce to a single-element `initializer_list<int>`, while `auto x{10};` will become an `int` (per C++17 rules). So this uniformity is another advantage of the syntax.

## Oddities of the Syntax

This syntax does create some circumstances that a user might not expect. List-qualified lists are capable of performing initialization that does not involve `initializer_list`s. Because the only different (with regard to initializing a type) between a list-qualified initialization and unqualified initialization is the selection of constructors, list-qualified initialization is capable of initializing things in non-list-like fashions.

[dcl.list.init]/3 lists many circumstances that are checked well before a the constructor behavior that list-qualified initialization changes. Perhaps the most unexpected circumstance is paragraph 3.2: the invocation of aggregate initialization, exactly as if the braced-init-list were not qualified.

It makes perfect sense to be able to use list-qualified list-initialization on many aggregates, since aggregates can be arrays. Some struct aggregate contain arrays or have members that are like arrays. For example a 3D vector type may store its members as `x`, `y`, and `z` instead of an array, but initializing them as a list seems perfectly reasonable.

The question is whether it reasonable to ever see `{l: "a string"s, 10, std::vector<int>{10, 20, 30};}` in code? Given a struct aggregate that has members of those types, would a user expect to be able to initialize it with a list-qualified init-list?

Qualified initialization, as proposed above, would do nothing to prevent this. And while it seems semantically incorrect, we wouldn't be actually breaking any code by permitting this.

If this is deemed to be unwanted behavior, we could declare that a non-empty list-qualified list must be able to deduce an `initializer_list` of some kind (as if by `auto x = {l: ...};`), even if the list is used for aggregate initialization or initializes an `initializer_list` of a different type than the deduced one. That last one is important, as we still want non-narrowing conversions to work:

	std::vector<float> f{l: 1, 2, 3, 4};

The list `{l: 1, 2, 3, 4}` would naturally deduce to `initalizer_list<int>`. But `vector<float>` only takes an `initializer_list<float>` And we still want that code to work. So what we should do is ensure that deduction is possible, not that the deduced type is what ultimately gets used.

Then again, such a rule may be more complex than it is worth.

## Typed List Qualified Initialization

Indeed, this brings up an issue that we may be able to fix with a modification of this feature. Consider the `vector<vector<float>>` type. If you attempt to use `emplace_back({l: 1, 2, 3, 4})`, that will deduce to the wrong type of list. And `vector`'s `initializer_list` constructor takes exactly and only one type of `initializer_list`, so this will result in a compile failure.

Now, given that we're working with completely new syntax, we could also add syntax to cover this. For example, we could permit the use of `{l typename: ...}`, which would always deduce to `initializer_list<typename>`.

It is less clear exactly how this should behave. Consider this:

	std::vector<float> f{l int: 1, 2, 3, 4};
	
On the surface, this seems decidedly incoherent, as if the author didn't know what they really wanted. And yet, it could be allowed.

If the rule is that the `<typename>` will only be used in deduced contexts, then this will function just fine. If instead we say that `{l typename: ...}` will always generate a `initializer_list<typename>{...}`, then this would not work. But this would mean that `{l typename:} syntax could never be used on an aggregate, which means that even the following would not be allowed:

    float arr[] = {l float: 1.0f, 2.0f, 3.0f, 4.0f};

While one might question the repetition of using  `float` twice (rather than `auto`), consider a struct aggregate that stored an array of floats. The use of `l float` here would simply be making it clear what's being initialized.

A way to get around this would be to say that if you used a list-and-type-qualified list, and that initialization considers `initializer_list` constructors (per [over.match.list]'s bullet point 1), then the only constructors which take `initializer_list<typename>` are considered. So the above `vector<float>` example will fail to compile, while still preserving the ability of such lists to initialize aggregates.

# Acknowledgments

* Joël Lamotte: For coming up with the idea to use {:} syntax.
* Malte Skarupke: For bringing up the issue of using uniform initialization in generic code.

# References:

* LWG issue [#2089][1]
* [N4462][2]: LWG 2089, Towards more perfect forwarding, Ville Voutilainen

[1]: http://cplusplus.github.io/LWG/lwg-active.html#2089
[2]: http://wg21.link/N4462