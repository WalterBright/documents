# Making printf @safe

| Field           | Value                                                           |
|-----------------|-----------------------------------------------------------------|
| DIP:            | (number/id -- assigned by DIP Manager)                          |
| Review Count:   | 0 (edited by DIP Manager)                                       |
| Author:         | Walter Bright walter@walterbright.com                           |
| Implementation: | (links to implementation PR if any)                             |
| Status:         | Will be set by the DIP manager (e.g. "Approved" or "Rejected")  |

## Abstract

`printf` is a very useful construct. It is probably the most optimized and debugged function
ever written. It is, however, not usable in `@safe` D code, as
it can be used in an unsafe manner. Many of these problems are mitigated by the addition
of checks for arguments that are mismatched with their corresponding format specifiers.
This proposal will resolve the remaining issues and make `printf` usable from `@safe`
code.


## Contents
* [Rationale](#rationale)
* [Prior Work](#prior-work)
* [Description](#description)
* [Breaking Changes and Deprecations](#breaking-changes-and-deprecations)
* [Reference](#reference)
* [Copyright & License](#copyright--license)
* [Reviews](#reviews)

## Rationale

`printf` is a workhorse function that has many advantages:

1. it is likely the most optimized and debugged non-trivial function ever written
2. it is very lightweight, in that minimal code is generated to call it
3. it works when nothing else does, as when porting D to a new platform
4. it works without need of druntime or Phobos
5. it works in BetterC mode
6. it is very well known

But, it has an unsafe interface. D is moving towards safe-by-default, and this makes
it clumsy to use printf. printf calls can be accepted within @safe code by prefixing
the call with `debug`, but that doesn't work when printf is desired in non-debug
builds.

The unsafe problems with printf are:

1. mismatch of arguments with format specifiers. These are detected by `pragma(printf)`.
2. the %s takes as an argument a pointer to a string. While the string is only read,
the pointer still walks the string in an unbounded manner
3. the %.*s parameter takes an argument of the form (int,char*). The int is the number
of characters to print, but a value < 0 has unsafe behavior
4. a format string that is not a literal and the compiler cannot check it

These are fixable problems, or can be constrained so they are memory safe.

## Prior Work

I am not aware of attempts to fix the safety problems of printf, other than writing a
replacement for it.


## Description

It will only come in play if the format string is a string literal.

### %s Format Specifier Rewrite

Currently, the compiler will check format specifiers against the argument types.
If there is a mismatch, an error is diagnosed. This proposal will cause the format
specifier to be rewritten to match the argument type, if the format specifier
is `%s`.

If the format specifier is `%s` and the corresponding argument is a D array of char
or wchar_t, the specifier will
be replaced with `%.*s` (or `%.*ls`) and the argument will be replaced with two arguments of
the form:

```
cast(int)argument.length & 0x7FFF_FFFF, argument.ptr
```

The mask ensures that the length is within the range where `.*s` works safely. If
the actual length exceeds 0x7FFF_FFFF, the entire string output will be truncated,
but at least it will be memory safe.

### sprintf, fprintf, snprintf, etc.

This DIP applies to any function marked with `pragma(printf)` and `@safe` or `@trusted`.


## Breaking Changes and Deprecations

No known breaking changes.

## Reference

digitalmars.dio.ideas forum thread:
https://www.digitalmars.com/d/archives/digitalmars/dip/ideas/Make_printf_safe_567.html


## Copyright & License
Copyright (c) 2024 by the D Language Foundation

Licensed under [Creative Commons Zero 1.0](https://creativecommons.org/publicdomain/zero/1.0/legalcode.txt)

## Reviews
The DIP Manager will supplement this section with a summary of each review stage
of the DIP process beyond the Draft Review.
