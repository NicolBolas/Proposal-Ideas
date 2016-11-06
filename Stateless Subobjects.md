% Stateless Subobjects and Types, pre-release v3
%
% July 13, 2016

This proposal specifies the ability to declare member and base class subobjects which do not take up space in the objects they are contained within. It also allows the user to define classes which will always be used in such a fashion.

# Motivation and Scope {#motivation}

An object is stateless, conceptually speaking, if it has no non-static data members. As such, the state of any particular instance is no different from another. Because of this, there is no conceptual reason why a stateless object needs to have a region of storage associated with it.

In C++, a class would be stateless if it has no non-static data members (NSDMs) and all of its base classes likewise are stateless as well. `is_empty_v` qualifies as a useful test, but technically other classes could qualify as well (those with virtual members/base classes).

There are many times when a user wants to use a class that the user provides, but the class may or may not be logically stateless. If it is stateless, then it would be best if the class which uses it would not grow in size simply due to the presence of the stateless subobject. After all, a conceptually stateless object does not need to take up space to do its job.

To avoid this, users will frequently use standard layout rules and the empty base optimization to allow them to have access to a stateless subobject:

	template<typename Alloc>
	struct DataBlock : public Alloc
	{
	};

In this case, if `Alloc` is empty, then so long as `DataBlock` follows standard layout rules, the user is guaranteed that `DataBlock` is no larger than it would have been if `Alloc` were not empty. Even if it does not follow standard layout, the compiler is free to optimize the empty base `Alloc` away.

There are of course problems with such a use. If `Alloc` is not empty (and `DataBlock` does not require that it is), and `Alloc` declares a virtual function, it is possible that `DataBlock<Alloc>` will accidentally override that virtual function. Or not quite override it and cause a compile error. Whereas if `Alloc` were a non-static data member, this could easily be avoided.

Equally importantly, this use of `Alloc` is simply not how a user would naturally write this class. Inheritance is supposed to model an "is a" relationship. Yet conceptually, `DataBlock` is not an `Alloc`; it *has* an `Alloc`. `DataBlock` uses inheritance, not to model a relationship, but as an optimization tool, to keep the class from growing unnecessarily.

The intent of this proposal is to permit, among other things, classes like `DataBlock` to declare `Alloc` as a non-static data member. And if `Alloc` is empty, then `DataBlock` can use syntax to make the `Alloc` NSDM take up no space in `DataBlock`'s layout.

This would also be recursive, so that if `Alloc` had no NSDMs, and `DataBlock<Alloc>` had no NSDMs besides the `Alloc` member, then `DataBlock<Alloc>` would also be considered an empty type. And therefore, some other type could use it in a stateless fashion.

# Design

We define two concepts: stateless subobjects and stateless classes. A stateless class is really just shorthand for the former; it declares that instances of the class can only be created as stateless subobjects of some other type.

Note: The following syntax is entirely preliminary. This proposal is not wedded to the idea of introducing a new keyword or somesuch. This syntax is here simply for the sake of understanding how the functionality should generally be specified. The eventual syntax could be a contextual keyword like `final` and `override`, or it could be something else.

### Stateless subobojects

A non-static data member subobject of a type can be declared stateless with the following syntax:

	struct Data
	{
		stateless Type val;
	};
	
A base class subobject of a type can be declared stateless with the following syntax:

	struct Data : public stateless Type
	{
		...
	};

In both cases, if `Type` is not an empty class, then this is a compile error. `stateless` cannot be applied to a base class which uses `virtual` inheritance.

Member subobjects that are arrayed cannot be declared with `stateless` property. Variables which are not non-static data members cannot be declared with the stateless property.

`stateless` cannot be applied to a member of a `union` class.

The stateless property can be conditional, based on a compile-time condition like `noexcept`. This is done as follows:

	template<typename Alloc>
	struct DataBlock
	{
		stateless(is_empty_v<Alloc>) Alloc a;
	};

In this case, `a` will be a stateless member only if its type is empty. If the condition is false, then it is as if the stateless property were not applied to the subobject.

