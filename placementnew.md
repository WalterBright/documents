# Placement New Expression

| Field           | Value                                                           |
|-----------------|-----------------------------------------------------------------|
| DIP:            | (number/id -- assigned by DIP Manager)                          |
| Review Count:   | 0 (edited by DIP Manager)                                       |
| Author:         | Walter Bright walter@walterbright.com                           |
| Implementation: | (links to implementation PR if any)                             |
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
4. The CTFE engine also doesn't know what emplace does, and expands the templates
to simulate it, making simple operations slow and memory intensive
5. If one desires to use classes without the GC, such as in BetterC, it's just
awkward to use `emplace`.



## Prior Work

Druntime emplace:
https://dlang.org/phobos/core_lifetime.html

C++ placement new:
https://en.cppreference.com/w/cpp/language/new


## Description

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

If `Type` is a class, the `PlacementExpression` must produce a value of type `void[]`
and the length of the array must be of sufficient size to hold the class object.
The size of the memory object of `class Type` can be retrieved with the expression
`__traits(initSymbol, Type).length`.

Otherwise, the `PlacementExpression` must produce a value of type `Type*`. This must point to a memory
object into which will be placed the new'd object.

## Breaking Changes and Deprecations

No breaking changes or deprecations.

## Reference

## Copyright & License

Copyright (c) 2024 by the D Language Foundation

Licensed under [Creative Commons Zero 1.0](https://creativecommons.org/publicdomain/zero/1.0/legalcode.txt)

## Reviews
The DIP Manager will supplement this section with a summary of each review stage
of the DIP process beyond the Draft Review.
