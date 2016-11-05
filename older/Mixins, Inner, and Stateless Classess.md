% Mixin, Inner and Stateless Classes, v0.7
%
% November 3, 2015

This proposal specifies three new language features associated with classes. Two of these provide an inversion of the control of class subobjects. Mixin classes allow a (template) class to access the members of the class that derives from it, while inner classes are provided access to the members of the objects that contain them. The third feature, stateless classes, are classes which, when used as subobjects of other types, take up no space in those objects. They are useful mainly when applied to mixin or inner classes, which frequently rely on storage provided by the client class, having no non-static data members of their own.

# Concepts

## Inverted Subobject

A subobject is itself an object, but it is wholly controlled by the object in which it is contained. For the purposes of this discussion, we will consider two kinds of subobjects in C++: non-static data members (NSDMs) and base class members. An NSDM adds to the capabilities of the object to which it is a member by containment. Users talk to members, and the members implement their own data storage and APIs. Base class subobjects are similar in that they expand the capabilities of the containing class. However, they expand those capabilities directly, bringing all of their functionality into the direct interface of the derived class. Both of them have access controls which limit to whom such functionality is exposed.

With subobjects of all kinds, control flows in one direction: from the containing object to the objects it contains. Base classes can access their own members as well as any members of one of their base classes. But no base class can (directly) access members of its derived class. Similarly, NSDMs can access any NSDMs of their own, but they cannot access the interface of the object that contains them.

What this proposal suggests doing is changing this relationship. An "inverted subobject" is one where the relationship between the container and the contained objects is inverted. Inverted base subobjects can access their (direct) owning object, and inverted NSDM subobjects can access the object that they are contained within.

It turns out that both of these kinds of control inversions are in use today, either in C++ itself or in other languages. While conceptually these are just variations of the same thing, these features are often implemented as two distinct features with two distinct names.

Java provides the name for an NSDM that can access its owning class: inner class. The other concept's name is defined by the effects of the concept. If a base class can access the derived class's members, then it can add functionality to that class which integrates seamlessly with any particular implementation that provides the appropriate interface. Let us refer to this as a mixin[^1].

Note that mixins effectively allow a form of static polymorphism. The normal way to invert relationships in a hierarchy is to use `virtual` functions. The derived class implements the virtual interface declared by the base class. But this can be inefficient, due to the overhead of virtual calls.

Mixins effectively allow the same inversion in a static way. Not unsurprisingly, mixins in C++ are usually implemented using the static polymorphism tool of C++: templates.

# Uses

## Stateless Classes

Stateless classes by themselves have limited utility, but they are hardly useless. For example, iterator categories are tested currently via the presence of certain base class tags. These tags have no state besides the fact that they exist. These and similar types are reasonable use cases for statelessness without inverted classes.

## Mixins

The uses for mixins are legion; they are a general-purpose tool for adding data/functions to a class from outside of the class itself. That basic feature can be used for things like C# partial classes, where some data and/or interface is generated from a tool. This can be done today, but full mixin support allows the generated class to interact with the main class as though they were one class.

Mixins are used liberally within the range TS proposal, emulated as best as possible with C++. Providing full language support for them would be a boon to both the implementation of that library and users who want to use its facade and view mixins.

The ability to make mixins stateless where appropriate is the icing on the cake, as it allows the containing class to provide data storage for the mixin's interface. The range TS mixins are good candidates for this optimization, as they usually have no members.

The standard already has a few examples of mixin-like classes, which could be improved with language support. The class `enable_shared_from_this` is a mixin with its own storage; it effectively stores a `weak_ptr` (or the guts of one) in the class.

If it were a true mixin, it could be augmented with a SFINAE test of the derived type for a certain interface. If the derived class provides that interface, then it would refrain from storing data itself; it would instead store the `weak_ptr` through the derived class's interface. This is important because classes that use `enable_shared_from_this` cannot be standard layout. Whereas with true mixin support, the class could be empty or stateless, allowing the derived class to retain its layout compatibility.

Using the derived class's storage would also allow the derived class to access the `weak_ptr` directly. This would avoid the current method of doing `weak_ptr<T>(shared_from_this())`, which involves a needless quantity of atomic operations.

And this control inversion comes without any virtual overhead, which is the primary reason why `enable_shared_from_this` did not provide a virtual interface for such storage.

## Properties {#uses_properties}

Inner classes also have many uses, though they are not as immediately apparent.

Full disclosure: I am personally not a big fan of properties. However, I also know that this is a thing people want, and the thread/proposals that led to me developing this one were primarily about getting properties into the language somehow. Obviously, this feature has evolved since then.

Stateless inner classes make it possible for users to implement properties with zero overhead. Prior property implementations also seemed really hackey; they would require things like lambdas as initializers to some struct, or the property itself would have to hold the value (thus making it impossible for properties to represent things that were not values). With stateless inner classes, the property implementation has full access to most of the powers of the C++ object model, without a bunch of boilerplate to access the data.

They can overload `operator=` to perform assignment as the "setter" operation. They can overload `operator Typename` for implicit conversions as the "getter" operation. If the underlying value is of an appropriate type, they can even overload math assignment operators like `operator+=` and such. In effect, the property object can be a full proxy for the real thing, providing access, modification, or both, in all of the ways that C++ allows. And all without taking up added space.

Because there is a lot of boilerplate here to make a good property that plays by C++'s rules, this boilerplate can almost certainly be abstracted out... into a stateless *mixin*. The inner class property itself would be responsible only for providing a `get` function and a `set` function. The mixin is what would do all of the work for operator overloading and such.

Template polymorphism can be used here as well to improve the interface. For example, if the return value of `get` is a pointer (or a type that implements `operator->`), the property can synthesize an appropriate `operator->` overload. If no `set` method is provided, then the property does not synthesize assignment operators. If no `get` method is provided, the mixin does not allow data access. But more complex rigging can be accomplished as well.

If there is a non-const `get` overload that returns a reference, and there is no `set` method, then the mixin assumes that `get() = expr` is an acceptable means of changing the value. So its overloaded assignment operators will be synthesized by calling the non-const `get`. Furthermore, a non-const `get` allows the system to synthesize the `operator+=` and similar operators without having a `get`/`set` round-trip.

And thus, a property definition in a class would look like this:

	//Mixin usage with anonymous classes.
	stateless inner class : property<mixin>
	{
		//Private members are OK; `property` is implicitly a friend.
		
		const float &get() const {return value;}
		
		//non-const reference means `property` will use this to set as well.
		float &get() {return value;}
	} property_name;

The mixin does all of the complex work that synthesizes the appropriate operator overloads. The prefix part could even be wrapped up in a macro.

## Arbitrary Interface Views

Properties are really just a specific instance of a more general use case for stateless inner classes: providing different views of an object without creating new proxy objects.

Consider a four element floating-point vector, common in many 3D math libraries. The OpenGL Shading Language has such a data type as a basic, built-in type: `vec4`. It allows you to perform two kinds of swizzle operations on them. Swizzle extraction is used for accessing the elements of the vector as vectors of a different type. `vec.xyz` for example creates a 3D vector from the components of vector. This can be done very arbitrarily: `vec.yzx`, `vec.xxyx`, `vec.wxzw`, etc.

The other form of swizzling is for assignment operations. You can perform `vec.wzyx = vec.xyzw;`, which flips the order of the values in the vector. You can set only some of the elements with `vec.zx = ...;`. You can even do `vec.zx += `, applying assignment operators to swizzled vectors.

The only difference between assignment swizzling and regular swizzling is that you cannot repeat an element in the vector.

To do this in C++ is possible, so long as you write it as a function call. Calling `vec.zwxy()` would return a new `vec4` type that did the swizzling for you. That would be fine, until we want to start using hardware support for such objects.