To facilitate common use patterns, the `stateless(auto)` syntax for subobjects shall be equivalent to `stateless(std::is_empty_v<T>)`, where `T` is the type of the subobject being declared.

### Stateless behavior

Stateless subobjects have no effect on the layout or size of their containing object. Similarly, the rules for standard layout types ignore the presence of any stateless subobjects. And thus, two classes can be layout compatible even if they differ by the presence of stateless subobjects.

Stateless subobjects otherwise have the same effects on their containing types as regular subobjects do. For example, a stateless subobject can cause its container to not be trivially copyable, if it has a non-trivial copy constructor (even though the object by definition has no state to copy).

The definition of "empty class" (per `std::is_empty`) must now be expanded. The rules for an empty class are:

* Non-union class.
* No virtual functions.
* No virtual base classes.
* No non-empty base classes.
* No non-stateless NSDMs.

In all other ways, except where detailed below, a stateless subobject works identically to a non-stateless one. Stateless subobjects are initialized and destroyed just as if they were not stateless. Stateless subobjects undergo copy and move construction and assignment along with all other subobjects (though they will likely not be doing much). In particular, stateless subobjects are still members/base classes of the types that contain them, so aggregate initialization of members (and bases) will still have to take them into account:

	struct foo {};

	struct aggregate
	{
	  stateless foo f;
	  int b;
	};

	//It may have no state, but `f` must still be initialized.
	aggregate agg = {{}, 4};

Note that the alignment of a stateless subobject will still impact the alignment of the containing class. A `stateless` member declaration can include an `alignas` specifier as well.

### Stateless types

The stateless property can be applied to (non-union) types as well:

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

A stateless class definition is ill-formed if the class is not empty (per the expanded rules for empty classes).

The size of a stateless class is not modified by being stateless. Therefore, the size of a stateless class shall be no different from the size of any other empty class.

The statelessness of a type can be conditional, just like subobjects:

	template<typename T>
	stateless(is_empty_v<T>) struct Foo : public T {};
	
Here, `Foo` may or may not be stateless, depending on whether `T` is empty. If the constant expression evaluates to `false`, then the class is not stateless.

To make certain conditions easier, there will be the `stateless(auto)` syntax. When applied to a class declaration, `stateless(auto)` will cause the class to be stateless if the class is empty. If the class is not empty, then the class will not be stateless.

When a stateless class is used to create an object in a way that cannot have `stateless` applied to it, then the code behaves just as it would if the class did not have the `stateless` keyword. Therefore, you can heap allocate arrays of stateless classes, and they will function just like any heap allocated array of empty classes. You can declare an automatic variable of a stateless type, and it will behave as any empty class (note: implementations may optimize such objects to take up no stack space if possible, but they are not required to do so).

However, when a stateless class is used to declare a direct subobject of another type, that subobject will be implicitly stateless. It is perfectly valid to explicitly specify `stateless` on a member/base class of a stateless type as well. If the conditions on the two `stateless` properties do not agree (one resolves to true and the other false), then the program is il-formed.[^1]

Declaring an array of a stateless type as an NSDM is forbidden. Stateless types may not be used as members of a union.

The standard library does not need to have a traits class to detect if a type is stateless. A type with such a declaration can be used in any way that any type can, so `is_empty` is sufficient.

### Stateless anonymous types

Statelessness can be applied to anonymous types as well:

	stateless(auto) struct
	{
		Data d;
	} varName;

The `stateless` property applies to the struct declaration, not the variable. As such, `decltype(varName)` will be a stateless type if `Data` is a stateless type. Such declarations can only be made as non-static data members.

## Implementation and Restrictions

In a perfect world, the above would be the only restrictions on stateless types. C++ of course is never perfect.

### Memory locations

In C++ as it currently stands, every object needs to have a memory location. And two separate objects of unrelated types cannot have the *same* memory location.

Stateless subobojects effectively have to be able to break this rule. The memory location of a stateless subobject is more or less irrelevant. Since the type is stateless, there is no independent state to access. So one instance is functionally no different from another.[^2]

As such, the general idea is that a stateless subobject could have any address within the storage of the object which contains it. This means that a stateless subobject can have the same address as any other subobject in its containing object's type.

