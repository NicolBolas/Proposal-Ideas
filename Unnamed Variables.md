% Unnamed Variables
%
%

[EWG issue #35][ewg35] presents a problem that C++ has had need to solve for some time: declaring a variable without a name, which is used only for its constructor and (mainly) destructor. This proposal presents a way to make a small modification to an existing feature that makes this possible.

# Motivation

With RAII becoming more and more prevalent in C++ code, the need to create variables that are used only for their constructor and destructor increases. Scope guard types like `scope_exit`, `scope_fail`, and `scope_success` make this even more important. Consider the following simple boilerplate:

````
#define SCOPE_EXIT(f) auto _some_name_ = std::make_scope_exit(f);
````

Such a macro cannot be used multiple times in the same scope, as it would redefine the same variable name. Also, there is always the possibility that `_some_name_` is used elsewhere, creating a conflict there.

# Design

[EWG issue #35][ewg35] outlines some of the issues about the uniqueness of the name. Since that time, C++ gained a feature which is contingent on having variables with unique names within their contexts: [structured binding][sbinding], which is standardized under the concept of decomposition declarations.

Decomposition works by creating an unnamed variable, then introducing a number of names that represent operations on that variable: accessing one of its members or calling `get<N>` on it. As such, the standard problem of having a variable with a unique name in its context has already been solved.

Indeed, even the syntax for such a concept seems adequate and easily identifiable for the needs of just creating an unnamed variable:

````
auto [] = <expr>;
````

So the design for this feature involves making two changes to the existing standard:

1. Change the grammar to make the *identifier-list* in a decomposition declaration optional rather than mandatory. This should include the grammar for *simple-declaration* as well as in *for-range-declaration*.

2. Change [dcl.decomp] to recognize the possibility of an empty *identifier-list*. In such a case, it only introduces a uniquely named variable `e`, with none of the other decomposition machinery. So [dcl.decomp]/2-4 would not apply.

## Potential Problems

At present, [dcl.decomp] has a compile-time prohibition on using a decomposition declaration if the number of elements in the type do not match the number of names. By adding this feature, we would effectively permit the use of such a declaration in a mis-matched fashion, but only by providing no identifiers.

In practice, this should not be too much of a problem. A user who writes `auto []` is far more likely to genuinely want to make an unnamed variable than they are to have done so accidentally.

## Possible Extensions

At present, one problem with decomposition declarations is that you cannot discard one of the names. If you want to skip one of the elements of the object, you still have to name it.

But we could change things to permit the following:

````
auto [first, [], third] = expr;
````

Since `auto []` already means "unnamed variable", it makes sense to a user that using `[]` in such a context would mean to not name that member.

This would also be a natural step towards multi-level decomposition:

````
auto [first, [first_first, first_second], third] = expr;
````

These are not part of the proposal; they are just notations for future directions of decomposition. They also point out how well this meshes with decomposition.

# Impact on the Standard

This should be a backwards compatible change. Implementing it ought to be easy, since the hard work is fielded by decomposition declarations.

# Acknowledgements


[ewg35]: https://cplusplus.github.io/EWG/ewg-active.html#35
[sbinding]: http://wg21.link/P0144
