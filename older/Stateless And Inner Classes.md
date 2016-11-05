# Stateless and Inner Classes

This proposal describes two separate concepts, but they each contribute useful functionality to the other.

## Stateless Classes

Conceptually, a stateless class is a class which takes up no space when used as a direct member of another class. The purpose of this feature is to essentially require empty member optimization, but only where specified explicitly.

### Definition

This proposal does not yet suggest any particular syntax for defining a stateless class. The shorthand for this will be:

    <<stateless>> class Classname { };

All uses of a stateless class, where the statelessness of the class would actually matter, require the class definition. Therefore, the stateless class syntax does not need to be available for forward declarations of a class. Also, whatever stateless syntax is used, it should be able to be used with anonymous class definitions.

Stateless class definitions:

* May not derive from any non-stateless type.
* May not have NSDM's of non-stateless types.
* May not have virtual functions.
* May not have virtual base classes.

[note: These properties mean that a stateless class is always standard layout.]

In all other ways, a stateless class definition is equivalent to a non-stateless class definition.

### Usage of Types

The `sizeof` a stateless class is not affected by the fact that it is stateless. [note: This means you get what you would for any empty class.]

The standard library should have a template metafunction (and variable) for testing if a type is a stateless class.

Stateless classes can be used in exactly the same ways as regular classes with the following exceptions:

* Variables which are arrays of stateless types cannot be declared.[^1] Arrays of stateless types can still be allocated with `new[]`.