This is not really an implementation problem, since stateless subobojects by their very nature have no state to access. As such, the specific address of their `this` pointer is irrelevant. What we need, standard-wise, is to modify the rules for accessing objects and aliasing to allow a stateless subobject to have the same address as *any* other object. Because stateless objects have no value to access, we may not even need to change [basic.lval]/10.

The address of a stateless subobject can be any address within the storage of the object that contains it. The standard need not specify exactly which address.

Implementation-wise, the difficulty will be in changing how types are laid out. That is, modifying ABIs to allow member subobjects that have no size and enforcing EBO even when it might otherwise have been forbidden.

The behavior of the *user* converting pointers/references between two unrelated stateless subobjects should still be undefined. We just need rules to allow stateless subobjects to be assigned a memory location at the discretion of the compiler, which may not be unique from other unrelated objects.

### Stateless arrays

This design expressly forbids the declaration of stateless member subobjects that are arrayed. The reason for this has to do with pointer arithmetic.

In C++, the following should be perfectly valid for any type `T`, regarldess of where the array of `T` is declared:

	T t[5];
	T *first = &t[0];
	T *last = first + 5;
	assert(sizeof(T) * 5 == last - first);

The whole point of a stateless subobject is that it does not take up space within another type. If the array above were a member of another type, this code still ought to work. But, since `sizeof(T)` is non-zero, that means that those values have to take up space somewhere. Each pointer has to be valid.

Under non-array conditions, the pointer to a stateless subobject could always have the same address as its container (recursively, to the last non-stateless container). Adding 1 to any such pointer will still be a valid address; this is required because any object can be considered an array of 1 value ([expr.add]/4). If the containing class is an empty class with a size of 1, then adding one to a stateless member's pointer is no less valid than adding 1 to the container's address.

When dealing with an array, this is not necessarily the case. If the stateless array has a count larger than 1, and it is contained in an empty class, there is no guarantee that adding a number larger than 1 will result in a valid address for the purposes of iteration.

Several alternatives were considered:

1. Declare that `sizeof` for stateless subobjects is zero. I am not nearly familiar enough with the C++ standard to know the level of horrors that this would unleash, but I can imagine that it's very bad. It would also make it impossible to have a distinction between stateless subobjects and stateless types, as the `sizeof` can be applied to a pointer to the object, which cannot know if the object it points to is stateless.

2. Declare that stateless subobjects which are arrayed will still take up space as though they were not declared `stateless`.

3. Forbid declaring arrays of stateless subobjects altogether.

Option 1 is probably out due to various potential issues. #2 seems tempting, but it makes the use of `stateless T arr[4];` something of a lie. Earlier designs used #3, then switched to #2. But the current design goes back to #3 for an important reason.

In the current design, it is the *use* of a type which is stateless. Types can be declared stateless as well, but this really only means that all uses of them to declare objects will implicitly be stateless. So this restricts how the type may be used, but not the semantics of it.

Because statelessness can be applied to any empty class at its point of use, if a user wants to be able to use an empty class in a non-stateless way, then they simply do not declare the class to be stateless.

And therefore, if you declare a class to be stateless, you truly intend for it to only be used as a subobject of another type. This would be useful for a CRTP base class, for example, where it is a logical error to create an object of the class which is not a base class.

### Trivial copyability

Trivial copyability is another area that can be problematic with stateless subobjects. We explicitly forbid trivial copying into a base class subobject of another class ([basic.types]/2).

Similarly, we must explicitly forbid trivial copying into stateless subobojects. Trivial copying *from* a stateless subobject should be OK. Though this is only "OK" in the sense that the program is not ill-formed or that there us undefined behavior. The value copied is undefined, but that doesn't matter, since you could only ever use that value to initialize a type that is layout compatible with an empty class. Namely, another empty class. And empty classes have no value.

### offsetof

`offsetof` will obviously not be affected by stateless subobject members of another type. However, if `offsetof` is applied to a stateless subobject, what is the result?

I would suggest that this either be undefined or 0.

# Potential issues

## Empty object identity {#identity}

