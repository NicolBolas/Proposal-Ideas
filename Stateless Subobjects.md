% Zero sized Subobjects and Stateless Types, pre-release v5
%
% July 25, 2016

This proposal specifies the ability to declare member and base class subobjects which do not take up space in the objects they are contained within. It also allows the user to define classes which will always be used in such a fashion.

# Motivation and Scope {#motivation}

C++'s object model does not allow for the size of any complete type to be 0. Every complete object must take up a certain region of storage.

However, the object model does permit the layout of a class to allocate no storage to a particular subobject. The standard even effectively mandates this through standard layout rules: an empty, standard layout base class in a standard layout type must effectively have zero storage within the class it is a base class of. This does not affect the `sizeof` the base class subobject, as stated in [expr.sizeof]/2.

This proposal is to expand these zero sized subobjects in two ways. First, to enforce that a particular base class subobject will be zero sized. Second, and most importantly, to enforce that a particular *member* subobject will be zero sized.

One of the primary use cases for this feature is as follows.

There are many times when a user wants to use a class that the user provides, but the class may or may not have members. If it is has no members, then the class using should not grow in size simply due to the presence of an empty subobject. After all, an object which has no members does not need to take up space to do its job.

To avoid this, users will frequently use standard layout rules and the empty base optimization to allow them to have access to a zero sized subobject:

	template<typename Alloc>
	struct DataBlock : public Alloc
	{
	};

In this case, if `Alloc` is empty, then so long as `DataBlock` follows standard layout rules, the user is guaranteed that `DataBlock` is no larger than it would have been if `Alloc` were not empty. Even if it does not follow standard layout, the compiler is free to make the empty base `Alloc` zero sized.

There are problems with such a use. If `Alloc` is not empty (and `DataBlock` does not require that it is), and `Alloc` declares a virtual function, it is possible that `DataBlock<Alloc>` will accidentally override that virtual function. Or not quite override it and cause a compile error. Whereas if `Alloc` were a non-static data member, this could easily be avoided.

Equally importantly, this use of `Alloc` is simply not how a user would naturally write this class. Inheritance is supposed to model an "is a" relationship. Yet conceptually, `DataBlock` is not an `Alloc`; it *has* an `Alloc`. `DataBlock` uses inheritance, not to properly model the relationship, but simply as an optimization tool, to keep the class from growing unnecessarily.

Another problem arises if `Alloc` is *not* empty and if `DataBlock` wanted to have members of its own. In that case, `DataBlock<Alloc>` could not be a standard layout type, since both itself and the base class have NSDMs. If we had instead made `Alloc` an actual member, then whether `DataBlock` is standard layout depends on whether `Alloc` is.

The intent of this proposal is to permit, among other things, classes like `DataBlock` to declare `Alloc` as a non-static data member. And if `Alloc` is empty, then `DataBlock` can use syntax to make the `Alloc` NSDM take up no space in `DataBlock`'s layout.

This would also be recursive, so that if `Alloc` had no NSDMs, and `DataBlock<Alloc>` had no NSDMs besides the `Alloc` member, then `DataBlock<Alloc>` would also be considered an empty type. And therefore, some other type could use it in a zero sized fashion.

This proposal also pushes forth the concept of a "stateless type". This is a type which, when declared as a subobject, will always be used in a zero sized fashion. But it means something more. It represents a type which fundamentally lacks identity. The C++ object model requires every instance of an object to have a distinct address. But most empty types don't care about having a distinct address.

For the purposes of this proposal, stateless types are used as a way to get around a fundamental limitation of C++'s object model. Future proposals however can build upon this to add more functionality.

# Design {#design}

Note: The following syntax is entirely preliminary. This proposal is not wedded to the idea of introducing new keywords or somesuch. This syntax is here simply for the sake of understanding how the functionality should generally be specified. The eventual syntax could be a contextual keyword like `final` and `override`, or it could be something else entirely.

## Empty layout type

The `is_empty` trait is true for a class which has no NSDMs in itself or any of its base classes, as well as no virtual functions or inheritance. It is also restricted to class types.

However, that alone does not mean that the type can functionally be zero sized within a subobject. An empty type can have the same base class instance multiple times. The C++ object model does not allow two object instances of the same type to have the same address. So the layout of this "empty" type will have to be larger than a truly empty type. This means that the type cannot be zero sized when used as a subobject.

