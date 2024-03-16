# Implicit Conversion of Template Instantiations

| Field           | Value                                                           |
|-----------------|-----------------------------------------------------------------|
| DIP:            | (number/id -- assigned by DIP Manager)                          |
| Review Count:   | 0 (edited by DIP Manager)                                       |
| Author:         | Walter Bright walter@walterbright.com                           |
| Implementation: | (links to implementation PR if any)                             |
| Status:         | Will be set by the DIP manager (e.g. "Approved" or "Rejected")  |

## Abstract

`const(T[])` can be implicitly converted to and from `const(T)[]`.
But `const(X!T)` cannot be implicitly converted to and from `X!(const T)`.
The lack of this means a template cannot, for example, emulate the
behavior of a slice. This proposal aims to fix that.

Only templated structs and classes will be covered. There will not
be any changes to the syntax.

## Contents
* [Rationale](#rationale)
* [Prior Work](#prior-work)
* [Description](#description)
* [Breaking Changes and Deprecations](#breaking-changes-and-deprecations)
* [Reference](#reference)
* [Copyright & License](#copyright--license)
* [Reviews](#reviews)

## Rationale

This has been requested repeatedly for years.


## Prior Work

I am not aware of any.

There have been numerous suggestions to allow implicit conversions like
C++ does, i.e. with a special member function. I have consistently opposed them,
as the C++ experience with them is it makes overloads and some other constructions
incomprehensible. I don't see how they completely solve this particular issue, either.

## Description

```d
void moon1(const(int)[]);
void moon2(const int[]);

void sun()
{
    const(int)[] a;
    moon1(a);   // works
    moon2(a);   // works

    const int[] b;
    moon1(b);   // works
    moon2(b);   // works
}
```

However:

```d
struct X(T) { T[] t; }

void moon1(X!(const int));
void moon2(const X!int);

void sun()
{
    X!(const int) a;
    moon1(a);   // works
    moon2(a);   // fails

    const X!int b;
    moon1(b);   // fails
    moon2(b);   // works
}
```

It is reasonable that this should work analogously to the builtin array version.

It's been suggested that `X!(const T)` and `const(X!T)` actually be the same type.
This can work for trivial types, but will not work for more complex things.
For example:

```d
struct X(T)
{
    T t;
    void bar(T);
}
```
If X is instantiated with `X!(const int* p)`, then `bar` becomes `void bar(const int*)`.
But `const X!(int*)` will instantiate `bar` as `void bar(int*)`.
The function parameter types are different! The two instantiations cannot be the same type.
(Another word for this is the functions are not "covariant".)
Consider also that `const(int)[]` and `const(int)[]` are not the same type, either,
so it isn't necessary for `X!(const T)` and `const(X!T)` to be the same type.

It's only necessary that the two types are implicitly convertible to each other.
More specifically, they are implicitly convertible with qualifier conversion.
https://dlang.org/spec/function.html#function-overloading
Other implicit conversions will not be considered in this proposal.

The template struct or template class will be treated as a list of its members.
Only members that are fields or non-static functions are considered.
For each field, an implicit qualified conversionis tried.
For each function a test for covariance and contravariance is performed in a manner analogous
to how a derived function is determined to override its ancestor.
If all of these tests pass, then the template struct or template class type is considered
implicitly convertible with qualified conversions.

This is all completely consistent with how the type system currently works, which is critical
for a consistent type system. It maintains the guarantees D provides w.r.t. const guarantees,
etc., without any kludges or user intervention.


## Breaking Changes and Deprecations

Code that relies on this conversion not working, like is-expressions, __traits(compiles),
and function overloading, will now behave differently.

I am not aware of any existing code that would rely on this behavior.

## Reference

https://www.digitalmars.com/d/archives/digitalmars/D/sumtypes_for_D_366242.html

## Copyright & License
Copyright (c) 2024 by the D Language Foundation

Licensed under [Creative Commons Zero 1.0](https://creativecommons.org/publicdomain/zero/1.0/legalcode.txt)

## Reviews