A type with no data members is empty of valid data. It has no real value representation. But under current C++ rules, it does have one important piece of state: itself.

The present rules of C++ require that every object have identity that is (almost) unique from every other object. There are two exceptions to this: a member subobject of a type may have a pointer value equal to the containing object. And base class subobject(s) may have pointer values equal to their derived classes.

However, the current rules effectively guarantee that, for any two distinct objects of the same dynamic type `T`, they will have *different addresses*. As such, it is possible for an empty type to have out-of-object state by using its `this` pointer as a unique identifier to access said state.

The current version of stateless subobjects changes things so that it breaks that rule. If a type has two stateless subobjects of the same type, the compiler may assign them the same address. And if that type expects its object to have identity, the object's functionality may be broken.

There are ways to resolve this problem.

### Explicitly forbid the problem cases

The problem arises because two stateless subobjects of the same type exist within the same containing type. So we can simply forbid that. This is a special case rule, much as we declare that types stop being standard layout if their first NSDM is the same type as one of their empty base classes.

We declare that an object which has more than one stateless subobject of the same type (whether base class or member) is il-formed. This would be recursive, up the subobject containment hierarchy. However, it stops looking if it recurses through most non-stateless subobjects.

Here are some examples to clarify things:

	stateless struct no_state{};
	struct multiple
	{
		no_state ns1;
		no_state ns2;	//Il-formed, same type as before.
	};

	struct empty		//Empty, but not implicitly stateless.
	{
		no_state ns1;
	};

	struct contain1
	{
		stateless empty e;
		no_state ns2;	//Il-formed, contains same type as stateless e.ns1
	};

	struct contain2 : stateless empty
	{
		no_state ns2;	//Il-formed, contains same type as stateless base class empty::ns1
	};

	struct works
	{
		empty e;
		no_state ns2;	//Fine. `e` is empty but *not* stateless
						//Thus `e` has identity, and so do all of its members.
	};

	struct fails : empty
	{
		no_state ns2;	//Il-formed. Base class is empty and by standard-layout rules
						//has no identity.
	};

As noted in the last example, this would also require adding rules dealing with EBO interaction with stateless subobjects. There would also need to be a "first member" for non-stateless subobjects, similar to the standard layout rule.

This would be quite complicated, but I think such rules can be hashed out.

A first-pass at such a rule would be, for each stateless subobject, check:

* Any stateless subobjects in the same object.
* Any empty-but-not-stateless base class subobjects.
	* Both of these recurse up the containment hierarchy.
* The first non-stateless member subobject.
	* Recursively check for stateless subobjects and empty-but-not-stateless base classes.
	* Recursively check the first member subobject of each type.

### Ignore the problem

We could just ignore the problem. After all, it would only be a problem in the most rare of circumstances, where:

1. The user has created an empty type that gains state state through its identity.

2. The user (or another user) uses this type as a stateless subobject.

3. The user (or another user) uses this type as a stateless subobject more than once.

I would suggest that users who do #1 can defend themselves against #2 simply by declaring that the type has an NSDM of type `char`. Alternatively, we can define special syntax to allow users to actively prevent an empty type from being used as a stateless subobject.

### There is no problem

Alternatively, we could redefine the problem. It only occurs when two or more stateless subobjects exist within the same stateless mass. So we can simply declare that, when that happens, the objects are *not distinct objects*. One is an alias for the other.

As such, we have identity only of the ultimate non-stateless containing subobject. If the user expects (given the above example) `multiple::ns1` and `multiple::ns2` to be different objects, then the user is simply wrong to think so.

This sounds like the previous suggestion, but it really is different. In the previous suggestion, it's a corner case. With this suggestion, we conceptually re-evaluate what it means for a subobject to be stateless. That being stateless means that the object shares its identity with all other stateless objects in the same non-stateless container.

That is, rather than declaring it to be an unfortunate bug, we embrace it as a feature. As such, we *actively forbid* implementations from allowing different stateless instances at the same level from having different addresses. `multiple::ns1` *must* have the same address as `multiple::ns2`.

This also means that there is only one such object. This creates an interesting problem with object lifetime and construction/destruction calls. Indeed, any per-member compiler-generation function call.

