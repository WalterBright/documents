# Reducing the legal permutations of ref-return-scope

| Field           | Value                                                           |
|-----------------|-----------------------------------------------------------------|
| DIP:            | (number/id -- assigned by DIP Manager)                          |
| Review Count:   | 0 (edited by DIP Manager)                                       |
| Author:         | Walter Bright walter@walterbright.com                           |
| Implementation: | (links to implementation PR if any)                             |
| Status:         | Will be set by the DIP manager (e.g. "Approved" or "Rejected")  |

## Abstract

The `ref`, `return` and `scope` can be applied to a function parameter or a member
function or delegate in any order. This DIP makes illegal any ordering that is not
described in the Specification.



## Contents
* [Rationale](#rationale)
* [Prior Work](#prior-work)
* [Description](#description)
* [Breaking Changes and Deprecations](#breaking-changes-and-deprecations)
* [Reference](#reference)
* [Copyright & License](#copyright--license)
* [Reviews](#reviews)

## Rationale

The trouble is it's forgettable which order means
what. The Specification is careful to use only one ordering in its examples.
By constraining the ordering to what's in the Specification, it will be easier
to select the intended behavior, and easier to search for examples.


## Prior Work

Previously, `scope return` was given special meaning. The other orderings were left
alone for the purpose of legacy compatibility.

## Description

For a parameter to be returning a ref, `return ref` will be required. Constructions like
`ref return`, `return const ref`, etc. will no longer be accepted.


## Breaking Changes and Deprecations

This will break existing code. Hence enabling it will be behind a `-preview=return` switch,
and will be the default in a future edition.

## Reference

https://dlang.org/spec/function.html#return-ref-parameters

https://dlang.org/spec/function.html#return-scope-parameters

https://dlang.org/spec/function.html#ref-return-scope-parameters

Laying some groundwork for the implementation:
https://github.com/dlang/dmd/pull/21369

## Copyright & License
Copyright (c) 2025 by the D Language Foundation

Licensed under [Creative Commons Zero 1.0](https://creativecommons.org/publicdomain/zero/1.0/legalcode.txt)

## Reviews
The DIP Manager will supplement this section with a summary of each review stage
of the DIP process beyond the Draft Review.
