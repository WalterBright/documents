# ref For Variable Declarations

| Field           | Value                                                           |
|-----------------|-----------------------------------------------------------------|
| DIP:            | (number/id -- assigned by DIP Manager)                          |
| Review Count:   | 0 (edited by DIP Manager)                                       |
| Author:         | Walter Bright walter@walterbright.com                           |
| Implementation: | (links to implementation PR if any)                             |
| Status:         | Will be set by the DIP manager (e.g. "Approved" or "Rejected")  |

## Abstract

Enable local variables do be declared as `ref`.


## Contents
* [Rationale](#rationale)
* [Prior Work](#prior-work)
* [Description](#description)
* [Breaking Changes and Deprecations](#breaking-changes-and-deprecations)
* [Reference](#reference)
* [Copyright & License](#copyright--license)
* [Reviews](#reviews)

## Rationale

Ref declarations are a restricted form of pointer declarations. They are restricted
in that:

1. they cannot reassigned after initialization
2. pointer arithmetic cannot be done on them
3. they cannot be copied, other than being used to initialize another ref

Decades of successful use have clearly demonstrated the utility of this form of
restricted pointer. Ref declarations are a major tool for writing memory safe code,
and self-documenting the restricted use of being a ref rather than a pointer.

Currently, ref declarations are only allowed for:

1. function parameters
2. function return values
3. declaration references to element types in a foreach statement

Of particular interest here is the success of item (3). There doesn't appear to
be a downside of using `ref` for such declarations, so by extension ordinary
local variables should also benefit from being declared as `ref`. Often I've run
across cases where declaring a local as `ref` rather than a pointer would have
made the code nicer and safer.


## Prior Work

C++ allows variables to be declared as a reference rather than pointer. The same
goes for ref struct field declarations, although that is not part of this proposal.


## Description

`ref` is already allowed in the grammar for variable declarations, it is disallowed
in the semantic pass.
https://dlang.org/spec/declaration.html#VarDeclarations
This proposal will "turn it on" so to speak, and its behavior will be the same as
the current behavior for foreach ref parameters.

Returning a ref variable from a function that returns a ref is not allowed. I.e.
a ref cannot be assigned to a ref with a scope that exceeds the scope of the source.


```
ref int dark(ref int x, int i)
{
    ref j = i;     // j now points to i
    j = 3;         // now i is 3 as well

    ref int y = x; // ok
    if (i)
        return x;  // ok
    else
        return y;  // nope
}
```


C++ rvalue references are not part of this proposal.


## Breaking Changes and Deprecations

No breaking changes are anticipated.

## Reference

References in D:
https://dlang.org/spec/declaration.html#ref-storage

Foreach ref parameters:
https://dlang.org/spec/statement.html#foreach_ref_parameters

References in C++:
https://en.cppreference.com/w/cpp/language/reference

## Copyright & License
Copyright (c) 2024 by the D Language Foundation

Licensed under [Creative Commons Zero 1.0](https://creativecommons.org/publicdomain/zero/1.0/legalcode.txt)

## Reviews
