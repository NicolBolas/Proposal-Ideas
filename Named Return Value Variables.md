Title: Named Return Value Variables


# Motivation and Scope {#motivation}

Guaranteed elision made return value optimization and most other forms of elision required C++ behavior. Guaranteed elision did this by redefining the meaning of prvalue, making it not an object but an *initializer*. So a `return {};` means that the initializer `{}` applies directly to whatever object that prvalue is applied to.

Guaranteed elision allows the return of values which are immobile. A function could return a `std::lock_guard` object, thus allowing helper functions to initialize immobile objects.

There is one form of elision that guaranteed elision cannot cover: named return value optimization. Most compilers provide NRVO, but only under specific circumstances which differ from compiler to compiler. Even where they do so, the returned types need to have accessible copy/move constructors.

This is problematic for types with two-stage construction. `unique_lock` is a good example. If a lock times out, the `unique_lock` object lives on in an unlocked state. You can use member functions to try again.

Of course, `unique_lock` is by design moveable rather than immobile. But it would not be difficult to consider wanting to have the features of `unique_lock` while still having the static scoping of `lock_guard`. But if you want to write a helper function that returns a locked `unique_lock`, then you have to be able to create a named variable for that object.

# Design

What we want is the ability to declare a variable to actually *be* the function's return value object. This allows us to directly manipulate that object. That object shall be called a "return value variable" (RVV).

Note: the following uses a strawman syntax. It can be changed as needed.

## Variable declaration

A function-scoped variable can be given the return value storage class:

    return_value Typename variable_name;

This is a return value variable declaration. `variable_name` now represents the return value object. The storage duration for this object can extend beyond the instance of the function that created it. This is much like how `static` variables in function scope exist beyond the instance where they are first initialized.

RVV declarations may not use structured binding. [note: Is this strictly necessary? The `return auto;` syntax below allows returning the RVV object without explicitly naming it.]

## Scope

Once an RVV is declared within a block, all subsequent statements and sub-blocks within that block are said to be "within the scope of a return value variable", with the exception of any function declared within that scope. So the statements of a lambda expression after an RVV are not within the scope of an RVV.

[note: Lambdas can capture RVVs just like they can `static` local variables or any other object. But the fact that it is an RVV will not be preserved to the lambda, just like the fact that a captured `static` local variable is `static` local.]

If a declaration creates an RVV while within the scope of an RVV, the program is ill-formed. This includes multiple declarations on in the same statement.

If control flow exits the scope of a block containing an RVV declaration in any way other than by returning or throwing an exception, the program has undefined behavior. The idea behind this rule is that, in declaring a return value variable, you have made a promise to actually return from this function, and this object represents that return. And since that object would not be accessible outside of its scope, you're not allowed to leave that scope without returning.

RVVs represent the actual return value object. As such, when the scope containing an RVV exits by returning, the RVV is not destroyed.

However, if the scope exits due to throwing an exception, the RVV is destroyed along with automatic variables in its scope. The order of an RVV's destruction relative to automatic variables in the same scope is undefined [note: This undefined order is probably OK semantically. After all, if you returned it, it wouldn't be deleted until after the other local variables were all cleaned up, regardless of the order you declared them in.]

## RVV and Return Types

A function may only have an RVV declaration if the function's return type is a value type or `auto` (never for `decltype(auto)`).

For functions that use the placeholder `auto` as its return type, each RVV declaration contributes to the deduction of the return value type. The type of the RVV (whether deduced from the initializer or explicitly specified) is the type the RVV contributes for return value deduction.

If a function containing an RVV declaration uses an explicit type for its return type, then each RVV declaration must have a type which exactly matches the specified return type, including cv-qualification. If a placeholder+initializer is used in the RVV declaration, then it must deduce the exact type of the specified return type, including cv-qualification.

To avoid unnecessary repetition of typenames, an RVV declaration may use a placeholder type *without* an initializer, but only if the function uses an explicit type for its return type. [note: It might also be OK in return type deduction cases if there are previous non-discarded statements that contribute to return type deduction].

## Returning RVVs

The statement `return auto;` can be used to return the RVV if an RVV is in scope of the statement. The statement `return variable_name;` will also return the RVV if `variable_name` names the RVV.

These are the only ways of returning an RVV. As such, if an RVV declaration is in scope, a `return` statement *must* be of one of the two forms mentioned above.

Since the RVV is the result object from a function, there is no copying or moving from the variable.

# Issues

Perhaps the biggest issue here is linguistic.

Guaranteed elision got away with what it did by redefining the concept of "prvalue". A prvalue is at present an initializer for an object.

What we are doing here is initializing an object in a conceptual sense, but unlike guaranteed elision, our "initialization" is not initialization as the standard defines it. It is much more than that: we actually manipulate the object after its creation. That is indeed the whole point.

Given this, what then is a prvalue? It has to be more than just an "initializer" for an object, since the object is initialized well before the function returns. Indeed, the object gets a name (the RVV), then loses that name as it becomes a prvalue, then possibly gets a new name. Indeed, even that new name might be an RVV, which will be lost again.

This is very new for C++ from a conceptual sense in terms of the object model. From a compiler implementation standpoint, this ought to be pretty easy.

# Change-list

# Acknowledgments