Many CPUs have explicit support for 2-element or 4-element floating-point vectors. They have hardware support for many swizzling operations. The type of what is stored in them isn't `float[4]`; it's usually a compiler-based intrinsic type like `__float128` or somesuch.

Such hardware has built-in support for performing swizzle operations. This means more than simply creating a swizzled version of a `__float128` from another such value. It usually means that swizzling can be a fundamental part of an assignment operation.

Stateless inner classes have the potential to implement whatever is needed to make swizzling work, both as output and as input. They would allow writing something that looks like shader code: `some_vec.zyx = vec_a.xyx + vec_b.zzz`.

One could emulate this by having `vec3::xyz()` return an object which can be converted to a vector. But this opens up a number of problems.

This object would have to store a reference to the original object. That creates lifetime issues, particularly with temporaries:

    vec3 vec_a{...};
    vec3 vec_b = vec_a.zyx(); //OK, implicit conversion helps.
    auto val = (vec_a + vec_b).zyx(); //Oops: reference to now destroyed temporary.

If `zyx` were actually a stateless inner class member, two things would happen. First, `auto val` would fail to compile, since you cannot create an inner class instance outside of the class it is an inner class of. So you would have to change it to `auto &val`. That would also fail to compile, since you cannot store a non-`const` lvalue reference to a member of a temporary. So you would have to use `auto &&val` or `const auto &val`.

This also helps ensure that you cannot do things like return such a proxy view by value.

Second, because your `auto &&val` is a reference to a member of the temporary, the lifetime of the whole temporary is extended. So this has no lifetime issues:

    auto &&val = (vec_a + vec_b).zyx;

The temporary will continue to exist so long as `val` does.

Another benefit is that there is zero overhead to using these inner class views of an object. With functions that return proxy objects, you have to create a class instance that holds a reference to the real type. And while compilers may optimize such things away, they are still there as part of the language.

Proxy iterators are a good place to see where this overhead could be non-trivial. Particularly in proxy output iterators, every time you do `*it++ = ...`, you call the proxy's constructor, which copies a pointer/reference to whatever is needed into the proxy object. It gets used once to do the copy, then destroyed.

With inner classes, the proxy type can be a stateless inner class of the iterator type itself. So `operator*` returns a reference to the inner class. The inner class's `operator=` will store the value, possibly updating the position of the iterator. And this neatly avoids having to create and destroy an object every time insertion happens.

Aggressive compiler optimization may remove these concerns, but inner classes ensure that they don't exist, even in debug builds.

The use of interface views also makes it possible to provide different ways to view the same data. An example used to illustrate this was a class that stores colors. The class would internally store the color in the sRGB colorspace, but through a stateless inner class interface, you could modify the value in the HSV colorspace.

Without inner classes, this would require a bunch of explicit calls to conversion functions: `color.set_hsv(color.get_hsv().set_hue(new_hue))`. With stateless inner classes, it requires `color.hsv.hue = new_hue`. This is much cleaner and clearer to the user. And probably more efficient.

One could even allow `color.hsv.hue += new_hue`. Without the inner class, a similar operation would have either required an explicit `add_hue` function or the user would need to do this:

	auto &&hsv = color.get_hsv(); //Avoid double get call.
	color.set_hsv(hsv.set_hue(hsv.get_hue() + new_hue));

