# __rvalue and Move Semantics

| Field           | Value                                                           |
|-----------------|-----------------------------------------------------------------|
| DIP:            | (number/id -- assigned by DIP Manager)                          |
| Review Count:   | 0 (edited by DIP Manager)                                       |
| Author:         | Walter Bright walter@walterbright.com, Timon Gehr, Manu Evans   |
| Implementation: | https://github.com/dlang/dmd/pull/17050                         |
| Status:         | Will be set by the DIP manager (e.g. "Approved" or "Rejected")  |

## Abstract

Adds the language intrinsic `__rvalue(Expression)` indicating that `Expression` is
to be considered an rvalue for matching with function parameters. This enables
move semantics when calling a function with non-ref semantics.

## Contents
* [Rationale](#rationale)
* [Prior Work](#prior-work)
* [Description](#description)
* [Breaking Changes and Deprecations](#breaking-changes-and-deprecations)
* [Reference](#reference)
* [Copyright & License](#copyright--license)
* [Reviews](#reviews)

## Rationale

Move semantics are desirable for runtime and resource efficiency, since a resource
can be moved to a new object rather than duplicated and then destroyed.
They've been requested for inclusion several times. Other languages,
such as C++, have move semantics which are popular.

## Prior Work

C++ Move Semantics https://medium.com/@daijue/c-move-semantics-30faf1e855cd

Move Semantics Rust vs C++ https://users.rust-lang.org/t/move-semantics-rust-vs-c/61274

C++ Move Semantics Considered Harmful (Rust Is Better) https://news.ycombinator.com/item?id=33702435


## Description

### Grammar

An `RvalueExpression` is an additional `PrimaryExpression`:

```
RvalueExpression:
    __rvalue ( AssignExpression )
```

### Overloading

If both a ref and non-ref parameter overloads are present, an rvalue is preferably matched to the
non-ref parameters, and an lvalue is preferably matched to the ref parameter.
An `RvalueExpression` will preferably match with the non-ref parameter.

### Semantics of Arguments Matched to Rvalue Parameters

An rvalue argument is considered to be owned by the function called. Hence, if an lvalue is matched
to the rvalue argument, a copy is made of the lvalue to be passed to the function. The function
will then call the destructor (if any) on the parameter at the conclusion of the function.
An rvalue argument is not copied, as it is assumed to already be unique, and is also destroyed at
the conclusion of the function. The destruction is automatically appended to the function body
by the compiler.

The function cannot know if its parameter originated as an rvalue or is a copy of an lvalue.

This means that an __rvalue(lvalue expression) argument destroys the expression upon function return.
Attempts to continue to use the lvalue expression are invalid. The compiler won't always be able
to detect a use after being passed to the function, which means that the destructor for the object
must reset the object's contents to its initial value, or at least a benign value.

```
struct S
{
    ubyte* p;
    ~this()
    {
      free(p);
      // add: `p = null;` here
    }
}

void aggh(S s)
{
}

void oops()
{
    S s;
    s.p = cast(ubyte*)malloc(10);
    aggh(__rvalue(s));
    free(s.p); // error, double free of s.p
}
```

### Move Assignment With __rvalue

```
struct S
{
    ubyte* p;
    ~this()
    {
      free(p);
      p = null;
    }
    void opAssign(S s)
    {
        this.p = s.p;
        s.p = null;
    }
}

void oops()
{
    S s;
    s.p = cast(ubyte*)malloc(10);
    S t;
    t = __rvalue(s);
}
```

### Move Construction With __rvalue

```
struct S
{
    ubyte* p;
    ~this()
    {
      free(p);
      p = null;
    }
    this(S s)
    {
        this.p = s.p;
        s.p = null;
    }
}

void oops()
{
    S s;
    S t = __rvalue(s);
}
```

## Breaking Changes and Deprecations

No changes to the language, existing code will work as it always has.

## Reference
Optional links to reference material such as existing discussions, research papers
or any other supplementary materials.

## Copyright & License
Copyright (c) 2024 by the D Language Foundation

Licensed under [Creative Commons Zero 1.0](https://creativecommons.org/publicdomain/zero/1.0/legalcode.txt)

## Reviews
The DIP Manager will supplement this section with a summary of each review stage
of the DIP process beyond the Draft Review.