On the one hand, if there is only one such object, then there should only be one constructor and one destructor call. But there are two member variables, and having only one call would be weird. Indeed, the two objects could be subobjects of different stateless subobjects.

What's worse is that, if the user declares a copy constructor or assignment operator, they will have to list both objects in their member initializer. If they declare an assignment operator, they have to list both objects.

So the only logical alternative here is that you *have* to call the constructor multiple times. In which case, the object model has to change.

In order for stateless subobjects of any kind to work, we declare that many objects of different types live in the same region of storage. In order for two stateless subobjects of the same type to live in the same region of storage, we have to declare that, since they have no value representation, then a memory location can conceptually hold any number of stateless subobjects of any type. Even if they are the same type.

As such, constructing one member subobject begins its lifetime. Constructing a second begins its lifetime. This leaves you with two objects of the same type in the same location of memory.

### Only stateless types can lose identity

This is effectively a combination of the first and third ideas.

As currently presented here, a type declared `stateless` has very few differences from a type that is empty. The primary difference is that, if it is used to declare a subobject, then that declaration will implicitly be `stateless`.

However, we can add certain features, based on the fact that the writer of the class declared it to be `stateless`. By declaring the class as such, we can define that the user has declared that the class *itself* lacks unique identity. As such, the user has entered into a contract that makes it clear that two subobjects of the same containing object refer to the same object.

Thus, we apply the third idea, but only to types which the user has explicitly blessed as such.

For all empty-but-not-stateless types, we apply the first rule. If we declare a `stateless` subobjects for a type that is empty-but-not-stateless, compilation will fail if that type could alias with another instance of itself elsewhere within that object, recursively up the subobject hierarchy.

### Only stateless types can be stateless subojbects.

With this solution, we use the assumption implicit in the previous idea (that if you declare a type to be `stateless`, then you are saying that your code will not rely on it having unique identity). And we simply say that a subobject can only be declared stateless if its type is explicitly declared stateless.

This solves the unique identity problem by forcing the user to explicitly specify when it is OK to lose unique identity. However, it fails to solve the general problem of people using EBO as a means for having a stateless subobject. They will continue to do so, as the set of `stateless` declared objects will always be smaller than the set of empty types that qualify for EBO.

## The Unknown

One of the most difficult aspects of dealing with any form of stateless type system is that it represents something that is very new and very complicated for C++ to deal with. As such, the wording for this needs to be quite precise and comprehensive, lest we introduce subtle bugs into the standard.

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
	stateless(auto) struct allow_zero_size : public T {...};

It is a wrapper which, when used as an NSDM, will not take up any space in its containing object if the given type is empty. And if it is not, then it will.

The theoretical idea behind this is that it doesn't require new syntax. And in that proposal, `allow_zero_size` would not be forbidden from being in arrays or any of the other forbearances that this proposal forbids stateless types from.

This makes the behavior of such a type rather more difficult to specify. The standard would still need to have changes to allow subobojects which take up no space to work, since users have to be able to get pointers/references to them and the like. Once that work is done, exactly how you specify that a subobject takes up no space does not really matter.

## Stateless by type declaration

An older version of this idea permitted statelessness in a form similar to what is specified here. But that design evolved from the needs of inner classes and mixins, where it was solely on an implementation of a type to be used for a specific purpose.

As such, in that design, a class was declared stateless. Once this was done, that type could only be used to declare objects as direct subobject of another type, and they would be implicitly stateless. Since both inner classes and mixins are intended for only those uses, that was not a significant limitation.

The problem with this solution is that it does not solve the general stateless problem outlined in the [motivation](#motivation) section.

# Changelist

## From pre-release v2:

* Added details about the empty object identity problem, as well as a number of potential solutions.

## From pre-release v1:

* Clarified that stateless types act like regular empty types in all cases except when declared as direct subobjects of classes.

* Explicitly noted that the alignment of stateless subobjects will be respected by their containers.

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

[^1]: It may be reasonable to use the logical `or` between the two, instead of failing on a mis-match.

[^2]: In theory, at any rate. [There is a discussion](#identity) of a case where empty types can effectively have state.