This takes up two lines now. If you accept the double get, you get the gargantuan expression:

	color.set_hsv(color.get_hsv().set_hue(color.get_hsv.get_hue() + new_hue)

And remember: all of these inner classes can be *stateless*. All they do is enhance the class's interface; they impose no overhead on their containing classes.

Not every set of types with different representations should use such views. For example, [Howard Hinnant's Date/Time library](https://github.com/HowardHinnant/date) provides `day_point` and `year_month_day`, which are two versions of the same data. I would not suggest giving `day_point` a view that provides `year_month_day`'s interface; they should be two separate types, due to the conversion overhead.

Though it could still have property-like objects for setting/getting dates in different formats.

# Current Implementations

Inner classes and mixins can be implemented in the language as it currently stands. The question is what kind of gymnastics are required to implement them, as well as the ability to compose them from different library implementations. And of course how easy it is to get the implementation wrong.

## Mixin

For our purposes, mixin implementations should allow:

1. The mixin class to access the members of the class it "mixes" into. How much access is a question, but even just public access would be functional.

2. The class being mixed into can access the members of the mixin. Again, just public members would be something.

3. Users of the class being mixed into should be able to access public members of either class through an instance of the containing class.

4. Multiple mixins can be applied to the same class (to the degree that their interfaces do not conflict). And separate mixins into the same type can effectively communicate with one another.

5. Mixin composition. That is, building a new mixin which is the sum of itself with a second mixin.

The rules of C++ do not allow a concrete class to be able to call functions it does not know about. And since we want mixins to be arbitrarily compose-able, a mixin cannot know what it is derived from. The obvious solution here is a template of some kind.

There are two general strategies for building mixins in C++. One relies on explicitly inverting the class hierarchy, while the other involves using a regular hierarchy and clever casting tricks. They each have their drawbacks.

### Hierarchy Inversion

In this case, mixins are implemented by having the mixin be the *derived* class. Each such mixin looks something like this:

	template<typename Base>
	struct mixin : public Base
	{
		//Add members here.
	};

Users in this case do not use the `Base` class directly. They use the fully composed mixin type, which is built from a nested template sequence, usually hidden in an alias:

	using MyClass = mixin1<mixin2<mixin3<Base>>>;

This allows mixins to have access to `Base` through a template interface. Concepts can be used to specify the particular interface needed.

The downsides of this approach are many. It does not fulfill #2, as the base class cannot directly communicate with any mixins that it uses. This is an acceptable limitation if the class's main functionality is built from mixins. But if a mixin is being used to extend the class's functionality, this hierarchy inversion becomes untenable.

It also makes it impossible to have constructs like private mixins, where the mixin provides some facility that is merely used by the `Base` class without being exposed to external code. Yet this it still needs to be a mixin, since that functionality requires the inverted relationship. So these mixins are only useful for public interfaces.

It also makes #3 difficult, as users must name the entire mixin chain when naming the type. Obviously a typedef makes this work just fine, but it does make the ultimate typename rather cumbersome. It also means that `mixin1<mixin2<Base>>` is a fundamentally different type from `mixin2<mixin1<Base>>`, despite being functionally identical.

On the plus side, it makes it easy for two mixins to communicate effectively. Granted, the communication can only proceed in one direction. But mixin composition works.

Another positive of this approach is that it's very simple and difficult to get wrong. It relies on normal inheritance relationships in C++. Inversion is achieved simply by deriving in the opposite direction.

### CRTP

The curiously reoccurring template pattern (CRTP) can be used for a more featureful mixin. The way this works is simple. The mixin is a template, which is passed the *derived* class as a template argument. Thus, the derived class inherits the interface of the mixin. But because the mixin knows its derived class type, it is able to perform a `static_cast` of its `this` pointer to that type. And thereby access the derived class's members.

Here is an example implementation of this:

	template<typename derived>
	class mixin
	{
	public:
		void caller() const
		{
			auto _this = get_this();
			_this->some_function();
		}

	private:
		derived *get_this() {return static_cast<const derived *>(this);}
		const derived *get_this() const { return static_cast<const derived *>(this); }
	};

	class destination : public mixin<destination>
	{
	public:
		void some_function() {...}
	};

This kind of mixin satisfies all of the requirements. There is full cross-talk; the derived class can access the mixin and vice-versa. Though they can only access the public members of each, that is an acceptable limitation. Users of the derived class can access the mixin's public members (assuming public inheritance) as though they were members of the derived class. And thanks to multiple inheritance, it is easy for one type to use multiple mixins. Indeed, mixins can be composed with one another.

The most obvious problem is the inconvenience. The need to convert the `this` pointer manually is a particular annoyance. But we don't necessarily add language features for simple convenience. So instead, let us look at where this pattern breaks down.

Mixin composition works by passing the derived class through to the base class. That is, if you have a mixin `A`, and you want to build a combined mixin `B` out of it, you do this:

	template<typename derived>
	class A {...};

	template<typename derived>
	class B : public A<derived> {...};

And therefore, if `B` implements part of the interface that `A` requires, it will pick up that interface through its access to `derived` (since `derived` is derived from `B`).

However, the inversion of control here is not truly inverted. If `derived` *also* implements the interface from `A`, then `derived`'s version is the one that `A` will call. This works as expected for inheritance: the most derived version wins.

That is not inverting the control relationship. If we wanted true inversion, then the least derived version should win. So if both `B` and `derived` implement a part of `A`'s interface, then `A` ought to call `B`, not `derived`.

There is no way to achieve this in C++ as it stands. Or at least, not without other consequences. If you tried this:

	template<typename derived>
	class B1 : public A<B1<derived>> {...};

Then `B1` is required to implement the *entire* interface that `A` expects. So if `B1` wants to pass some of that interface through, it must do so explicitly.

This particular form of derivation is useful, but it means something fundamentally different from `B`. `B1` is saying that it is using a mixin, but it is not *composing* itself from that mixin. That is, `B1` is a mixin that just so happens to be using a mixin in its implementation. Whereas `B` is intended to be full mixin composition: the interace `B` exposes is a combination of its own requirements and those of `A`.

The CRTP mixin implementation also has no way to check for mistakes. Here are some things you have to do with such mixins that are not (directly) checked for:

* Use the correct type when deriving from them. The compiler will catch the error in the implementation, but only because it's instantiating the template and can't find the interface it expects. So the compilation error will be far from the actual problem.
* Not use virtual inheritance. That breaks the `static_cast`. The compiler can't check for that one.
* Never declare objects of mixin types directly. You cannot write this declaration: `mixin<SomeType> a;`. The compiler won't catch that either; the mixin will assume it is derived from `SomeType`. And even if `SomeType` does derive from it, it also assumes that any base class instance is also a derived class instance. Obviously, this declaration violates that expectation, leading to odd runtime failures as `static_cast` goes awry.

So the CRTP implementation, while providing nearly all of the behavior we want, is somewhat fragile.

There are a few limitations that apply to both CRTP and hierarchy inversion.

It is impossible to use either method with anonymous classes. Both methods rely on the user being able to type the name of the class. This is not a particularly problematic limitation, since C++ does not even allow such classes to declare constructors/destructors.

A more burdensome problem has to do with the size of the resulting object. Many mixins have no NSDMs. One of the main purposes of mixins is to allow the contained object to provide the storage for some data, or simply to augment the interface of the contained type.

Despite having no data, such objects can impact the size of the resulting object. If the contained type was standard layout, then empty base optimization takes care of the issue. But if it was not (perhaps the containing object has virtual functions, or some other mixin brings in its own members), then compilers are free to not optimize empty bases. And therefore, you cannot rely on such optimizations. If class size is of importance to you, every mixin potentially could be making your class bigger.

Needlessly.

## Inner Class

The concept of Java-style inner classes needs some explanation, as it is very unlike normal C++ classes.

In C++, if you declare a class within another class definition (nested classes), the nested class has few connections to the class it is declared within. It does have access to all members that the outer class has access to, but it is otherwise no different from any other class. In C++, nested classes exist mainly for scoping of the class's name.

In Java, static nested classes are much like regular C++ nested class: they are friends of the other class and have their names scoped, but that's about it. But a non-static nested class is very special; it is an inner class.

A Java inner class instance must be directly linked to some instance of the outer class. Java even provides special syntax for allocating such instances from an existing outer class instance. An inner class can always implicitly convert its `this` pointer to a pointer to the instance that their current object instance is linked to.

So if there is an instance `a` of a class `A`, and it has an inner class member `b`, then any non-static member function of `b` could call non-static member functions in `a`. Not just in the class, but in the *specific instance* `a` which holds this particular `b` instance object. Java does not even limit this to instances that are directly members of the outer class; in this example, `b` does not have to be a member field of class `A`.

One can pass Java inner class members around to other functions, which can call functions on those inner class members. The members that get called can themselves still access `a`, even though the function that called them was passed `b`.

In terms of implementation, what is needed is a way to transform a pointer to `b` into a pointer to its owning object of type `A`. In Java, this is done by literally sticking a hidden pointer into the inner class type; garbage collection handles lifetime issues. This also allows Java inner classes to be declared outside of the class they are nested within. The code allocating it still has to have an object of that type, but the variable it gets stored within is not directly a part of the owning class.

Without generalized garbage collection, C++ can't really do that. As such, implementing inner classes in C++ is difficult. But this is not impossible. It does require quite a bit of help, though.

The general idea is to first restrict inner classes to being NSDMs of their containing instances. Then, have the containing class's constructors pass a `this` to every inner class member as part of those members' construction. Each inner class member stores a copy of that pointer. When an inner class member is being copied, care must be taken not to copy the `this` member.

These kinds of implementations are functional. But unlike mixins, it's a lot harder to ignore the difficulties of implementation. *Every* constructor of the owning class has a bit of boilerplate in it, making the whole process quite fragile. Every constructor of the inner class has to store the `this` pointer correctly. And because copying it cannot copy the `this` pointer, it cannot not be trivially copyable.

This also means that the class storing a member of the inner class type cannot be trivially copied either.

Composition of inner classes (inner classes of inner classes) is problematic. Each inner class gets to access its direct outer class, but they can access no outer classes beyond that without even more special coding. Though at least in this case, because inner classes are concrete classes, it is at least possible for the inner class to read the outer inner class's pointer to *its* outer class. But that requires that the user manually determine which function in which class in the inverted hierarchy needs to be called and then use the correct pointer to make that call.

Oh, and while empty base optimization is sometimes possible for mixins, there is no allowance in C++ for empty *member* optimizations. So every empty inner class member bloats the class. Though the fact that "empty" inner classes still need that `this` pointer means that they're not empty in *practice*, just in concept.

## Stateless Optimization

There are many uses of these inverted subobjects. However, quite a few of them involve the inverted class having no NSDMs at all. The state these objects manipulate or access is provided by the owning class, with the mixin/inner class merely providing an interface.

The problem is that, by the rules of C++, all members do take up space in the final class. While empty mixins are required to take up no space when used as bases of standard layout types, there is no such optimization for non-static data members. And even with base classes, if the owning class cannot be standard layout, then removing this space is entirely optional.

This issue has been one of the main reasons why people do not find some library implementations of properties attractive (implemented as a very limited form of inner classes). Making the interface slightly nicer to use is usually not worth the overhead. Therefore, finding a way to remove this overhead would be ideal, and would for many people be considered a necessary first step.

As such, this proposal includes not only the two inverted subobject types, but a third feature that both of the other two can make significant use of. This is the ability to declare that an empty class shall not take up room when declared as a subobject of another type.

# Design

Now that we have discussed what these features are, their limitations with current implementations, and what we could gain from putting them in the language, let's talk about how to add them into the C++ language. Here are the goals for the overall design of these features:

* Inverted and stateless classes should be restricted in functionality only where their design requires it, or to the degree which is implementable. For example, inner class instances are directly associated with an instance of their owning class. As such, it makes no sense to be able to declare automatic variables of them or put them on the heap, so restricting them to being declared as members of that class is acceptable. However, it should be perfectly reasonable to pass around references or pointers to such objects.

* If a class contains inverted subobjects, the class should be impacted by this as little as possible, except where obvious. In particular, we should try as much as possible to avoid complicating standard layout and trivial copy-ability rules, unless it is absolutely necessary to the implementation.

A word about syntax. This proposal effectively adds a number of new keywords to the language. This represents what I feel is the "perfect" syntax for these features, the best possible. However, this should be considered placeholder syntax, not the final form. For the time being, focus instead on whether we want the functionality and how well it is defined, not the syntax used to state it.

Overall, this document should be looked on as a start point and a rational, not the final word. There are many elements of this design which can be debated and adjusted.

## Stateless Objects and Classes

There are two concepts here: stateless subobjects and stateless classes. A stateless class is really just shorthand for the former; it declares that the class can only be used as a stateless subobject.

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

The definition of "empty class" must now be expanded to include classes which have stateless subobjects, both members and base classes.

The stateless property can be applied to types as well:

	stateless class ClassName : BaseClasses
	{
		...
	};

The `stateless` keyword must be associated with any forward declarations for stateless types.

	class ClassName;  //Not a stateless type.

	stateless class ClassName  //Compiler error: you told me it wasn't stateless before.
	{
	};

And vice-versa.

A stateless class definition is ill-formed if it is not empty (per the expanded rules for empty classes).

The size of a stateless class is not modified by being stateless. Therefore, the size of a stateless class shall be no different from the size of any other empty class.

The standard library should have an appropriate traits class to detect if a type is declared stateless. Though in most cases, `is_empty` will be sufficient.

The statelessness of a subobject can be conditional, much like `noexcept` can have a conditional expression:

	template<typename Alloc>
	class Bar
	{
	private:
		stateless(is_empty_v<Alloc>) Alloc allocator;
	};

	This means that `Bar` (given no other NSDM declarations) will be an empty class if `Alloc` is empty.

The statelessness of a class can also be conditional:

	template<typename T>
	stateless(is_empty_v<T>) struct Foo : public T {};
	
Here, `Foo` may or may not be stateless, depending on whether `T` is empty. If the constant expression evaluates to `false`, then the class is not considered stateless. However, a class which has a `stateless` declaration, even if the expression evaluates to `false`, is a *conditionally stateless* class.

A class which is conditionally stateless can only be used to declare objects in places where objects of a stateless type are permitted. This means that conditionally stateless classes can only be used to declare subobjects of another type. They cannot be used to declare automatic or global variables, nor can they be allocated with any form of `new` expression (directly, of course; you can use `new` on a type that contains a stateless subobject).

An object declaration of a stateless class (member or base class) implicitly applies the `stateless` keyword, along with its `stateless` expression. It is perfectly valid to specify `stateless` on a member/base class of a stateless type. If both the type and the declaration `stateless` have an expression, then the subobject will only be stateless if *both* expressions evaluate to `true`.

Stateless subobjects have no effect on the layout of the containing object. A stateless NSDM does not alter the layout of the class of which it is a member. A stateless base subobject (which must not be `virtual`, as previously discussed) does not alter the storage or layout of the derived class. Thus, the layout/size of a type is only affected by its non-stateless/virtual subobjects.

Similarly, the rules for standard layout types ignore the presence of any stateless subobjects. And thus, two classes can be layout compatible even if they differ by the presence of stateless subobjects.

Stateless subobjects otherwise have the same effects on their containing types as regular subobjects do. For example, a stateless subobject can cause its container to not be trivially copyable, if it has a non-trivial copy constructor (even though the object by definition has no state to copy).

In all other ways (except where discussed below), stateless types can be used exactly like regular types.

### Implementation and Restrictions

In a perfect world, the above would be the only restrictions on stateless types. C++ of course is never perfect.

#### Memory locations

In C++ as it currently stands, every object needs to have a memory location. And two unrelated types cannot have the *same* memory location.

Stateless subobojects effectively have to be able to break this rule. The memory location of a stateless subobject is more or less irrelevant. Since the type is stateless, there is no independent state to access.

As such, the general idea is that a stateless subobject can have any address within the storage of the object which contains it. This means that a stateless subobject can have the same address as any other subobject in its containing object's type.

This is not really an implementation problem, since stateless subobojects by their very nature have no state to access. As such, the specific nature of their `this` pointer is irrelevant. So we need to modify the rules for accessing objects and aliasing to allow a stateless subobject to have the same address as any other object. Of course, because stateless objects have no value, we may not even need to change [basic.lval]/10.

Implementation-wise, the difficulty will be in changing how such types are laid out. Permitting a member subobject which has no size, and forcing EBO.

The behavior of the *user* converting pointers/references between two unrelated stateless subobjects should still be undefined. We just need rules to allow stateless subobjects to be assigned a memory location at the discretion of the compiler, which may not be unique from other unrelated objects.

#### Stateless arrays

This design expressly forbids the declaration of member subobjects that are arrayed. It also forbids the declaration of even conditionally stateless type as an array. The reason for that has to do with pointer arithmetic.

In C++, the following should be perfectly valid for any type:

	T t[5];
	T *first = &t[0];
	T *last = first + 5;
	assert(sizeof(T) * 5 == last - first);

The whole point of a stateless subobject is that it does not take up space within another type. If the array above were a member of another type, this code still ought to work. But, since `sizeof(T)` is non-zero, that means that those values have to take up space somewhere. Each pointer has to be valid.

Under non-array conditions, the pointer to a stateless subobject could always have the same address as its container (recursively, to the last non-stateless container). Adding 1 to any such pointer will still be a valid address; this is required because any object can be considered an array of 1 value ([expr.add]/4). If the containing class is an empty class with a size of 1, then adding one to a stateless member's pointer is no different from adding 1 to its own address.

When dealing with an array, this is not necessarily the case. If the stateless array has a count larger than 1, and it is contained in an empty class, there is no guarantee that adding a number larger than 1 will result in a valid address for the purposes of iteration.

Several alternatives were considered:

1. Declare that `sizeof` for stateless subobjects is zero. I am not nearly familiar enough with the C++ standard to know the level of horrors that this would unleash, but I can imagine that it's very bad.

2. Declare that stateless subobjects will only not take up space if they are direct subobjects of a class type. This is as opposed to being subobjects of arrays that themselves are subobjects of a class.

3. Forbid declaring arrays of stateless subobjects altogether.

Option 1 is probably out due to various potential issues. #2 seems tempting, but it makes the use of `stateless` something of a lie. Earlier designs used #2, but #3 works better for the current one.

In the current design, it is the *use* of a type which is stateless. Types can be declared stateless as well, but this really only means that all uses of them to declare objects will implicitly be stateless. So this restricts how the type may be used.

Because statelessness can be applied to any empty class at its point of use, if a user wants to be able to use an empty class in a non-stateless way, then they simply do not declare the class to be stateless.

And therefore, if you declare a class to be stateless, you truly intend for it to only be used as a subobject of another type.

#### Trivial copyability

Trivial copyability is another area that can be problematic with stateless subobjects. We explicitly forbid trivial copying into a base class subobject of another class ([basic.types]/2).

Similarly, we must explicitly forbid trivial copying into stateless subobojects. Trivial copying from a stateless subobject should be OK.

#### offsetof

`offsetof` will obviously not be affected by stateless subobject members of another type. However, if `offsetof` is applied to a stateless subobject, what is the result?

I would suggest that this either be undefined or 0.

## Inverted Class Members

Inverted classes of either kind can access their owning instance members from their member functions. But not from within *all* of their member functions.

Inverted class constructors and destructors explicitly do not have access to their owning class members. This is for the same reasons why virtual functions don't work in them. Member subobjects are constructed before their containing objects, so their owners don't exist yet. And member subobjects are destroyed after the destruction of their containing objects, so they would be accessing objects that have stopped existed.

But in all other cases, inverted class non-static member functions can access their owning class members.

## Inverted Class Scoping

The current design allows inner classes and mixins to access *everything* from their containers, with such access propagating through inner classes and mixins up to the first non-inner class/mixin type (or with a certain form of inheritance for mixins). This includes private members.

This seems like it breaks encapsulation a lot. However, I would argue that it does not truly do so.

Inner classes can only be declared within another class declaration. They can only be used as NSDMs of that class (or a one derived from it). Inner classes are as much a part of that class as one of its member functions. Thus, they should have the same rights as one of its member functions.

Mixins are a bit more troublesome in theory. A mixin is not contained within a class definition. So giving it private access makes it seem like users can gain access to a type whenever they want.

However, the mixin syntax proposed below requires that the mixin is added to the class at declaration time. And therefore, the implementer of the class is the one who decides which mixins to use and which not to use. So users are unable to inject a mixin into a class that does not deliberately choose to use it. Using a mixin is thus like declaring that a class is a friend: the implementer is still the one in control of who gets access.

If this is considered a deal-breaker, we can restrict mixins to only being able to access the public members of those they are mixed into. Alternatively, we can add syntax to specify what a particular mixin has access to. The class deriving from the mixin would probably be the best to determine this.

## Inner Classes

An inner class is a nested class that is declared as follows:

	inner class Name
	{
		...
	};

Inner classes may only be declared as nested classes. The class an inner class is declared within is the direct owning class of that inner class. The direct owning class of an inner class may itself be an inner class.

As with normal class definitions, `Name` is optional (anonymous inner classes are quite useful). The `inner` keyword is part of the class's declaration, so if it is forward declared, it must be consistently declared with this keyword.

An inner class may or may not be stateless. If present, the `stateless` keyword can go before or after the `inner` keyword. An inner class not explicitly declared `stateless` is a stateful inner class.

An inner class can be derived from non-inner classes normally.

An inner class may be derived from another inner class, but only if the direct owning class of both is the same, or the direct owning class of the derived inner class is itself derived (non-virtually) from the direct owning class of the base inner class. From the derived inner class, it should be possible to go to its direct owner, then walk down the class hierarchy (non-virtually) to the class that is the direct owning class of the base inner class.

A non-inner class may not be derived from an inner class type.

Inner classes may have members and other definitions as befitting a regular class.

The standard library should have a template metafunction (and variable) for testing if a type is an inner class type.

### Type Usage

Variables of inner class types can only be declared as NSDM's of a class. And the class which they are declared within must be either:

* The direct owning class of the inner class.
* A class non-virtually inherited from the direct owning class of the inner class. The derived class in question must have access to the type (so private/protected inheritance can sever access, even without virtual inheritance).[^2]

    Note that this means that the class which contains an instance of an inner class need not be exactly its owner. However, the container does need to be non-virtually derived from the owner.

Inner class types cannot be used as the type in `new` expressions. Temporaries cannot be created of inner class types. The only way to create an object of an inner class type is as an NSDM subobject of a type or a base class subobject of another inner class (as a consequence, it is never legal to explicitly call the destructor of an inner class).

Pointers and references to inner class types work as normal for any type.

Inner classes cannot be aggregated into arrays. The reason for this restriction [is discussed later](#inner_impl), but it is primarily due to implementation limitations.

How inner classes affect layout compatibility, trivial copyability, and various other aspects is very important. These decisions also factor heavily into how compilers can implement them. The following is the list of effects that are implementable when combined with the above restriction on inner class arrays. The [later discussion of implementation strategies](#inner_impl) allows for different usage restrictions and different effects.

The specific interactions with the containing class are:

* Inner class members can be copied/moved exactly as any other type. Specifically, if a stateful inner class type is trivially copyable, it should be legal to do a `memcpy(dest, src, sizeof(T))` from it to another instance of that type, even if the destination instance is in a different containing object instance.
* The fact that a particular NSDM is of an inner class type has no effect on [trivial copyability](http://en.cppreference.com/w/cpp/concept/TriviallyCopyable) with regard to the containing class. So an inner class member could only prevent the containing class from being trivially copyable if the inner class itself were not trivially copyable, not simply due to being an inner class.
* Types that declare stateful inner class members cannot have a trivial default constructor, and therefore cannot be [trivial types](http://en.cppreference.com/w/cpp/concept/TrivialType).
* Stateless inner class members do not affect the layout of the class containing them, as is normal for stateless classes.
* Stateful inner class members prevent their owning types from being standard layout.

In all other ways, inner class types work just like regular types.

### Template Specialization

If an inner class is a template, then all of the specializations of that template must also be inner classes. And vice-versa. A template class's instantiation will therefore be either inner or not inner, regardless of the template parameters used on it.

The purpose of this rule is really user-sanity. Users probably don't mean for a specialization of some inner class template to not be an inner class. And vice-versa. So it's far more likely that this would happen due to user error than deliberate action.

### Containment Scope

Inner classes can be nested within inner classes. The reach of an inner class always extends up its ownership hierarchy to the first non-inner class. That first non-inner class is the boundary of the reach of the inner class. The classes between the inner class and the reaching class (including that class) are the inner class's scope.

Because inner classes are nested classes within every class in their scope (and possibly outside of it), inner classes have access to everything that their scope can access.

Within member functions of inner classes (except for constructors and destructors), the following rules are added.

Members (static or not) of an inner class may access members of instances within its scope through a pointer/reference to its own class (including implicit `this`) as though it were derived from the classes in its scope. Name lookup takes place as though the direct owning class were placed at the end of the inner class's list of derived classes. Therefore explicitly derived classes take priority over owning ones. This implied derivation propagates up the entire scope of the inner class.

Qualified lookup within inner class member functions can also be performed through the scope as well. Similarly, inner classes may implicitly convert pointers/references of their own types into pointers/references to any class in their scope, under similar derived rules.

Friends of inner classes gain these abilities as well (mainly to allow lambdas to work).

Non-friends may perform explicit conversions from a pointer/reference to an inner class to a pointer/reference to one of its owning classes via `static_cast`, so long as they have access to the class's name. But implicit conversion is not allowed.

### Destructors and transformation

There is a danger with this transformation mechanism. As stated above, the ability of inner class members (and friends) to transform pointers to one of the types in its scope and access members is not provided to the inner class's constructors and destructors. This is obvious due to lifetime rules. The owning object is guaranteed not to exist while the constructor is being called. Similarly, the owning object must have been destroyed before calling an inner class destructor.

The problem is what happens if you are not yet in a destructor, but will inevitably be there. Consider the following:

	struct A
	{
		inner struct B
		{
			inner struct C
			{
				void do_stuff() {outer();}
			};
			
			C c;
			
			~B() {c.do_stuff();}
		};
		
		B b;
		
		void outer() {...}
	};

	A a;

It's perfectly fine to call `a.b.c.do_stuff()`. However, when `a` is destroyed, `~B` will call a member of `C`. That's seemingly fine, since `c` will still exist. The problem is that `C::do_stuff` will call a member of `A`, which is a class that was destroyed before `~B` was even called.

This cannot be dealt with by making the calling of inner class members from a destructor within that inner class's scope be il-formed. One could always pass a reference to the inner class member to someone else who calls it.

On the surface, this seems similar to the problem of virtual functions being called during constructors/destructors. The obvious difference is that those calls are dynamic, while these conversions are always static. So if a constructor/destructor calls a function that performs such conversion, either implicitly or explicitly, there is no way to tell whether it is being done in a constructor/destructor or not.

Therefore, we must say that it is undefined behavior to perform any transformation to a class who's lifetime has ended. The big concern here is that it opens up a way to easily, and undetectably, do bad things.

### Implementation {#inner_impl}

The fundamental operation of inner classes is the (static) transformation from a pointer to the inner class type to a pointer to its direct owning class type. Once such a conversion is possible for one step, it is possible for every step through an inner class's scope. This operation has to be implemented for two cases: one for stateless inner classes and one for non-stateless inner classes.

#### Stateless

The stateless version is based on the fact that pointers to stateless classes could point to anything. Therefore, they can point directly at their owning class instance. So the memory location of a stateless inner class pointer should always be the same memory location of their owning class. If their owning class is itself a stateless inner class, then this propagates up the ownership hierarchy to the first non-stateless inner class or non-inner class.

Note that if the owner is a stateless class but not an inner class, then the memory location for that instance can be determined in the usual way for any other stateless class.

There is a snag that appears when a stateless inner class is an NSDM of a class derived from its owner. In this case, the compiler needs to recognize that its owning class is a base class of its container. This is a static property, so it is easy to know about. Once the compiler recognizes that, it can assign the pointer/reference of the NSDM by doing a `static_cast` from the derived class to the base class, then giving that memory location the inner class type.

This becomes more complex when deriving from inner classes:

	class Base
	{
	public:
		inner stateless struct In1
		{ int GetBase() const {return baseVar;}  };

	private:
		int baseVar;
	};
	
	class Derived : public Base;
	{
	public:
		inner stateless struct In2 : public Base::In1
		{
			int GetBase2() const {return GetBase();}
		};
		
		In2 acc;
		
	private:		
		int deriVar;
	};
	
The key element to remember here is that both `Base` and `Derived` have non-stateless NSDMs, so they both take up space. `Derived` is not standard layout, and therefore the pointer to `Base` need not have the same memory location as a pointer to `Derived`.

But even with this complexity, the general plan still works. In `Derived::In2::GetBase2`, the compiler recognizes that `GetBase` is a member of the base class of `Derived::In2`. Therefore it needs to convert `this` from `Derived::In2*` to `Base::In1*`. The compiler can also see that the base class is an inner class, whose owner is a base class of this inner class's owner.

So the sequence of conversion steps is as follows. The compiler converts the current `this` pointer of type `Derived::In2*` into a `Derived*` (changing only the type), then converts it into a `Base*` (possibly changing the memory location), then converts it into an `Base::In1*` (again changing only the type). The same thing happens for calling `acc.GetBase()` directly.

All of these conversions are static, based on compile-time generated offsets. They do not rely on a particular instance of any of the stateless inner classes. In terms of performance, it is no different from doing `static_cast<Base*>(&derived)`.

#### Stateful

Stateful inner classes are more complex. This is because each stateful class instance must have a separate address from every other stateful NSDM. Because each member has its own state, the conversion from `Outer::Inner*` to `Outer*` is no longer a simple typecast or a static pointer offset. And if we want to be able to pass around references to stateful inner classes, the only data we can use for that conversion process is the inner class's `this` pointer alone.

Therefore, for each stateful inner class member, the compiler must have access to some data (typically a byte offset) which is used to convert `this` from `Outer::Inner*` to `Outer*`. For multiple nestings of stateful inner classes, each level has its own offset to get to the next level.

The key question is where this offset is stored.

One thing is certain: the offset must be stored someplace that is accessible from just a pointer to `this`. After all, users may have only a pointer/reference to an inner class member, and they have every reason to expect that this will be a working pointer/reference. In such a case, the only pieces of information the compiler has are the class definitions and `this`.

We are left with two alternative implementations. The offset could be stored within the inner class itself, in a hidden member (ala vtable pointers). Or the offset could be stored in the direct owning class, in memory that is directly adjacent to the instance.

Both of these have behavioral consequences. Both solutions break standard layout, as a consequence of having to insert additional data. But this is to be expected. The question is what functionality each option allows and what it forbids.

**Option 1: In-Object Memory:** This is the most obvious implementation strategy. The offset is simply a hidden field of the inner class.

The downside of this approach is that stateful inner classes cannot be trivially copied. This is for the same reason that virtual classes cannot be trivially copied. The value of the offset is based on the specific member variable and where it is within its owning type. As such, copying it between two *different* members (of the same type) is really bad. And since the offset is part of the type, you can't copy the two types with a simple `memcpy(dest, src, sizeof(T))`. And therefore, they are not trivially copyable.

The odd part here is that, while inner classes themselves are not trivially copyable, the class containing inner class members still can be trivially copied. This works because the data is an offset, not a pointer directly to the owner. As such, trivial copies can work if they are they are between the same variable (the variable defines the offset). Which they will be if the copy is between two instances of the containing class.

So this option does allow the containing object to remain trivially copyable. But the rules for trivial copyability become more complex, since the containing class does contain a member that is not trivially copyable.

The rules would have to say that inner classes are not trivially copyable. But if the only reason that the class is not trivially copyable is that it is an inner class, then using it as a base class or NSDM does not affect the trivial copyability of the containing class.

Also, the containing class cannot be *trivial*. For each stateful inner class NSDM, the default constructor must provide an offset, based on the layout of the class. Note that these offsets must exist for all non-copy/move constructors of the containing class.

**Option 2: Adjacent Memory:** The offset could be stored in the containing class as a hidden value. The offset would be stored directly before its associated inner class.

If the offset is adjacent to the object, then fetching the offset with only `this` is easy: simply decrement the pointer so that it points to the adjacent memory. The class definition will tell the compiler if it is stateful, and if it is, then it knows it needs to do this work to get the offset.

The adjacent memory method preserves trivial copyability of both the inner and containing classes, because the offset is where it needs to be: associated with the type that actually defines that offset. The offset is defined by the arrangement of the members of the outer class. So the outer class is the one that stores the offset. The outer class can still be trivially copied because the offset for each member is a static property, not a dynamic one.

The outer class cannot be *trivial* however, since its default constructor (and every other non-trivial-copy/move constructor) will need to initialize this hidden data.

The downside of this choice is that it breaks the ability to have arrays of stateful inner class members. The problem is that arrays in C++ are required to be contiguous, that the pointer distance from one array element to another must be exactly `sizeof(Class)`.

Note that since we forbid arrays of stateless subobjects period, forbidding arrays of state<b>ful</b> inner classes is not far afield.

The question is which is more useful: simpler trivial copyability rules and trivial copyability for inner classes vs. aggregating arrays of stateful inner classes? This proposal focuses on the former.

**Generating Offsets:** Stateful inner classes are complicated because inner classes can be used in two subobject forms: NSDMs and base classes.

The offset for the NSDM is simple: it is the byte offset from the pointer to that member to the owning class. Even if the NSDM is declared in a derived class of the owner, there is still some offset from that member to the owner.

The offset for base inner classes is more complex, as is where it gets computed. The byte offset needs to be an offset from the current `this` pointer to the owning instance. However, the current `this` pointer is not necessarily the NSDM's address; it's a base class subobject of that NSDM.

This offset must be generated by the containing class of the NSDM. So not only must it generate the offset for the NSDM inner class, it has to generate one for each inner class base of the NSDM.

**Stateless and Stateful:** Consider the following code:

	struct Base
	{
		stateless inner struct i_base {};
		
		i_base mem_base;
		
		int base_data;
	};

	struct Derived
	{
		inner struct i_derived : public Base::i_base {};
		
		i_derived mem_derived;
	};
	
	void Func(Base::i_base &member);

This is an interesting case because it excercises all of our rules: offsets for the stateless `i_derived` and the statically-determined offset for `i_base`.

You can call `Func` with a reference to `Base::mem_base`, and it will do its work just fine. But how does the call to `Func` work with a reference to `Derived::mem_derived`?

The compiler does type conversion. It knows that it is converting from a reference to a stateful inner class to its base class. And it can see that the base class is stateless. And it can see that the class the stateless base is a base class of is *not* the owner of the `i_derived` class.

So it takes the pointer to `mem_derived` and converts it to a pointer to `Base`, as described above. Then, it uses that pointer value for the reference given to `Func`.

If `i_base` were instead a member of `Derived`, then the offset in `mem_derived` would be used to convert the pointer to a pointer to the `Derived` instance.

### Other Implementation Based Restrictions

You will notice quite a lot of places where virtual inheritance is forbidden. This is done to ensure that all of the pointer conversions remain *static* operations. Virtual inheritance would require dynamic operations, where a simple pointer offset is not possible. Virtual functions themselves are fine.

Strict aliasing in C++ says that two unrelated pointers point to different objects. However, inner class members are related to one another, even if they are technically siblings. So the definition of "related" needs to be expanded to cover inner classes.

Specifically, inner class types are related to the class they are declared within. Recursively up to the first non-inner class.

## Mixin Classes

The syntax for mixin declarations is somewhat in question at present. The general ideas for the syntax is to:

1. Make being a mixin a first-class property of the type, which is associated with the type's declaration. This makes it possible for the compiler to verify that the user is using the mixin type correctly.

2. Rather than arbitrarily deciding which template parameter is the derived class type, allow the user to specify it explicitly. This makes the system easier for users to use, allowing the template argument interface to be what is most natural for the type.

3. Allow the directly derived class to not have to type its classname directly into the base class instantiation. This prevents user error, as well as making it abundantly clear to the reader that a mixin is being used. It also allows anonymous classes to use mixins.

4. Prevent users from accidentally putting the derived class type in the wrong location in the argument list. That is, both the derived class and the mixin implementation must agree which template argument is the mixin's owning type.

5. Allow the class including a mixin to determine if it is simply using the mixin or whether it is composing itself with the mixin.

### Class Declaration

A mixin class is a template class that is declared as follows:

	template<parameters>
	class name mixin<parameter>
	{
		...
	};

The `parameter` associated with the `mixin` keyword must be the name of one of the `parameters` listed in the template argument list. This parameter must be a typename (it can be constrained by a concept). The specified template parameter becomes the owning type for the mixin.

The `parameter` may not be any form of expression; it must be exactly the name of a template parameter.

The `mixin` part of the class declaration must be a part of any forward declarations of the class. And it must consistently use the same parameter name (unless it is a template specialization).

A class that is declared as an inner class cannot also be a mixin.[^3] A mixin class can be declared `stateless`.

Mixin classes obey all the rules of templates and classes, except where noted below.

The standard library should have a template metafunction (and variable) for testing if a type is an mixin class.

### Type Usage

There are two general ways that a mixin can be incorporated into a class. The class using it may want to be composed with the mixin, such that it forms a new mixin that is the union of itself and the mixin it is being composed with. Or the class may wish to simply use the mixin in its implementation.

To declare that a type is using a mixin, you derive from it with this syntax:

	template<typename Derived>
	class name mixin<Derived>
	{	
		...
	};
	
	class derived : public name<mixin>
	{
	};

When performing mixin composition, the syntax changes to this:

	template<typename Derived>
	class name mixin<Derived>
	{	
		...
	};
	
	template<typename Derived>
	class composed mixin<Derived> : public name<compose>
	{
		...
	};

Note that compose syntax only works if the derived class is a mixin or an [inner class](#inverted_scope).

No variables of mixin types can be declared. Objects of mixin types can only be created as base subobjects of other classes, and they must use one of the two above syntaxes when deriving the class. Virtual inheritance of mixin classes is not allowed.

The `mixin` and `compose` keywords are used in the template instantiation syntax to refer to the specific template parameter that was specified to be the mixin type. The template will be instantiated as if `mixin`/`compose` was replaced by the derived class. If the mixin has multiple template parameters, the others can be filled in as normal. The `mixin`/`compose` keywords must be applied to the template parameter that was declared to be the mixin parameter in the template declaration.

Template substitution works exactly as if the user had used `name<derived>`. However, the system will note whether `mixin` or `compose` was used, as which one is used will have an effect on the [reach of the mixin](#mixin_scope).

Not using the `mixin`/`compose` keyword when deriving from a mixin class template is an error, even if it correctly names the derived type:

	class mine : public name<mine> //Not allowed
	{
	};

Similarly, when deriving from non-mixin types, the use of the `mixin` or `compose` keyword in the template argument list is not allowed.

By using a special keyword to denote that a mixin is being used, we prevent several classes of errors (not passing the right type, etc). But we also allow the use of mixins on anonymous types:

	//No name? No problem.
	class : public_name<mixin>
	{
	} variable_name;

When naming the concrete mixin base class *within* the definition of members of the direct derived class, `mixin`/`compose` must be used in the template argument list. And the keyword used must match what was in the class definition. Outside of members of the derived class, the user must name the derived class explicitly.

Pointers and references to mixin classes can be used as normal. Mixin base classes operate as normal base classes do, except where mentioned below.

### Type Aliases and Mixins {#mixin_aliases}

Aliases for mixins work unusually. A mixin must always have a template parameter specified as the mixin's owning type. And aliases are no exception. Therefore, `typedef` cannot be used to create a mixin alias, only `using` syntax.

The `using` declaration must have at least one template parameter, and the syntax must specify which template parameter is the mixin's owning type. Furthermore, that parameter must be used in the alias itself. Here is an example:

	template<typename T, typename derived>
	class old_mixin mixin<derived>;

	typedef old_mixin<int, some_class> bad1; //Illegal, no template parameter for the mixin type.

	template<typename T>
	using bad2 = old_mixin<T, some_class>; //Illegal, mixin type must refer to a template parameter.

	template<typename derived>
	using bad3 = old_mixin<int, derived>; //Illegal, new alias must specify which parameter is a mixin.

	template<typename T, typename derived>
	using bad4 mixin<T> = old_mixin<T, derived>; //Illegal, the specified mixin parameter is not passed to the original type's mixin parameter.

	template<typename derived>
	using correct mixin<derived> = old_mixin<int, derived>; //Works

Mixin template parameters can be constrained by concepts, but they must still resolve to a type.

The reason why mixin aliases must always have a template parameter for the mixin type is because of the syntax used for deriving from a mixin. The current syntax has the user put the `mixin`/`compose` keyword in the template argument list, which not only fills in the class but tells the compiler to make sure it's a mixin.

If we want to have cases like `bad1` or `bad2` work (those mixins could only be used with `some_class`), it is still important that the user deriving from the class make it clear that they are aware that it is a mixin. Because inheriting from a mixin [implicitly makes the mixin a friend of the class](#mixin_scope), it is very important to make it clear to the user at all times that a mixin is being derived from. We would also need a way to specify the particular form of inheritance: `mixin` vs. `compose`.

### Mixin Specializations

If one specialization of a template class is not a mixin, then all specializations must not be mixins.

Similarly, if one specialization is a mixin, all of them must be mixins. A mixin specialization may not declare a concrete class to be the mixin type. The parameter may be constrained via concepts, but it must still resolve to a type.

The reason for this limitation is similar to the [one for aliases](#mixin_aliases). If we allow specializations for a specific mixin base/derived class pairing, there would need to be some way for the user of the mixin to provide some idea that they intend to derive from that mixin.

### Inversion Scope {#mixin_scope}

Scoping for mixins is a bit more complex, due to the fact that the kind of composition affects how far a mixin can reach.

An instantiation of a mixin could reach through many classes that inherit from it. The class that directly derives from it is always reachable. However, a mixin's reach can only extend *through* that class if it used `compose` during its derivation. Therefore, a mixin can reach each derived class up its hierarchy to the first non-`compose` derivation. This first non-`compose` derivation is the mixin instantiation's reaching class.

The classes between the mixin class and its reaching class (including that class) are the mixin's scope.

Mixin instantiations are implicitly a friend of every class within their scope. Additionally, mixins are implicitly friends with every class that is explicitly friends of one of the classes in its scope.

Note that this rule *explicitly* excludes the implicit friendships from inner classes and other mixin derivation. The reason for excluding such friendships is to maintain some control over the mixin's reach. If we have a mixin that includes another mixin but does not use `compose`, we do not want the used mixin to be able to access anything beyond its reach. So the new mixin could have its own reach, while preventing the non-composed mixin from being able to have visibility through it.

Within member functions of mixin classes (except for constructors and destructors), the following rules are added.

Members (static or not) of a mixin class may access members of instances within its scope through a pointer/reference to its own class (including implicit this) as though it were derived from the classes in its scope. Name lookup takes place as though the owning class were placed at the end of the mixin class's list of derived classes. Therefore explicitly derived classes take priority over owning ones. This implied derivation propagates up the entire scope of the mixin.

Qualified lookup within mixin member functions can also be performed up the ownership hierarchy as well (this is not particularly useful, as the mixin only knows about its direct parent). Similarly, mixin classes may implicitly convert pointers/references of their own types into pointers/references to their owners, up through the mixin's scope.

Friends of mixins gain these abilities as well (mainly to allow lambdas to work).

The ability of non-friends to perform such conversions depends entirely on the access classes of those non-friends and of the inheritance diagram. Or to put it another way, the rules of casting up and down the hierarchy remain unchanged for non-friends of mixins.

### Possible Additions

* Given a mixin class, syntax could be added to get the derived class type. Perhaps `mixin<mixin_type>` would resolve to a typename. This would always be of the immediate parent class.

## Inverted Scope Interactions {#inverted_scope}

Inner classes and mixins propagate access and friendship up to the first non-inner class or non-`compose` mixin inheritance. But because they are really the same concept implemented in different ways, they should interact well together.

If a mixin declares an inner class, the inner class's reach extends through its mixin owner up to that class's eventual reach.

For the reverse, things are similar. When an inner class derives from a mixin, it may use `mixin` or `compose`. If it uses `compose`, then the reach of the mixin extends through the inner class to the eventual reach of that inner class. If it just uses `mixin`, the reach stops with the inner class.

Inner classes should always be considered part of the implementations of their owners. Mixins sometimes should and sometimes should not, depending on the class that is using it.

So if you have this:

	template<typename derived>
	struct M2 mixin<derived>
	{
		inner struct I : M1<compose>
		{
			int val_i;
		} in;
		
		int val_m2
	}

	struct T : public M2<mixin>
	{
		int val_t;
	};

In instances of `T`, functions from `M2<T>::M1<I>` will be able to access `val_i`, `val_m2`, and `val_t`, in that order. If `I` had used `mixin` instead of `compose`, `M1<I>` would only be able to access `val_i`.

This feature allows users to use mixins to write the implementation of inner classes outside of the containing class. Thus the guts of an implementation can be used in multiple classes, while still allowing class-specific tweaks to be provided by the inner class using the mixin. For example, `I` could provide its own `val_m2` that shadows what `M1` can access. And because `M1`'s implementation is ignorant about the world outside of the mixin that it is declared within, it has no way to bypass this shadowing.

# Alternatives Not Taken

There are some alternative possible designs for these features. This will discuss why these are not being proposed.

## Full Java Inner Classes

This proposal is more narrowly focused than Java-style inner classes. In this proposal, inner class variables must be direct members of either their owning class or a class statically derived from it.

In Java, inner class variables can be allocated anywhere. And they can be attached to any instance of their owning class. Java even provides special `new` allocation syntax for doing so: `instance.new inner_class(...)`.

This is reasonable for Java because it is garbage collected. Therefore, an inner class can safely store a pointer to its owner. In C++, this is much less safe. While pointers/references to other objects have always been a part of C++, C++ also has various ownership relationships built into the type system. And any ability to create instances of inner classes as non-members would also need to allow the expression of that ownership. Would such an inner class take its owner as a `shared_ptr`?

Such functionality is simply too cumbersome for C++ as a language. It also lacks much of the utility of inner classes.

The restricted version of inner classes proposed here are sufficiently useful in their own right, and avoid pushing too far into dangerous and complex ownership issues.

## Mixin as a Separate Construct

Mixins, as presented here, are simply a small modification of the existing type system. For the most part, they are just template classes, with all of the rights and restrictions thereof, except where it conflicts with the mixin design.

Other languages implement mixins as a separate thing, outside of the type system. D-style mixins, for example, are essentially a type-safe macro system. A type which uses a mixin effectively copies and pastes its definitions into another class.

The mixin system presented here is much more modest. Even so, it provides most of the functionality of a D-style mixin. The only downside of this version, relative to D-style mixins, is that it relies on C++'s multiple inheritance. Some people dislike multiple inheritance, both for objective and subjective reasons. But even this has advantages, as template functions can explicitly take pointers/references to classes which derive from a mixin, instead of using a concept-based interface.

# Remaining Issues to Resolve

* Pointers-to-member variables of stateless type. Is that even a problem? Could they not all have the same effective data?

* Mixin aliases & specializations. Currently, the mixin owning type must always be a template parameter, preventing complete mixin specializations for specific types. Should we allow full specialization?

* Full mixin inversion. The CRTP implementation of mixin composition works by passing the derived class through the mixin. This means that, if the composed mixin implements some part of the interface of one of its components, the derived class can still override that interface. However, the mixin language feature does not do this; it works based on checking the class hierarchy in reverse order: least derived first, then more derived. Which way is the way that users will expect it to work? Or do we need to explicitly allow the user to choose how it works per-mixin or per-derivation?
	
# Acknowledgements

* 

# Changes

* From version 0.6:
	* Added conditional stateless expressions.
	* Stateless arrays are allowed. Inner class arrays are still forbidden.
	* Better examples for the utility of inner classes.
	* Statelessness redefined as primarily a subobject property, not a class property.
	* Stateless arrays forbidden again. Arrays of empty classes are allowed.

* From version 0.5:
	* Altered intro and restructured the document, putting uses after concepts.
	* Removed erroneous claims about current CRTP implementations.
	* Added functionality for explicitly restricting the reach of a mixin.
	* Added discussion of mixin overriding inversion.
	* Added section on alternative designs in other languages.
	* Subsection on how inner class offsets are generated.
	* Added section on changes.

[^1]: The term "mixin" as it is used by C++ may be different from how it is used in other languages. However, the concept is essentially the same: add features from outside of a class which work as if they had been declared within it as much as possible. Other languages might implement this concept as macros or via some explicit construct. C++ usually implements it through inheritance.

[^2]: Implementation note. For stateless inner classes in derived classes, the pointer to the stateless class has to have the same value as the base class. So special pointer gymnastics are needed when getting pointers/references to such variables. For stateful inner classes, the offset simply needs to be able to jump out of the derived class type and into the base class. So for some implementations, that may need to be a negative offset.

[^3]: This is not a strictly necessary restriction. Inner classes can be derived from other inner classes, after all. However, an inner class mixin could only ever be used as the base class of another mixin. And if you use `compose` when deriving that inner class, then the mixin can access the scope of the inner class deriving from it. So there is no need to declare the mixin to be an inner class. Also, it confuses questions of how the compiler converts the `this` pointer, whether it uses inner class conversion or mixin conversion. By forbidding them explicitly, we make it clear how conversion works for each step.