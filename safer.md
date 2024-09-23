# Safer D


| Field           | Value                                                           |
|-----------------|-----------------------------------------------------------------|
| DIP:            | (number/id -- assigned by DIP Manager)                          |
| Review Count:   | 0 (edited by DIP Manager)                                       |
| Author:         | Walter Bright walter@walterbright.com                           |
| Implementation: | (links to implementation PR if any)                             |
| Status:         | Will be set by the DIP manager (e.g. "Approved" or "Rejected")  |

## Abstract

For functions annotated with `@trusted`, or not annotated at all, enable
`@safe` checks that have simple, local fixes. This will be enabled via
the switch `-preview=safer`.


## Contents
* [Rationale](#rationale)
* [Prior Work](#prior-work)
* [Description](#description)
* [Breaking Changes and Deprecations](#breaking-changes-and-deprecations)
* [Reference](#reference)
* [Copyright & License](#copyright--license)
* [Reviews](#reviews)

## Rationale

Currently, D functions come in one of 4 categories:

1. default (no annotation)
2. `@safe`
3. `@trusted`
4. `@system`

With `@safe`, all memory safety checks are on. `@system` means all off.
`@trusted` also means all off, but with an interface that the user vouches
for as `@safe`. The default one is just like `@system`, but array bounds
checking is on.

Most code is likely written as the default, and the idea is to move such
code to `@safe`. The problem is that `@safe` is an all or nothing proposition.
For example, `@safe` functions cannot call default functions. They cannot
call printf or other C functions, as C functions do not default to being
`@safe` or `@trusted`. So converting code from default to `@safe` is an all or
nothing proposition, not a function-by=function one. This has proved to be
a significant barrier for those who wish to use the compiler to make their
code safer.

One mechanism is to mark all functions as `@trusted`, and then one by one convert
them to `@safe`. The problems with this are:

1. `@trusted` functions are supposed to have an `@safe` interface. Many are
uncomfortable with mispreresenting a function in this manner.
2. It's still an all or nothing proposition to convert such funtions to
`@safe`. One cannot call printf, for example.

Another mechanism is to mark a function as `@safe`, and then use an `@trusted`
lambda to wrap any unsafe code that cannot be easily converted. This technique
has been used extensively in the past, and has caused unhappiness due to its
(deliberately ugly) appearance. We've tried to move away from that. Another
weakness is a function marked `@safe` may not really be `@safe`, as the reviewer
would need to examine function body.

Others have proposed marking such pieces of code with a new construct, an
"unsafe" construct:

```
@safe void foo()
{
    unsafe { printf("my unsafe call"); }
}
```
but that remains ugly. It's also a blunt instrument, as parts of the unsafe
code that could be checked as `@safe` is not.

Checking for unsafe code by default will make existing code safer without
requiring the user to make the whole program `@safe`, misrepresent code
as `@trusted`, or re-engineer significant amounts of code.

## Prior work


## Description

Use of the `-preview=safer` switch will enable many safety checks that are
normally only done in functions marked `@safe`.

For example:
```
int battery(int[] a)
{
    int* p = a.ptr;
    return *p;
}
```
This could result in an invalid pointer value placed in `p` if the length
of array `a` is zero. This is detected in `@safe` code with the suggestion that
`a.ptr` be replaced with `&a[0]`, where a runtime array bounds check is
performed.

For example:
```
int acid()
{
    int* p;
    {
	int i;
	p = &i;
    }
    return *p;
}
```
This is a code bug, as p has a longer lifetime than `i`, and when `i` goes out
of scope then `p` will be pointing to an invalid value. This is detected in
`@safe` functions, but not in default functions.
(The fix is to move the declaration of `i` prior to the declaration of `p`.)

All the checks currently enabled in `@safe` code, that are easily fixed (as in
the fix is constrained to the function), will be enabled in `-preview=safer`
code.

Code not easily fixed, such as calls to `@system` or default functions, will be
allowed as before.

This does not affect name mangling or the interface to the function.


## Breaking Changes

The breaking changes are the obvious new errors being detected. But since
the name mangling or the interface to the function does not change, that
should be the extend of it.

Of course, since it is a global compiler switch (not a separate annotation)
all default functions will be checked that the compiler needs to compile.
This means existing libraries will need to be upgraded for use by users
wanting safer code. A possible accommodation for this would be only source
files explicitly listed on the command line would be subject to the
`-preview=safer` switch.

## Reference

## Copyright & License
Copyright (c) 2024 by the D Language Foundation

Licensed under [Creative Commons Zero 1.0](https://creativecommons.org/publicdomain/zero/1.0/legalcode.txt)

## Reviews

