# Placement New Expression

| Field           | Value                                                           |
|-----------------|-----------------------------------------------------------------|
| DIP:            | (number/id -- assigned by DIP Manager)                          |
| Review Count:   | 0 (edited by DIP Manager)                                       |
| Author:         | Walter Bright walter@walterbright.com inspired by Manu Evans    |
| Implementation: | https://github.com/dlang/dmd/pull/17057                         |
| Status:         | Will be set by the DIP manager (e.g. "Approved" or "Rejected")  |

## Abstract

Operator new currently only allows for allocation on the GC heap. With
the placement new expression, operator new can initialize an object into
any location. It replaces the functionality of core.lifetime.emplace.

## Contents
* [Rationale](#rationale)
* [Prior Work](#prior-work)
* [Description](#description)
* [Breaking Changes and Deprecations](#breaking-changes-and-deprecations)
* [Reference](#reference)
* [Copyright & License](#copyright--license)
* [Reviews](#reviews)

## Rationale

The existing operator new allocates on the GC heap, or, if it is initializing
a class object that has the `scope` attribute, on the stack. It cannot be used
with storage allocated by malloc() or any other storage allocator. Nor can it
be used to initialize slots in an array.

To solve this problem, core.lifetime.emplace was created. It works. But it has
some shortcomings:

1. it is complicated
2. it simulates what the compiler does to initialize objects, such as:

```
core.lifetime.copyEmplace(s, t);
```
simulates:
```
S s = t;
```
3. The compiler doesn't know what emplace does, and so cannot make use of
information like lifetimes.
4. `emplace` generates template bloat.
5. If one desires to use classes without the GC, such as in BetterC, it's just
awkward to use `emplace`.



## Prior Work

The old "class allocators":
https://dlang.org/deprecate.html#Class%20allocators%20and%20deallocators

Druntime emplace:
https://dlang.org/phobos/core_lifetime.html

C++ placement new:
https://en.cppreference.com/w/cpp/language/new


## Description

Placement new explicitly provides the storage for NewExpression to initialize with the newly created
value, rather than using the GC.

The current grammar for NewExp is:

```
NewExpression:
    new Type
    new Type [ AssignExpression ]
    new Type ( NamedArgumentList opt )
    NewAnonClassExpression

NewAnonClassExpression:
    new class ConstructorArgs opt AnonBaseClassList opt AggregateBody
```
This will be amended to include an optional `PlacementExpression` prefix:
```
NewExpression:
    new PlacementExpression opt Type
    new PlacementExpression opt Type [ AssignExpression ]
    new PlacementExpression opt Type ( NamedArgumentList opt )
    NewAnonClassExpression

NewAnonClassExpression:
    new PlacementExpression opt class ConstructorArgs opt AnonBaseClassList opt AggregateBody

PlacementExpression:
    ( AssignExpression )
```

If `Type` is a basic type or a struct, the `PlacementExpression` must produce an lvalue that has a size larger or
equal to `sizeof(Type)`.

```
struct S
{
    float d;
    int i;
    char c;
}

void main()
{
    S s;
    S* p = new (s) S();
    assert(p.i == 0 && p.c == 0xFF);
}
```

If `Type` is a class, the `PlacementExpression` must produce a value of type that is of a sufficient
size to hold the class object such as `void[__traits(classInstanceSize, Type)]`.

```
class C
{
    int i, j = 4;
}

void main()
{
    void[__traits(classInstanceSize, C)] k;
    C c = new(k) C;
    assert(c.j == 4);
    assert(cast(void*)c == cast(void*)k.ptr);
}
```

Placement `new` cannot be used for associative arrays, as associative arrays are designed to be
on the GC heap. The size of the associative array allocated is determined by the runtime library,
so cannot be set by the user.

The use of placement new is not allowed in `@safe` code.

To allocate storage with an allocator function such as `malloc()`, a simple template can
be used:

```
import core.stdc.stdlib;

struct S { int i = 1, j = 4, k = 9; }

ref void[T.sizeof] mallocate(T)() {
    return malloc(T.sizeof)[0 .. T.sizeof];
}

void main() {
    S* ps = new(mallocate!S()) S;
    assert(ps.i == 1);
    assert(ps.j == 4);
    assert(ps.k == 9);
}
```

## Breaking Changes and Deprecations

No breaking changes or deprecations.

## Reference

## Copyright & License

Copyright (c) 2024 by the D Language Foundation

Licensed under [Creative Commons Zero 1.0](https://creativecommons.org/publicdomain/zero/1.0/legalcode.txt)

## Reviews
The DIP Manager will supplement this section with a summary of each review stage
of the DIP process beyond the Draft Review.
