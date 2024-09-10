# Sum Types

| Field           | Value                                                           |
|-----------------|-----------------------------------------------------------------|
| DIP:            | (number/id -- assigned by DIP Manager)                          |
| Review Count:   | 0 (edited by DIP Manager)                                       |
| Author:         | Walter Bright walter@walterbright.com                           |
| Implementation: | (links to implementation PR if any)                             |
| Status:         | Will be set by the DIP manager (e.g. "Approved" or "Rejected")  |

## Abstract

Describes a builtin sum type (aka Tagged Union) addition to D.

## Contents
* [Rationale](#rationale)
* [Prior Work](#prior-work)
* [Description](#description)
* [Breaking Changes and Deprecations](#breaking-changes-and-deprecations)
* [Reference](#reference)
* [Copyright & License](#copyright--license)
* [Reviews](#reviews)


## Rationale

Sum types have proven to be useful and popular in many languages.
Sum types work well with pattern matching, which have also proven to be useful and popular.
The primary feature of sum types, as opposed to C-style untagged unions, is the guaranteed safe access to its members.
Handling all cases of a sumtype can also be guaranteed.
Sumtypes are a robust way to prevent access to a value without first checking if it is
invalid, which is a common source of programming errors.
Sumtypes are an alternative to throwing exceptions when functions need a way to indicate
that an error occurred and the return result is is not valid.



##Prior Work

### Phobos [std.sumtype](https://dlang.org/phobos/std_sumtype.html)

This is an excellent implementation that works with the limitations of
D's expressiveness, but has [limitations](https://chadaustin.me/2015/07/sum-types/)
stemming from needing to use types to specify the cases of a sum type.

* std.sumtype cannot include regular enum members
* std.sumtype cannot optimize the tag out of existence, for example, when having:
<pre>
enum Option { None, int* Ptr }
</pre>
* cannot produce compile time error if not all the arms are accounted for in a pattern match
rather than a thrown exception
* cannot extract sub-parts of a match and assign them to variables
* matches are by type only, not by value
* an int and a pointer cannot both be in a sumtype and be safe
* cannot have two field declarations be of the same type


### Swift [Enumerations](https://docs.swift.org/swift-book/LanguageGuide/Enumerations.html#//apple_ref/doc/uid/TP40014097-CH12-ID146)

### Rust [Enumerations](https://doc.rust-lang.org/reference/items/enumerations.html)

### C++ [boost.variant](http://www.boost.org/doc/libs/1_58_0/doc/html/variant.html)

## Description

A sum type has a set of uniquely named members.
Only one of these members can exist in it at any one time.
Which one is existing is specified by a hidden field called a tag.

Sum types have characteristics of enums, structs, and unions. It's like an enum in
that it can enclose a set of named integer values. It's like a struct in that it
contains two fields, a tag and a union of all the members of the sum type. It's
like a union in that the second field is shared by all the members.

The size of the sum type is the size of the tag plus the size of
the largest member plus any alignment padding.

Member functions of field declarations are restricted the same way union member functions are.

Members of a sum type can either have integer values assigned sequentially by the
implementation, unique integer values assigned by the programmer, or be values of a specified type.

The first member of the sumtype will be the default initializer for it.

Sumtypes can be copied.

Members of sumtypes cannot have copy constructors, postblits, or destructors.

The type of the EnumMember of the sumtype is the smallest integer type that can
represent all of the EnumMember values.
An unsigned type will be used if there are no negative values.

A special case of sumtypes will enable use of non-null pointers.

A new expression, QueryExpression, is introduced to enable querying a sumtype to
see if it contains a specified member.

###Grammar

<pre>
SumTypeDeclaration:
    `sumtype` Identifier SumTypeBody
    SumTypeTemplateDeclaration

SumTypeBody:
    `{` SumTypeMembers `}`

SumTypeMembers:
    SumTypeMembers
    SumTypeMember `,`
    SumTypeMember `,` SumTypeMembers

SumTypeMember:
    EnumMemberAttributes SumTypeMember
    EnumMember
    FieldDeclaration

EnumMember:
    Identifier
    Identifier = AssignExpression

FieldDeclaration:
    Type Identifier
    Type Identifier = AssignExpression

SumTypeTemplateDeclaration:
    `sumtype` Identifier TemplateParameters Constraint (opt) SumTypeBody

QueryExpression:
    `?` PostfixExpression `.` Identifier
</pre>

#### Alternative Syntax

<pre>
sumtype Option(T) = None | Some(T);
</pre>

as opposed to the proposed:

<pre>
sumtype Option(T) { None, Some(T) }
</pre>

###Examples

A function return could have two things returned, one signifies an error,
the other the result. This can be implemented as:

<pre>
sumtype Result
{
    Error,
    int Value
}

Result func()
{
    Result res;  // initialized to Result.Error
    if (things went well)
        res.Value = 25; // the non-error result
    return res;
}

void plugh()
{
    Result ret = func();
    if (?ret.Error)
        printf("error returned\n");
    else
        printf("result is %d\n", ret.Value);
}
</pre>


If an `int` is stored in `Kod`, only an `int` can be extracted
(this is done with a runtime check of the tag).
The same for an `int*`. Thus, while the `int` and
the `int*` occupy the same storage, they cannot be conflated.
The size of `Kod` will consume space for 2 `int`s - one for the tag, the other for `x` and `y`.

<pre>
enum Kod
{
    int x;
    int* p;
}
</pre>

###Templates

SumTypes can be parameterized like struct declarations:

<pre>
enum Option(T)
{
    int Integer,
    T Other,
}
</pre>

###Query Expression

The Query Expression returns a `bool` `true` if the SumType contains
the specified variant, `false` if not.

<pre>
x = Xyzzy.busy;
if (?x.busy)
    writeln("busy");
else
    writeln("not busy");
</pre>

###Safety

The idea is to prevent attempts to access a value in the SumType that
is not the value that is actually in it (as specified by the tag).

####writes

Writing to a SumType is safe, because it overwrites any existing value
in it with the new value.

<pre>
enum Xyzzy { busy, int allow, int* ptr }

Xyzzy x;
x = Xyzzy.busy;     // ok
x = Xyzzy.allow(3); // ok
int i;
x = Xyzzy.ptr(&i);  // ok
</pre>

####reads

Only the value that is actually in the SumType can be read:

<pre>
Xyzzy x = Xyzzy.busy;
Xyzzy y = x.busy; // ok
y = x.allow;      // runtime error
y = ?x.allow ? x.allow : Xyzzy.busy;
x = Xyzzy.allow(3);
y = x.allow;      // ok
</pre>

####pointers and references

Pointers and references to SumType typed members is useful
for calling member functions those members.
This makes the following a problem:

<pre>
int i;
Xyzzy x = Xyzzy.allow(3);
int** p = &x.allow;  // ok
x = Xyzzy.busy;
*p = null;      // uh-oh
</pre>

The most pragmatic approach for now is to simply disallow taking the address
of or a reference to a member of a SumType in `@safe` code.

###Implicit Conversions and Casts

Sumtypes do not implicitly convert to any other types.
Nor can they be cast.


### Is Expressions

[IsExpressions](https://dlang.org/spec/expression.html#is_expression)
are extended with the keyword `sumtype` to identify a sumtype.


### Name Mangling

A TypeSubtype is added.

<pre>
TypeSubtype:
    `No` QualifiedName
</pre>


### Special Case for Non-Null Pointers

Sumtypes can be used to create a non-null pointer.
For example, the following
shares storage between the `Null` member and the `Ptr` member.
Thus, `Ptr` cannot be extracted without checking for `Null`,
`Pointer` is default initialized to `Null`.

<pre>
sumtype Pointer
{
    Null,
    int* Ptr,
}

Pointer nnp;   // default initialized to Pointer.Null
if (?nnp.Null) printf("null pointer\n"); // ok
int* p = nnp.Ptr;  // runtime error
int i;
nnp.Ptr = &i;
p = nnp.Ptr; // ok
</pre>

The problem here is:

<pre>
nnp.Ptr = cast(int*)null;
p = nnp.Ptr;  // compiles, and yet p is set to null
</pre>

The fix is to recognize a sumtype of the form "member that is 0" (let's call it `Null`)
followed by "pointer". The sumtype then merges the tag with the pointer. Hence,
if a null pointer is assigned to the sumtype, the sumtype is tagged with `Null`.

(Languages which don't allow null pointers at all don't need this special case.)

###Implementation in DMD

SumTypes share selected characteristics of `enums`, `struct`s, and `union`s.
Trying to implement it as a subtype of one of them likely is going to be too
complicated to be worthwhile. So, a new Type and a new Declaration construct
needs to be built for it.

But since a subtype with only enum members can be implemented as
an enum, the compiler should do that rewrite. Similarly, a SumType
with only one field declaration should be rewritten as a struct
(and the tag can be omitted).
Furthermore, a subtype with an enum member with a value of 0 and
a field declaration that is a pointer can be rewritten as just a
pointer.


###Pattern Matching

A pattern matching statement suitable for accessing SumTypes is
the subject of another DIP.


###Future Directions

Tuple declarations may be added in addition to enum members and field declarations.


## Breaking Changes and Deprecations

`sumtype` is already used as a module name in Phobos. We'll have to:

* find a different keyword
* change std.sumtype module name
* use a context sensitive keyword

## Reference

* [Wikipedia Tagged Union](https://en.wikipedia.org/wiki/Tagged_union)
* [Sum Types Are Coming](https://chadaustin.me/2015/07/sum-types/)

## Copyright & License
Copyright (c) 2022 by the D Language Foundation

Licensed under [Creative Commons Zero 1.0](https://creativecommons.org/publicdomain/zero/1.0/legalcode.txt)

## Reviews
The DIP Manager will supplement this section with a summary of each
review stage of the DIP process beyond the Draft Review.

