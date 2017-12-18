Title: Stateless and Inner Classes

This proposal describes two separate concepts, but they each contribute useful functionality to the other.

# Stateless Classes

Conceptually, a stateless class is a class which takes up no space when used as a direct member of another class. The purpose of this feature is to essentially require empty member optimization, but only where specified explicitly.

## Definition

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

## Usage of Types

The `sizeof` a stateless class is not affected by the fact that it is stateless. [note: This means you get what you would for any empty class.]

The standard library should have a template metafunction (and variable) for testing if a type is a stateless class.

Stateless classes can be used in exactly the same ways as regular classes with the following exceptions:

* Variables which are arrays of stateless types cannot be declared.[^1] Arrays of stateless types can still be allocated with `new[]`.

If a class declares a NSDM of a stateless class type, or non-virtually derives from a stateless class type, the size of the class will not be affected by the presence of this member/base class. Stateless members/base classes do not affect the layout of members within the object. [note: For example, if a class has all of its regular members public, it may have a private stateless member without affecting whether it is [standard layout](http://en.cppreference.com/w/cpp/concept/StandardLayoutType).] Stateless members/base classes can affect whether the class is [trivial](http://en.cppreference.com/w/cpp/concept/TrivialType) or [trivially copyable](http://en.cppreference.com/w/cpp/concept/TriviallyCopyable) as normal. [note: If the stateless class is not trivially-copyable, then classes that use them will not be as well.]

**Question:** Do we want stateless class NSDM's to take part in aggregate initialization? If we want stateless class members to effectively be transparent, to purely be implementation details, then we probably don't want them to be part of aggregate initialization. At the same time, stateless classes still exist and can still have constructors.

## On Implementation {#stateless_impl}

Implementing stateless types should be easy in the compiler. A more difficult question is within the language.

Objects are defined as a region of memory. And while stateless types do have some region of memory, that region may be shared by other, unrelated types. A type that has multiple stateless members will likely give them all the same pointer value (a pointer to their owning type).

This already comes up with regard to the [empty base optimization](http://en.cppreference.com/w/cpp/language/ebo). When this optimization is employed, all empty base classes have the same pointer address. Standard layout rules require EBO even for multiple inheritance with empty base classes, which means multiple seemingly unrelated types can end up with the same pointer address, refering to the same region of memory.

So the standard already deals with the concept of multiple, disparate objects in the same memory. Implementing stateless members only requires that the standard handle them at the NSDM level.

# Inner Classes

The Java concept of an inner class is a nested class, all objects of which are bound to some instance of the containing class. The inner class can access members (static and non-static) of the containing class/instance as easily as it accesses its own. In this way, an inner class instance is effectively an extension of the outer class instance.

This proposed C++ concept is similar (hence the name), but it does not allow for dynamic allocation of such objects the way one can in Java.[^2]

From an OOP perspective, inheritance represents the concepts "is a" or "implemented in terms of", relative to the derived class. Containment represents "has a", relative to the containing class. The C++ version of inner classes represent the concept "part of"; inner class objects are distinct sub-parts of their containing class. They are different types, but they are types which are inseparable from their owning type.

## Definition

Inner classes can only be defined in the scope of a class definition. The class which the inner class is directly nested within is called the *owning class*. [note: The owning class may itself be an inner class] This proposal does not yet suggest any particular syntax for defining inner classes. The shorthand for such definitions will be:

    <<inner>> class Classname { };
	
Uses of inner classes that require the compiler to know that it is an inner class also requires the full definition of the inner class. As such, the inner class syntax does not have to be used in the class declaration; just the definition. However, the inner class syntax must also allow for anonymous inner class definitions.

Also, such syntax should be able to be used with the above stateless syntax. Neither implies the other, but it should be possible to use both when so desired.

## Usage of Types

Objects of inner class types can only be created as NSDM's of their owning class. This means that inner class types cannot be used as the type in `new` expressions, temporary-manifestation, or even base class subobjects. Objects of inner class types cannot have their destructors explicitly called.

The standard library should have a template metafunction (and variable) for testing if a type is an inner class.

Pointers and references to inner class types otherwise work as normal for any type.

Questions regarding layout compatibility, trivial copyability, and certain specific uses for inner classes are discussed in the [implementation](#inner_impl) section. This is because these behaviors materially affect how the compiler actually makes these things work. The behavior, based on looking at the limits of implementations, is as follows:

* Stateless inner class members do not affect layout compatibility.
* Inner class members, stateless or not, affect trivial copyability in the same way as any other type. [note: The class declaring the member is not trivially copyable only if the member type is not trivially copyable.]
* Objects which have stateful inner class members cannot be trivial types. [note: They can still be trivially copyable, but their non-copy/move constructors can never be trivial.]
* Inner class members can be copied/moved exactly as any other type. [note: Specifically, if a stateful inner class member is trivially copyable, it should be legal to do a `memcpy(dest, src, sizeof(T))` on it into another inner class member of the same type.]

In all other ways, inner class types work just like regular types.

## Names and Access

Inner classes are nested classes and therefore follow the general access rules of nested classes. That is, they can access their owning class names and have all of the rights of any member of their owning class. Similarly, the owning class does not necessarily have rights to the inner class members.

There is a substantial change when it comes to inner class accessing their owning classes: accessing non-static members.

Within members of an inner class, and any friends of that inner class type, pointers/references to that inner class type are implicitly convertible to an instance of their owning class. This works recursively up the inner class ownership hierarchy to the first non-inner class owner.

This conversion is equivalent to a base class conversion, and it affects name lookup in the same way as a base class [note: The conversion in particular applies to `this`]. However, base class instances have name lookup priority over owning class instances. More recent inner class owners have priority over less recent inner class owners.

The name lookup can be scoped with a class name to ensure that the correct member is found.

For code which does not have non-private access to an inner class, the inner class is not implicitly convertible to its owning class. Such code only has access to the object's public interface.

Pointers and references to inner class members can be explicitly converted to point to their owning class instance via `static_cast`. The reverse conversion is not allowed.

The exceptions to this access are within inner class constructors, NSDM initializers, and destructors. Because inner classes must be members, their owning class has not been constructed yet. As such, they should not be able to modify their owning object before they have been fully constructed. Nor should they be able to modify their owning object in their destructor, since the lifetime of that object has ended.

Obviously, this can be enforced for constructors and destructors only. They can call other member functions, but these will have undefined behavior if they access any aspects of the owning object instance.

## On Implementations {#inner_impl}

Implementing inner classes is something of a concern. The fundamental operation that separates an inner class from other class instances is the ability to convert a pointer/reference to an instance of itself to an instance of its direct owning class. Such a mechanism can be recursive, if the direct owner is itself an inner class.

In Java, this conversion is trivial; when the inner class instance is created, you stick a hidden `this` pointer into the inner class object. Garbage collection makes lifetime issues work out, since it is referencing its owning type.

Such an implementation is possible in C++ (minus the GC). Indeed, there are class frameworks that implement this today. However, "hidden `this` implementations cause a number of problems, beyond the syntactic difficulties of using those frameworks:

* It breaks trivial copyability, as the hidden `this` pointer shouldn't be copied or moved.
* It introduces overhead where none might be necessary.

Only a compiler is capable of implementing inner classes without these flaws (where possible). Even so, compiler implementations lead to some problems.

It is possible to implement stateless inner classes with zero overhead in the owning class. This could be done easily enough by having the pointer to the stateless inner class member be the same address as its owning object. Thus, the fundamental operation is just casting a pointer to a new type.

Where implementations get complicated is with stateful inner classes. Because each member has its own state, the conversion from `Outer::Inner*` to `Outer*` is no longer a simple typecast. It depends on exactly where that member is within its object, which has to do with the class's memory layout.

Therefore, for each stateful inner class member, the compiler may[^3] need to have access to some data (typically a byte offset) which is used to convert `this` from `Outer::Inner*` to `Outer*`. For multiple nestings of stateful inner classes, each level can have its own offset to get to the next level.

[^3]: If a stateful inner class instance is used exactly once in the owning type, the compiler can pre-compute the offset and store it statically. Since inner class instances can only be created as member subobjects of their owners, if there is only one such instance created, then there is only one offset for that entire type, so it need not be stored. This is a useful optimization, but it does not change any of the rules here, much like devirtualization does not make a virtual class trivially copyable.

The ultimately question is where this offset is stored.

The fundamental operation of inner classes takes as input exactly one value: a pointer/reference to the inner class subobject. Therefore, whatever mechanism we come up with must be able to use `this` *alone* to find the offset.

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

# Uses and Deficiencies

Empty member optimization, and guaranteed empty base optimization outside of standard layout, has been something that users have wanted for quite some time. 

Inner classes can be used for properties.

One of the biggest downsides of this approach is the difficulty in sharing implementations of inner classes across disparate owning classes. One can use a template base class and the [CRTP](https://en.wikipedia.org/wiki/Curiously_recurring_template_pattern) to share implementations, but it's a huge pain. This requires writing a forwarding function for every member of the base class, which means if the base class changes, the derived class does not get the updates. Plus, each owning class must make the template base a friend explicitly, just to access the internal members.

Alleviating these problems would make inner classes even more useful, allowing them to be used to implement a form of mixins and such.

To do that, we could define the concept of an "inner class template". That is, one could declare an inner class outside the scope of a class, but only explicitly as a template. One of the template parameters would be the owning class type. And thus, name accesses that could implicitly use `this` would be considered dependent accesses.

The use of such a template mirrors the restrictions of inner classes. They could be instantiated essentially anywhere (but only with a complete type for the owning class), but variables of an instantiated type could only be declared in the woning class (or a non-virtually derived one). Such instantiations should retain the properties of inner classes: being friends of its containing scope and so forth.

Then again, it would also break encapsulation. Because template inner classes are implicitly friends of their owning type, it would be possible for a derived class to access a base class's private data by simply creating a template inner class, but giving it the owning type of its base class.


[^1]: The point of stateless classes is that they don't take up memory when they are members of objects. However, each array element has to take up space; this is a basic rule of C++ and pointer arithmetic. Forbidding stateless classes from being aggregated into arrays ensures that the user's expectation (not taking up space) are always met. This could be limited solely to arrays declared as NSDM's, but it's cleaner to just forbid them altogether.

[^2]: This is as much due to lifetime issues as anything else. Java is garbage collected, so it is reasonable to have a `this` pointer hidden within an inner class that points to the owning instance. And thus, the owning instance will not be collected until after its inner class objects are no longer in use. In C++, object lifetimes don't work that way, so we restrict this functionality to the one scenario where inner object lifetimes can be guaranteed. That is, direct members of the owning class.

