% Unnamed Variables
%
%

# Motivation

[EWG issue #35][ewg35] outlines the primary motivation for this feature. With RAII becoming more and more prevalent in C++ code, the need to create variables that are used only for their constructor and destructor increases.

With scope guard types like `scope_exit`, `scope_fail`, and `scope_success`, the ability to create a variable which has no name becomes even more important. It would be convenient to be able to write a macro to take care of the boilerplate:

````
#define SCOPE_EXIT(f) auto _some_name_ = std::make_scope_exit(f);
````

Unfortunately, such a macro cannot be used multiple times in the same scope, as it would redefine the same variable. Also, there is always the possibility that `_some_name_` is used elsewhere, creating a conflict.

# Design

[EWG issue #35][ewg35] outlines some of the issues about the uniqueness of the name. Since that time, C++ gained a feature which is contingent on having variables with unique names within their contexts: [structured binding][sbinding], which is standardized under the concept of decomposition declarations.

Decomposition works by creating an unnamed variable, then introducing a number of names that represent operations on that variable: accessing one of its members or calling `get<N>` on it.

As such, the standard burden of having an unnamed variable already exists. The domain of its uniqueness is well established and adequate for our needs.

Indeed, even the syntax for such a concept seems adequate and easily identifiable:

````
auto [] = <expr>;
````

So the design for this feature involves making two changes to the existing standard:

1. Change the grammar to make the *identifier-list* in a decomposition declaration optional rather than mandatory. This should include the grammar for *simple-declaration* as well as in *for-range-declaration*.

2. Change [dcl.decomp] to recognize the possibility of an empty *identifier-list*. In such a case, it only introduces a uniquely named variable `e`, with none of the other decomposition machinery. So [dcl.decomp]/2-4 would not apply.

## Potential Problems or Limitations

At present, [dcl.decomp] has a compile-time prohibition on using a decomposition declaration if the number of elements in the type do not match the number of names. By adding this feature, we would effectively permit the use of such a declaration in a mis-matched fashion, but only by providing no identifiers.

In practice, this should not be too much of a problem. A user who writes `auto []` is far more likely to deliberately have not used names for the members than to have accidentally not done so.

A potential limitation of this idea is that, if we used a different syntax, we could combine them with decomposition declarations in a way that is not possible now. For example, if `auto ><` was used to represent an unnamed variable, we could in theory permit this:

````
auto [first, ><, third] = expr;
````

This would be a way to skip elements in a decomposition, in addition to being a way to avoid naming a variable.

Of course, we could always permit `[]` to appear in a decomposition declaration as well:

````
auto [first, [], third] = expr;
````

Indeed, such syntax would be a natural part of permitting multi-level decompositions. As such, having `[]` act to discard a decomposed internal is good.

These are not part of the proposal; they are just notations for future directions of decomposition.

# Impact on the Standard

# Acknowledgements


[ewg35]: https://cplusplus.github.io/EWG/ewg-active.html#35
[sbinding]: http://wg21.link/P0144
