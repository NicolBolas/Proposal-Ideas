% Uninitializing Constructor
%
%

# Motivation and Scope

Having a default constructor and/or default member initializers perform initialization is very much akin to making value initialization be the default form of initialization in container insertion operations.

The problem with doing so is that it becomes impossible to allow an object of that type to be uninitialized. Currently, only the presence of a trivial default constructor can cause the object to be uninitialized. If the default constructor exists and is non-trivial (or the class has DMIs), then there is no way to create uninitialized instances of the class. That may be what you want in some cases. But sometimes, you want to have the default constructor initialize the object, while having some other constructor leave the object (and its subobjects) uninitialized.

It would be useful to have constructors that can construct the object without initializing its members. Obviously, this would be an opt-in feature; we absolutely do not want objects to accidentally be uninitialized.

# Design

A constructor can be declared to be an uninitializing constructor with the following syntax:

`````
struct A
{
	A(some_type) = uninitialize;
};
`````

Where `uninitialize` is a special identifier. There should not be any grammatical problems with such a construct.

An uninitializing constructor is a trivial constructor that will not initialize any of its subobjects. Any default member initializers of non-static data members will not be used.

Declaration of an uninitializing constructor for a type requires the following:

1. The type has no virtual member functions or base classes.
2. Subobjects must either have a trivial default constructor or at least one uninitializing constructor. The constructor in question must be accessible from this constructor.
3. All parameters to the constructor must be reference types.

By declaring a type to have an uninitializing constructor, you are saying that it is OK for this object to be constructed in an uninitialized way. So if a subobject class has default member initializers (and thus a non-trivial default constructor), a user can declare that bypassing those initializers is OK through the use of an uninitialized constructor.

If a default constructor is declared uninitializing, and the aforementioned rules are satisfied, this default constructor will be considered both defaulted and trivial. Declaring the default constructor to be uninitialized does not weaken the above rules; if the object does not pass, then the program is ill-formed rather than simply deleting the default constructor.

When initializing an object, if an uninitializing constructor is selected, no matter the form of initialization, then the object will be uninitialized. Note that this is without regard to any default member initializers. They are ignored by uninitializing constructors.

Rule #3 exists to facilitate optimal implementations of uninitializing constructors. Such constructors cannot do anything with their parameters, but the parameters would ordinarily still be constructed and destroyed, per C++'s standard function calling rules. So side-effects of constructing and destroying the parameters would be visible.

By employing rule #3, we remove the chance of such side-effects. Therefore, we effectively permit the compiler to detect that such a constructor will be called and just drop the parameters on the floor.

Note that the lifetime rules in [basic.life] should be updated. Initializiation for an object is non-vacuous if it is initialized via a constructor that is not a trivial default constructor or an uninitializing constructor.

# Impact on the Standard

This adds a special identifier, `uninitialize`.

# Acknowledgments

* Sean Middleditch, for his alternative idea and some impetus for cleanup. Also, for the uninitialized constructor idea.