To be able to be zero sized, a type must be empty as well as standard-layout, since the standard-layout rules restrict types from having the same base class instance multiple types.

Thus, we define the concept of an empty layout type. An empty layout type is a class for which both `std::is_empty` and `std::is_standard_layout` are true.

The standard library will need a traits class `is_empty_layout` to detect if a type has an empty layout.

## Zero sized subobojects

A non-static data member subobject of a type can be declared to be zero sized with the following syntax:

	struct Data
	{
		zero_sized Type val;
	};
	
A base class subobject of a type can be declared to be zero sized with the following syntax:

	struct Data : public zero_sized Type
	{
		...
	};

For these declarations to be well-formed, `Type` must be an empty layout type.

`zero_sized` cannot be applied to:

* Members of a `union` class
* base classes that use `virtual` inheritance
* Member subobjects that are arrayed
* Variables that are not declared as non-static data members

The `zero_sized` property can be conditional, based on the type being declared, using this syntax:

	template<typename Alloc>
	struct DataBlock
	{
		zero_sized(auto) Alloc a;
	};

The subobject `a` will be a zero sized member only if its type is empty layouyt.

Classes which contain zero_sized subobjects, directly or indirectly, are subject to the [Unique Identity Rule](#unique_rule). Class declarations that violate the unique identity rule are il-formed.

## Zero-sized behavior {#zero_sized_behavior}

Zero sized subobjects have no effect on the layout or size of their containing object. Similarly, the rules for standard layout types ignore the presence of any zero sized subobjects (a class can be standard layout with public zero sized members and private non-zero sized members). And thus, two classes can be layout compatible even if they differ by the presence of zero sized subobjects.

Zero sized subobjects otherwise have the same effects on their containing types as regular subobjects do. For example, a zero sized subobject can cause its container to not be trivially copyable, if it has a non-trivial copy constructor (even though the object by definition has no state to copy).

The definition of "empty class" (per `std::is_empty`) must now be expanded. The rules for an empty class are:

* Non-union class
* No virtual functions
* No virtual base classes
* No non-empty base classes (zero sized base subobjects are, by definition, empty)
* No non-zero sized NSDMs

In all other ways, except where detailed below, a zero sized subobject works identically to a non-zero sized one. zero sized subobjects are initialized and destroyed just as if they were not zero sized. Zero sized subobjects undergo copy and move construction and assignment along with all other subobjects (though they will likely not be doing much). In particular, zero sized subobjects are still members/base classes of the types that contain them, so aggregate initialization of members (and bases) will still have to take them into account:

	struct foo {};

	struct aggregate
	{
	  zero_sized foo f;
	  int b;
	};

	//It may have no state, but `f` must still be initialized.
	aggregate agg = {{}, 4};

Note that the alignment of a zero sized subobject will still impact the alignment of the containing class. A `zero_sized` member declaration can include an `alignas` specifier as well.

## Stateless types

A type can be declared to have the stateless property:

	stateless class ClassName : BaseClasses
	{
		...
	};

The `stateless` keyword must be consistently associated with any forward declarations for stateless types.

	class ClassName;  //Not a stateless type.

	stateless class ClassName  //Compiler error: you told me it wasn't stateless before.
	{
	};

And vice-versa.

A stateless class definition is ill-formed if the class is not a empty layout type. However, a stateless class definition is also il-formed if it has any subobjects that are not themselves of stateless types.

The size of a stateless class is not affected by being stateless. Therefore, the size of a stateless class shall be no different from the size of any other empty layout class.

The statelessness of a type can be conditional:

	template<typename T>
	stateless(auto) struct Foo : public T {};

This will declare `Foo` to be a stateless type if all of its subobjects are also stateless. Note that, even if `stateless(auto)` is used, the class definition must not have array member subobjects.

The fact that a class is declared stateless has no effect on most uses of that type. When using the type to create an object that is not a subobject of another type, everything behaves normally.Therefore, you can heap allocate arrays of stateless classes, and they will function just like any heap allocated array of PST classes. You can declare an automatic or global variable of a stateless type, and it will behave as any PST class (note: implementations are free to optimize such objects to take up no space if possible, but they are not required to do so).

However, when a stateless class is used to declare a direct subobject of another type, that subobject declaration will implicitly be zero sized. It is perfectly valid to explicitly specify `zero_sized` on a member/base class of a stateless type as well.

Zero sized subobjects created through stateless classes are exempt from the [unique identity rule](#unique_rule).

Declaring an array of a stateless type as an NSDM is il-formed. Stateless types may not be used as members of a union.[^1]

The standard library will need a traits class to detect if a type is stateless, since this does restrict the type's use.

## Stateless anonymous types

Statelessness can be applied to anonymous types as well:

	stateless(auto) struct
	{
		Data d;
	} varName;

The `stateless` property applies to the struct declaration, not the variable. As such, `decltype(varName)` will be a stateless type if `Data` is a stateless type. Such declarations can only be made as non-static data members.

## Unique identity rule {#unique_rule}

Any class which contains (directly or indirectly) a subobject which is zero sized is subject to the unique identity rule. The goal of this rule is to ensure that each object within a struct has [unique identity](#identity) while fulfilling the layout requirements of zero-sized subobjects. This rule is as follows.

For every subobject, recursively, of a class `T`, if that subobject is declared `zero_sized`, and that subobject is not of a `stateless` type, then look through `T`'s subobjects, recursively. If there is another subobject of that type (whether zero sized or not), and that subobject is created from a different subobject *declaration*, then `T` violates the unique identity rule.

Note that "different subobject declaration" means the following:

    struct empty {};
	struct holder { zero_sized empty e; };
	
	struct data
	{
		holder h1;
		holder h2;
	};
	
`h2.e` does not cause a violation of the unique identity rule. The reason being that `h1.e` and `h2.e` both come from the same declaration: the definition of `holder`.

By contrast, these would be illegal:

	struct data2
	{
		holder h1;
		empty e2; //Same type as `h1::e`, but different declaration.
	};
	
	struct data2a
	{
		holder h1;
		zero_sized holder h2; //Same type as `h1`, but different declaration.
	};
	
Do note that the identity rule works even for derived classes and arrays of types that contain zero_sized subobjects:

	struct data3 : public holder
	{
		holder h2;
		holder hs[40];
	};
	
This works because of standard layout rules. Or rather, because `data3` is *not* standard layout. Because of that, `data3::holder` and `data::h2` must have distinct addresses. And therefore, their zero_sized subobjects will as well.

This does forbid many cases that theoretically could work. It may be possible to come up with more complex forms of this rule that allow zero sized subobjects to be used in more places. But with added complexity comes the chance to get something wrong.

### Once more with complexity

This is a more complex definition of the rule that would permit users more valid cases.

Let us define the concept of a "potentially aliasing subobject." A subobject is potentially aliasing if the address of that subobject *might* alias with the containing object itself. This requires casting a wide net, since implementations are explicitly allowed to vary. For some type `T`, the subobjects which are potentially aliasing are:

* If `T` is a union, then all of its NSDMs
* If `T` is not a union, 
	* All zero sized NSDMs
	* All (non-virtual?) base class subobjects
	* The first non-zero sized NSDM of each access class. If the first such member is an array, then only the first array element is considered

This rule applies recursively. For each subobject, its potentially aliasing subobjects are added to the list.

A type violates the unique identity rule if there is a potentially aliasing subobject declared `zero_sized`, who's type is *not* declared `stateless`, and there is another potentially aliasing subobject of the same type (whether declared `zero_sized` or not).

This permits the following case, which was illegal under the first definition:

	struct data2
	{
		holder h1;
		empty e2;
	};

This is now legal because `e2` is not potentially aliasing and therefore will not be considered. However, if it `empty` had been a base class or `e2` was declared stateless, then it would violate the rule.

Consider these:

	struct data4a : holder
	{
		holder h1;
		empty e2;
	};
	
	struct data4b : holder
	{
		empty e2;
		holder h1;
	};

Neither of these are legal. The base class `holder` has a zero sized `empty` NSDM. So whether `holder` or `empty` is first, there is a potential alias with the base class.

Note that C++'s general forbearance of aliasing with the same type prevents `data4a` from being a genuine case of aliasing. Even though `holder` is an empty class, `data4a` isn't standard layout because the base class is the same type as the first NSDM. So the standard requires that the base class and the first member have different addresses. Thus, the two zero sized members of each cannot alias. So this is a false positive.

# Design rationale

A number of restrictions on the use of zero sized subobjects come from implementation restrictions or the needs of basic C++ assumptions about types. This section goes into detail about some of the rationales for the design decisions, as well as alternatives that were discarded.

## Memory locations

C++'s object model already permits the concept of zero sized subobjects ([intro.object]/5-6). At present, these are only applied to base classes ([class]/4), and only *required* under standard layout rules ([class]/7). Such zero sized subobjects are permitted to have the same address as their containing object.

We simply want to piggy-back off of these rules. However, this my cause problems with strict aliasing rules, as we would have two unrelated types which have the same address (two of the same type with the same address is covered under the [unique identity rule](#identity)).

So we might need an alteration to [basic.lval]/10 to permit zero sized subobjects to work. It would be an added exemption to the list of exemptions there.

I do not believe that this is an implementation problem, since zero sized subobojects by their very nature have no state to access. As such, the specific address of their `this` pointer is irrelevant (again, pursuant to unique identity).

Given that the address of a zero sized subobject could be any valid address within the object, should we declare that the address is the address of its containing object? This would match with the address for zero sized base class subobjects.

Implementation-wise, the difficulty will be in changing how types are laid out. That is, modifying ABIs to allow member subobjects that have no size and enforcing EBO even when it might otherwise have not been performed.

The behavior of the *user* converting pointers/references between two unrelated zero sized subobjects should still be undefined.

## Arrays of zero sized objects

This design expressly forbids the declaration of zero sized member subobjects that are arrayed. The reason for this has to do with pointer arithmetic.

In C++, the following should be perfectly valid for any type `T`, regarldess of where the array of `T` is declared:

	T t[5];
	T *first = &t[0];
	T *last = first + 5;
	assert(sizeof(T) * 5 == last - first);

The whole point of a zero sized subobject is that it does not take up space within another type. If the array above were a member of another type, this code still ought to work. But, since `sizeof(T)` is non-zero, that means that those values have to take up space somewhere. Each pointer has to be valid.

Under non-array conditions, the pointer to a zero sized subobject could always have the same address as its container (recursively, to the last non-zero sized container). Adding 1 to any such pointer will still be a valid address; this is required because any object can be considered an array of 1 value ([expr.add]/4). If the containing class is an empty class with a size of 1, then adding one to a zero sized member's pointer is no less valid than adding 1 to the container's address.

When dealing with an array, this is not necessarily the case. If the array of zero sized objects has a count larger than 1, there is no guarantee that adding a number larger than 1 will result in a valid address for the purposes of iteration.

Several alternatives were considered:

1. Declare that `sizeof` for zero sized subobjects is zero. I am not nearly familiar enough with the C++ standard to know the level of horrors that this would unleash, but I can imagine that it's very bad. It would also make it impossible to have a distinction between zero sized subobjects and stateless types, as the `sizeof` can be applied to a pointer to the object, which cannot know if the object it points to is a zero sized subobject.

2. Declare that zero sized subobjects which are arrayed will still take up space as though they were not declared `zero sized`.

3. Forbid declaring arrays of zero sized subobjects altogether.

Option 1 is probably out due to various potential issues. #2 seems tempting, but it makes the use of `zero_sized T arr[4];` something of a lie. Earlier designs used #3, then switched to #2. But the current design goes back to #3 for an important reason.

In the current design, it is the *use* of a type which is zero sized. Because this can be applied to any empty layout class at its point of use, if a user wants to be able to use an empty layout class in a non-zero-sized way within a class (such as an array), then they simply do not declare the class to be `stateless`.

## Trivial copyability

Trivial copyability is another area that can be problematic with zero size subobjects. We explicitly forbid trivial copying into a base class subobject of another class ([basic.types]/2).

Similarly, we must explicitly forbid trivial copying into zero sized subobojects. Trivial copying *from* a zero sized subobject should be OK. Though this is only "OK" in the sense that the program is not ill-formed or that there us undefined behavior. The value copied is undefined, but that doesn't matter, since you could only ever use that value to initialize a type that is layout compatible with an PST class. Namely, another PST class. And PST classes have no value.

## offsetof

`offsetof` will obviously not be affected by zero sized subobject members of another type. However, if `offsetof` is applied to a zero sized subobject, what is the result?

If we define the pointer to a zero sized subobject to be the same as its container, then the `offsetof` should be 0. Otherwise, the `offsetof` should be undefined but within the region of storage for its container.

## Unique identity problem {#identity}

An empty type has no valid data; it has no real value representation. But under current C++ rules, it does have one important piece of state: itself.

The present rules of C++ allow a number of cases where the addresses of two object instances are equal. This involves subobjects: base classes can have the same address as the derived class. The first member subobject often has the address of the containing class. And for standard layout types, empty base classes will have the same address as the first member, which will have the same address as the containing class.

Despite all of this, there is one inviolate rule with subobject layout: for any two distinct objects of the same dynamic type `T`, they will have *different addresses*. Because of this rule, an empty type can grant itself state by using its `this` pointer as a unique identifier, with the state being stored and accessed externally to the object instance.

However, in order to make a subobject zero sized, in order for it to not affect the layout of its containing type, we have to accept that the pointer value could alias with another object. And therefore, the address of one instance of a type could alias with the address of another instance. If the type were written to use external state based on its `this` pointer (or if code using that type were written expecting it), then we would break that expectation and therefore potentially that code.

As such, blindly applying `zero_sized` to any empty layout type could have unpleasant consequences. Various alternatives were considered to solve this problem; the solution proposed here is the last one presented, and is implemented in the [design section above](#design).

### Ignore the problem

One solution considered was to just ignore the problem. The scope of the problem is such that it would only happen if:

1. The user has created an empty layout type that gains state state through its object identity.

2. The type is used as a zero sized subobject.

3. The type is used as a zero sized subobject which aliases with another instance of itself within the same object.

Users who do #1 can defend themselves against #2 simply by declaring that the type has an NSDM of type `char`. Alternatively, we can define special syntax to allow users to actively prevent a PST from being used as a stateless subobject.

### There is no problem

Another solution examined was to redefine the problem. It only occurs when two or more zero sized subobjects of the same type can alias each other within the same containing object. So we can simply declare that, when that happens, the objects are *not distinct objects*. One is an alias for the other.

As such, we have identity only of the ultimate non-zero sized containing subobject. If the user expects (given the above example) `multiple::ns1` and `multiple::ns2` to be different objects, then the user is simply wrong to think so.

This sounds like ignoring the problem, but it really is different. In the previous suggestion, it's a corner case. With this suggestion, we conceptually re-evaluate what it means to declare that a subobject is zero sized. That being zero sized means that the object shares its identity shall effectively lose its identity among any other subobjects of its own type within the same non-zero sized containing object.

That is, rather than declaring it to be an unfortunate bug, we embrace it as a feature. As such, we *actively forbid* implementations from allowing different zero sized instances at the same level from having different addresses. `multiple::ns1` *must* have the same address as `multiple::ns2`.

This also means that there is only one such object. This creates an interesting problem with object lifetime and construction/destruction calls. Indeed, any per-member compiler-generation function call.

On the one hand, if there is only one such object, then there should only be one constructor and one destructor call. But there are two member variables, and having only one call would be weird. Indeed, the two objects could be subobjects of different zero sized subobjects.

What's worse is that, if the user declares a copy constructor or assignment operator, they will have to list both objects in their member initializer. If they declare an assignment operator, they have to list both objects.

So the only logical alternative here is that you *have* to call the constructor multiple times. In which case, the object model has to change.

In order for zero sized subobjects of any kind to work, we declare that many objects of different types live in the same region of storage. In order for two zero sized subobjects of the same type to live in the same region of storage, we have to declare that, since they have no value representation, then a memory location can conceptually hold any number of zero sized subobjects of any type. Even if they are the same type.

As such, constructing one member subobject begins its lifetime. Constructing a second begins its lifetime. This leaves you with two objects of the same type in the same location of memory.

### Only stateless types can be zero sized subobjects

With this solution, we make the assumption that, if a user declares that a type is `stateless`, then they are making a strong claim about this type. Namely that losing identity is perfectly valid, that it is reasonable for two objects of the same type to have the same address and live in the same memory.

So we solve the problem by only permitting zero sized subobjects of types that are themselves declared stateless. That way, we know that the user expects such types to alias. This also means that stateless types can only have subobjects of other stateless types (lest they be able to lose identity as well).

This solves the unique identity problem by forcing the user to explicitly specify when it is OK to lose unique identity. However, it fails to solve the general problem of people using EBO as a means for having a zero sized subobject. They will continue to do so, as the set of `stateless` declared objects will always be smaller than the set of PST types that qualify for EBO.

### Explicitly forbid the problem cases

The unique identity problem arises because two zero sized subobjects of the same type exist within the same containing type in such a way that they could alias. So we can simply forbid that. This is a special case rule, very similar to the one that prevents types from being standard layout if their first NSDM is the same type as one of their empty base classes.

We declare that an object which has a zero sized subobject that can alias with another instance of that type is il-formed. This would be recursive, up the subobject containment hierarchy. That's the general idea, but the current rule takes note of the fact that a zero sized object cannot alias with other objects of its type if it is contained in the exact same object declaration.

### Not quite zero sized

Standard layout rules effectively side-step the unique identity problem by decreeing that a type which uses a base class more than once or has a base class that is the same type as its first NSDM (recursively) is not standard layout. And therefore, you no longer are guaranteed to have base classes not disturb the type's layout.

So one possible solution is to make `zero_sized` behave like standard layout: the declaration is a *request*, not a requirement. If the type cannot build a layout to fulfill that request, then the compiler is allowed to insert arbitrary space to make it work out, so that all potentially aliasing subobjects have the same size.

To make this useful however, it has to have the same reliability that standard layout provides to empty layout base classes. So there would have to be some system of rules which, if the user follows them, will guarantee that the layout will not be impacted by any `zero_sized` subobjects. If it's not reliable, people will just go back to using standard layout types with EBO.

Indeed, it could simply be standard layout rules, with some additions. That is, building a class for which zero sized subobjects could alias means that the type is not standard layout. After all, people frequently do rely on zero sized base class subobjects, even when their types are not standard layout.

The final rule of standard layout would thus become this. For the prospective type `T`:

Form a collection of subobject declaration in `T`, consisting of:

* All member declared `zero_sized`, if they are not `stateless`
* All base class subobjects, if they are not `stateless`
* The first non-zero sized NSDM (if it is an array, then the first element)

These rules are to be applied recursively for each type of these subobjects.

The type `T` is not standard layout if, within this collection, there are two or more subobjects with the same type.

Note that this rule explicitly exempts types declared `stateless`. Such types are intended to be able to alias, so their presence cannot break standard layout.

This rule ensures that it is safe to assign the memory location of zero sized subobjects to the same location of their containing type. In non-standard layout cases, implementations may have to give the empty class storage to ensure that it does not alias.

If the user wants a hard error if a type cannot satisfy the `zero_sized` requirement, they can apply `static_assert(is_standard_layout_v<T>);` wherever they wish.

This also means that we will not strictly need `zero_sized(auto)` syntax. Since `zero_sized` is a request rather than a requirement, applying it to non-empty layout types is not a compile error. It is just a request which could not be fulfilled.

While I personally dislike this solution (mainly because it makes `zero_sized` a suggestion rather than failing when it can't work), it does have the effect of having a rule that is comprehensive. The rule for standard layout types is simple and reasonable, covering existing base class cases as well as the new zero sized case. It allows stateless types to alias, while allowing the user to know when a zero sized type will be zero sized. At the same time, compilers have the freedom to add padding or even use existing padding to avoid aliasing.

### Only zero sized types can lose identity

This is effectively a combination of some of the previous ideas. The design in this paper uses this solution, as specified with the [unique identity rule](#unique_rule).

For all PST-but-not-stateless types, we apply a rule that forbids possible unique identity violations. However, if the type is declared `stateless`, then we know that the writer of the type has deliberately chosen to accept that the type lacks identity, so we permit them to be used without restriction.

The current unique identity rule is quite conservative, leaving out situations where zero sized subobjects cannot alias:

	struct empty {};
	struct alpha {zero_sized empty e;};
	struct beta {zero_sized empty e;};

	struct first
	{
		alpha a;
		alpha b;	//Legal and no aliasing
		beta c;		//Not legal but no aliasing
	};

# Potential issues

## The Unknown

One of the most difficult aspects of dealing with any system like this is that plays with C++'s fundamental object model in new and complex ways. As such, the wording for this needs to be quite precise and comprehensive, lest we introduce subtle bugs into the standard.

A good spec doctor needs to be involved with the standardization of this feature.

# Alternative Designs Explored

There has been many alternatives towards achieving the same ends. Every such alternative has to deal with 2 fundamental rules of the C++ object model:

1. Objects are defined by a region of storage.
2. Two different object instances have different regions of storage.

This current proposal is designed to avoid changes to rule #1. Stateless subobjects in this proposal still have a non-zero size. Instead, changes to rule #2 are made to permit such subobjects to be able to not disturb the layout of their container.

Other alternatives take a different approach to dealing with these rules.

## Magic standard library wrapper type

[This idea](https://groups.google.com/a/isocpp.org/d/msg/std-proposals/sAdqngl8pew/xCjHCLpuAAAJ) is focused on just the specific case of having empty classes potentially take up no space in their containing classes.

It would essentially declare a standard library type that is, in the context of this proposal, something like this:

	template<typename T>
	zero_sized(auto) struct allow_zero_size : public T {...};

It is a wrapper which, when used as an NSDM, will not take up any space in its containing object if the given type is empty. And if it is not, then it will.

The theoretical idea behind this is that it doesn't require new syntax. And in that proposal, `allow_zero_size` would not be forbidden from being in arrays or any of the other forbearances that this proposal forbids stateless types from.

This makes the behavior of such a type rather more difficult to specify. The standard would still need to have changes to allow subobojects which take up no space to work, since users have to be able to get pointers/references to them and the like. Once that work is done, exactly how you specify that a subobject takes up no space does not really matter.

## Zero sized by type declaration

An older version of this idea permitted statelessness in a form similar to what is specified here. But that design evolved from the needs of inner classes and mixins, where it was solely on an implementation of a type to be used for a specific purpose.

As such, in that design, a class was declared stateless. Once this was done, that type could only be used to declare objects as direct subobject of another type, and they would be implicitly zero sized. Since both inner classes and mixins are intended for only those uses, that was not a significant limitation.

The problem with this solution is that it does not solve the general zero sized problem outlined in the [motivation](#motivation) section.

# Changelist

## From pre-release v4:
* Renamed "possibly-stateless type" to "empty layout type".
* Stateless subobjects renamed to zero sized subobjects, to avoid confusion with stateless types. This also better reflects the standard wording.
* `zero_sized` and `stateless` declarations no longer can have general conditions. They are either forced or `(auto)`.
* Added another alternative examined in the unique identity rule case: allowing zero sized types to affect the layout of the object if the compiler cannot avoid object aliasing.

## From pre-release v3:
* Applying one possible solution for the unique identity problem.
* Adjusting design to accept the fact that `is_empty` is not enough to ensure viability for statelessness. We really need `is_empty` and `is_standard_layout`, thus provoking the need for PST.
* Because stateless types mean something much stronger, it is now expressly forbidden to have subobjects of stateless types that are not stateless types.

## From pre-release v2:
* Added details about the empty object identity problem, as well as a number of potential solutions.

## From pre-release v1:
* Clarified that stateless types act like regular empty types in all cases except when declared as direct subobjects of classes.
* Explicitly noted that the alignment of stateless subobjects will be respected by their containers.

# Standard notes:

* [intro.object]/5-6: Explains how base classes can be zero sized. Also explains that different instances of the same type must have distinct addresses.
* [basic.lval]/10: The "strict aliasing rule".
* [class]/4: States that member subobjects cannot be zero sized. But explicitly does not mention base classes.

# Acknowledgments

From the C++ Future Discussions forum:

* Avi Kivity
* A. Joël Lamotte, who's original thread on [Inline Classes](https://groups.google.com/a/isocpp.org/d/msg/std-proposals/u35GIuJECcQ/Yorc58iiBwAJ) was the seed for this idea.
* Vicente J. Botet Escriba
* Matt Calabrese
* Andrew Tomazos
* Matthew Woehlke
* Bengt Gustafsson
* Thiago Macieira, who initially identified the unique identity problem.

[^1]: This is mainly to avoid confusion. If you declare a stateless member, you expect it to be zero sized. So forbidding constructs where that is not possible is reasonable.