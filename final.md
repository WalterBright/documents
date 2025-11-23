# Static Single Assignment (SSA)

| Field           | Value                                                           |
|-----------------|-----------------------------------------------------------------|
| DIP:            | (number/id -- assigned by DIP Manager)                          |
| Review Count:   | 0 (edited by DIP Manager)                                       |
| Author:         | Walter Bright walter@walterbright.com                           |
| Implementation: | (links to implementation PR if any)                             |
| Status:         | Will be set by the DIP manager (e.g. "Approved" or "Rejected")  |

## Abstract

Static Single Assignment means that a declaration can be initialized, but its
value cannot be subsequently altered. The `final` storage class indicates a
SSA declaration. It is not a type constructor.

With modern compiler optimizers, SSA does not improve code generation.


## Contents
* [Rationale](#rationale)
* [Prior Work](#prior-work)
* [Description](#description)
* [Breaking Changes and Deprecations](#breaking-changes-and-deprecations)
* [Reference](#reference)
* [Copyright & License](#copyright--license)
* [Reviews](#reviews)

## Rationale

This makes code easier to understand, especially
complicated code, as it restricts the semantics of the assignment.

## Prior Work

[Single-Assignment C](https://en.wikipedia.org/wiki/SAC_programming_language)

[Rust](https://doc.rust-lang.org/book/ch03-01-variables-and-mutability.html)

## Description

`final` is introduced as a storage class for variable declarations. It is already
in the grammar for storage classes, https://dlang.org/spec/declaration.html#StorageClass,
but is currently diagnosed as an error when applied to variable declarations.

`final` means that the variable's contents, once initialized by the declaration,
can no longer be altered unless stepping outside of D semantics in `@system` code.
This requirement inevitably drives behaviors, as follows:

```d
final int f = 3;
f = 4; // error, f is final
int* p = &f; // error, cannot make mutable reference to final variable
const int* pc = &f; // ok, const will not allow alteration
```

`final` is not transitive. It does not work like `const`; another term for
it is "head const":

```d
int i;
final int* pi = &i;
*pi = 3; // ok, as the value of pi is not altered

const int c = 4;
final int* pc = &c; // error, c is const
```

A final declaration does not need an initializer; as default initialization
occurs:

```d
enum E { 1, 2 }
final E e; // initialized to 1
```

A final declaration with void initialization makes no sense and is not allowed:

```d
final int v = void; // error, cannot void initialize final declaration
```

A `ref` cannot be taken of a `final` declaration:
```d
final int i = 3;
ref int r = i; // error cannot make mutable reference to final `i`
const ref int cr = i; // ok
```

`final` does not affect the type:

```d
int i;
final pi = &i; // typeof(pi) is int*
```

`final` has no effect on a `ref` declaration, as `ref` cannot be rebound.

`final` can be applied to struct and class fields.

`final` can be applied to function parameters, but overloading is not affected.
Hence, it has no effect on name mangling.

`final` cannot be applied to template parameters.

`final` applied to a declaration of a variable of `class` type means the non-static fields of
the class can be modified, but the variable cannot be rebound to another class instance.

`final` applied to a declaration of a variable of `struct` type means the non-static fields of the
struct are implicitly final, and cannot be modified after initialization.

## Breaking Changes and Deprecations

There are no breaking changes nor deprecations.

## Reference

digitalmars.dip.ideas forum thread
https://www.digitalmars.com/d/archives/digitalmars/dip/ideas/Single_Assignment_1765.html

digitalmars.dip.development forum thread
https://www.digitalmars.com/d/archives/digitalmars/dip/development/First_Draft_Static_Single_Assignment_656.html

## Copyright & License
Copyright (c) 2025 by the D Language Foundation

Licensed under [Creative Commons Zero 1.0](https://creativecommons.org/publicdomain/zero/1.0/legalcode.txt)

## Reviews
The DIP Manager will supplement this section with a summary of each review stage
of the DIP process beyond the Draft Review.