If a class declares a NSDM of a stateless class type, or non-virtually derives from a stateless class type, the size of the class will not be affected by the presence of this member/base class. Stateless members/base classes do not affect the layout of members within the object. [note: For example, if a class has all of its regular members public, it may have a private stateless member without affecting whether it is [standard layout](http://en.cppreference.com/w/cpp/concept/StandardLayoutType).] Stateless members/base classes can affect whether the class is [trivial](http://en.cppreference.com/w/cpp/concept/TrivialType) or [trivially copyable](http://en.cppreference.com/w/cpp/concept/TriviallyCopyable) as normal. [note: If the stateless class is not trivially-copyable, then classes that use them will not be as well.]

**Question:** Do we want stateless class NSDM's to take part in aggregate initialization? If we want stateless class members to effectively be transparent, to purely be implementation details, then we probably don't want them to be part of aggregate initialization. At the same time, stateless classes still exist and can still have constructors.

### On Implementation {#stateless_impl}

Implementing stateless types should be easy in the compiler. A more difficult question is within the language.

Objects are defined as a region of memory. And while stateless types do have some region of memory, that region may be shared by other, unrelated types. A type that has multiple stateless members will likely give them all the same pointer value (a pointer to their owning type).

This already comes up with regard to the [empty base optimization](http://en.cppreference.com/w/cpp/language/ebo). When this optimization is employed, all empty base classes have the same pointer address. Standard layout rules require EBO even for multiple inheritance with empty base classes, which means multiple seemingly unrelated types can end up with the same pointer address, refering to the same region of memory.

So the standard already deals with the concept of multiple, disparate objects in the same memory. Implementing stateless members only requires that the standard handle them at the NSDM level.

## Inner Classes

The Java concept of an inner class is a nested class, all objects of which are bound to some instance of the containing class. The inner class can access members (static and non-static) of the containing class/instance as easily as it accesses its own. In this way, an inner class instance is effectively an extension of the outer class instance.

This proposed C++ concept is similar (hence the name), but it does not allow for dynamic allocation of such objects the way one can in Java.[^2]

### Definition

Inner classes can only be defined in the scope of a class; this class is called the *owning class*. [note: The owning class may itself be an inner class] This proposal does not yet suggest any particular syntax for defining inner classes. The shorthand for such definitions will be:

    <<inner>> class Classname { };
	
Uses of inner classes that require the compiler to know that it is an inner class also requires the full definition of the inner class. As such, the inner class syntax does not have to be used in the class declaration; just the definition. However, the inner class syntax must also allow for anonymous inner class definitions.

Also, such syntax should be able to be used with the above stateless syntax. Neither implies the other, but it should be possible to use both when so desired.

A class may only be derived from an inner class types if that class is itself an inner class. Also, when deriving from an inner class, the owning class of the derived class must be either the same class as the base class's owning class or a class non-virtually inherited[^3] from it. When deriving from an inner class type, the inheritance cannot be virtual.[^3]

Inner classes may derive from any available type, as normal. They may have any members, as normal, and they can use all other types, as normal.

### Usage of Types

Variables of inner class types can only be declared as NSDM's of a class. And the class which they are declared within must be either:

* The direct owner of the inner class.
* A class non-virtually inherited[^3] from the direct owner of the inner class.

Inner class types cannot be used as the type in `new` expressions, nor can temporaries be made of them. Objects of inner class types cannot have their destructors explicitly called.

The standard library should have a template metafunction (and variable) for testing if a type is an inner class.

Pointers and references to inner class types work as normal for any type.

Questions regarding layout compatibility, trivial copyability, and certain specific uses for inner classes are discussed in the [implementation](#inner_impl) section. This is because these behaviors materially affect how the compiler actually makes these things work. The behavior, based on looking at the limits of implementations, is as follows:

* Stateless inner class members do not affect layout compatibility.
* Inner class members, stateless or not, affect trivial copyability in the same way as any other type. [note: The class declaring the member is not trivially copyable only if the member type is not trivially copyable.]
* Objects which have stateful inner class members cannot be trivial types. [note: They can still be trivially copyable.]
* Inner class members can be copied/moved exactly as any other type. [note: Specifically, if a stateful inner class member is trivially copyable, it should be legal to do a `memcpy(dest, src, sizeof(T))` on it into another inner class member of the same type.]

In all other ways, inner class types work just like regular types.

### Names and Access

The inner class of a class is implicitly friends of its owning class class. This happens recursively up the inner class hierarchy. [note: An inner class of an inner class is friends with both its immediately owning class and the class that owns that one.]

Name lookup through inner classes can retrieve members of their owning classes as though they were derived from them. However, actual base classes have priority over owning class, when names conflict. The more recent owners have priority over less recent owning classes.

Pointers and references to inner class members can be converted to point to their owning class instance via `static_cast`. Such conversions can also be implicit, much like converting from a derived class to a base class. [note: The reverse conversion is not allowed. You have to access the specific member yourself.]

If a pointer/reference to an inner class names a member of an owning class (accessibility willing) and attempts to access it through that pointer/reference, the pointer/reference will first be converted to the proper type (and value) via `static_cast`. [note: This means that one can actually access the public members of an owning class through just a pointer/reference to the inner class. But not the private ones, because that breaks accessibility.]

The exceptions to this access are within inner class constructors, NSDM initializers, and destructors. Because inner classes must be members, their owning class may not have been constructed yet. [note: If the object is a member of a class derived from its owning type, then the owning type will be constructed before the inner class member.] As such, they should not be able to modify their owning object before they have been fully constructed. Nor should they be able to modify their owning object in their destructor, since the lifetime of that object has ended.

### On Implementations {#inner_impl}

Implementing inner classes is something of a concern. In Java, this is trivial; just stick a `this` pointer into the inner class object. Garbage collection makes everything work out.

Such an implementation is possible in C++ (minus the GC). Indeed, there are class frameworks that implement this today. However, such implementations cause a number of problems, beyond the syntactic difficulties of using those frameworks:

* It breaks trivial copyability, as the hidden `this` pointer shouldn't be copied or moved.
* It introduces overhead where none might be necessary.

Only a compiler is capable of implementing inner classes without these flaws (where possible). Even so, compiler implementations lead to some problems.

It is possible to implement stateless inner classes with zero overhead in the owning class. This could be done easily enough by having the pointer to the stateless inner class member be the same pointer value as its owning object. This allows the compiler to instantly know how to convert from an inner class instance to its owning instance.

This would work even when the stateless inner class is a member of a derived class of its owner. The compiler can statically convert the pointer to the derived class to a pointer to the base class (since virtual inheritance is not allowed) when accessing the inner class member.

Then, there is this even more complex case:

	class Base
	{
	public:
		<<inner>> <<stateless> class In1
		{ int GetBase() const {return baseVar;}  };

	private:
		int baseVar;
	};
	
	class Derived : public Base;
	{
	public:
		<<inner>> <<stateless>> class In2 : public Base::In1
		{
			int GetBase2() const {return GetBase();}
		};
		
		In2 acc;
		
	private:		
		int deriVar;
	};
	
	
The key to remember here is that both `Base` and `Derived` have non-stateless NSDMs, so they both take up space. And thus, the pointer to `Base` is not required to be the same as the pointer to `Derived`.

Even with this complexity, things still work. For `In2::GetBase2`, the compiler recognizes that `GetBase` is in the base class of `Derived::In2`. But it also realizes that this base class is an inner class. So it converts the `this` pointer of type `Derived::In2*` into a `Derived*`, then converts it into a `Base*`, then converts it into an `Base::In1*`. The same thing happens for calling `acc.GetBase` directly, or any expression that converts an `Derived::In2*` into `Base::In1*`.

All of these conversions are static, based on compile-time offsets. Therefore, one can obtain a pointer/reference to `acc` and manipulate it. The compiler knows the type of the object, and it can see that it is an inner class type, and do the appropriate math to make it all work.

Where implementations get complicated is with stateful inner classes. Because each member has its own state, the conversion from `Outer::Inner*` to `Outer*` is no longer a simple typecast. It depends on exactly where that member is within its object.

Therefore, for each stateful inner class member, the compiler must have access to some data (typically a byte offset) which is used to convert `this` from `Outer::Inner*` to `Outer*`. For multiple nestings of stateful inner classes, each level has its own offset to get to the next level.

The question is where this offset is stored.

One thing is certain: the offset must be stored someplace that is accessible with just a pointer to `this`. After all, users may have only a pointer/reference to an inner class member, and they have every reason to expect that this will function. In such a case, the only pieces of information the compiler has are the class definitions and `this`.

We are left with two alternative implementations. The offset could be stored within the inner class itself, in a hidden member (ala vtable pointers). Or the offset could be stored in the direct owning class, in memory that is directly adjacent to the instance.

Both of these have behavioral consequences. Both solutions break standard layout, as a consequence of having to insert additional offsets. But this is to be expected.

**Option 1: In-Object Memory:** This is the most obvious implementation strategy. The offset is simply a hidden field of the inner class.

The downside of this approach is that stateful inner classes *cannot* be trivially copied. This is for similar reasons as to why types with virtual functions/base classes cannot be trivially copied. The value of the offset is based on the specific member variable and where it is within its owning type. As such, copying it between two *different* members (of the same type) is really bad. And since the offset is part of the type, you can't copy the two types with a simple `memcpy(dest, src, sizeof(T))`. And therefore, they are not trivially copyable.

**Option 2: Adjacent Memory:** If the offset is in memory adjacent to the object, then fetching the offset with only `this` is easy: simply increment/decrement the pointer so that it points to the adjacent memory. The class definition will tell the compiler if it is stateful, and if it is, then it knows it needs to do this work to get the offset.

However, this breaks the ability to have arrays of stateful inner class members. The reason being that arrays in C++ are required to be contiguous, that the pointer distance from one array element to another must be exactly `sizeof(Class)`.

Note that arrays of stateless inner classes are forbidden because arrays of stateless classes period are forbidden. So forbidding arrays of inner classes of any kind is not too far afield.

The adjacent memory method preserves trivial copyability because the offset is where it needs to be: associated with the type that actually defines that offset. The offset is defined by the arrangement of the members of the outer class. So the outer class is the one that stores the offset. The outer class can still be trivially copied because the offset for each member is a static property, not a dynamic one.

The outer class cannot be *trivial* however, since its default constructor (and every other non-copy/move constructor) will need to initialize this hidden data.

Trivial copyability is probably more generally useful than the ability to aggregate stateful inner classes into arrays.

## Uses and Deficiencies

Empty member optimization, and guaranteed empty base optimization outside of standard layout, has been something that users have wanted for quite some time. 

Inner classes can be used for properties.

One of the biggest downsides of this approach is the difficulty in sharing implementations of inner classes across disparate owning classes. One can use a template base class and the [CRTP](https://en.wikipedia.org/wiki/Curiously_recurring_template_pattern) to share implementations, but it's a huge pain. This requires writing a forwarding function for every member of the base class, which means if the base class changes, the derived class does not get the updates. Plus, each owning class must make the template base a friend explicitly, just to access the internal members.

Alleviating these problems would make inner classes even more useful, allowing them to be used to implement a form of mixins and such.

To do that, we could define the concept of an "inner class template". That is, one could declare an inner class outside the scope of a class, but only explicitly as a template. One of the template parameters would be the owning class type. And thus, name accesses that could implicitly use `this` would be considered dependent accesses.

The use of such a template mirrors the restrictions of inner classes. They could be instantiated essentially anywhere (but only with a complete type for the owning class), but variables of an instantiated type could only be declared in the woning class (or a non-virtually derived one). Such instantiations should retain the properties of inner classes: being friends of its containing scope and so forth.

Then again, it would also break encapsulation. Because template inner classes are implicitly friends of their owning type, it would be possible for a derived class to access a base class's private data by simply creating a template inner class, but giving it the owning type of its base class.


[^1]: The point of stateless classes is that they don't take up memory when they are members of objects. However, each array element has to take up space; this is a basic rule of C++ and pointer arithmetic. Forbidding stateless classes from being aggregated into arrays ensures that the user's expectation (not taking up space) are always met. This could be limited solely to arrays declared as NSDM's, but it's cleaner to just forbid them altogether.

[^2]: This is as much due to lifetime issues as anything else. Java is garbage collected, so it is reasonable to have a `this` pointer hidden within an inner class that points to the owning instance. And thus, the owning instance will not be collected until after its inner class objects are no longer in use. In C++, object lifetimes don't work that way, so we restrict this functionality to the one scenario where inner object lifetimes can be guaranteed. That is, direct members of the owning class.

[^3]: The non-virtual part is for ease-of-implementation reasons. It is not yet clear if a static byte offset would be sufficient with implementations of virtual inheritance. It is also unclear if some alternative implementation would also be possible that could work with virtual inheritance.