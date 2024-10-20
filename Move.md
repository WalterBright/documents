
# Move Constructors and Move Assignment for D

by Walter Bright

The usual semantics in programming languages, include C, D, C++, Javascript, Java,
etc., is that when an object is assigned to a new location, a new object is created
and a copy is made (copy construction), or an existing object is overwritten with a copy of the source
object (copy assignment). This works well nearly all the time.

But, if the object contains a resource (such as allocated memory), copying the object
requires also making a copy of the allocated memory. A move, rather than a copy, moves
the resource to the new location. The obvious advantage of this is minimizing the use
of resources and minimizing the compute time.

This article presents a method in D for doing move construction and move assignment.
It only applies to structs, as there is no purpose for using moves instead of copies
for arithmetic types, pointers, and class objects.

What D currently has:
```
struct S
{
    P payload;

    this(ref S);      // copy constructor
    void opAssign(S); // copy assignment
    ~this();          // destructor
}
...
S t = s; // invoke copy constructor
S a;     // default initialization (a is set to a default value)
a = s;   // invoke opAssign
```

## Move Constructor

Moving a value from an existing object (source) into a new object (target) consists of:

1. default initializing the target
2. move constructing each field from the source to the target
3. default initializing the source

After this operation is complete, the lifetime of the source has ended, as it is
supposedly the "last use" of the source. However, the
compiler currently cannot reliably determine last use.
Default initialization of the source ensures that if/when a destructor is called
on the source, nothing happens. If a destructor is never called on it, that is
also benign as any destruction responsibility has moved to the target.

### Making Move Construction Work

The obvious method of making this work is to define a move constructor as:

```
this(S); // move constructor
```
as the parameter is inherently an rvalue, that's the only copy of it in existence,
so of course it should be moved rather than copied into the target. But this
runs aground with the following problems:

1. overloading `this(S)` with `this(ref S)` means that which one is selected
depends on details of the argument. This is also influenced by the `-preview=rvaluerefparam`
switch. The unhappy aspect of this is it is hard to determine which overload
will be selected, meaning it is hard to tell if it is a move or a copy.
From a comprehensibility point of view, this is not a good idea.

2. since `this(S)` requires an rvalue argument, a copy is made of the source and
then (under the hood) that copy is passed to the move constructor by ref. But wait,
what, a copy is made? Yes. There goes the entire point of having a move constructor
in the first place.

3. consider:
```
S t = s;
```
`s` is both an lvalue and an rvalue. How does one select between copying and moving?
The answer is to add an intrinsic, lets' call it `__rvalue`:
```
S t = __rvalue(s);
```
which tells the compiler to treat `s` as an rvalue.

This is not looking good. We still wind up with extra copies. Time for a new strategy.

Taking inspiration from __rvalue(S), specify a move constructor as:
```
struct S
{
    S opMove();
}
```
where the `this` represents the source, and the return value represents the target.
A move construction would look like:
```
S t = s.opMove();
```
The function body for it looks like:
```
struct S
{
    S opMove()
    {
        S s;                                                  (1)
        s.payload.opMoveAssign(payload);                      (2)
        memcpy(&this, __traits(initSymbol, S).ptr, S.sizeof); (3)
        return s;                                             (4)
    }
}
```
Stepping through the function:

1. `s` is default initialized
2. the source payload is moved to the target payload (we'll explain
`opMoveAssign` later)
3. `s` is default initialized
4. `s` is returned

But wait! Isn't `s` then copied to the target after it returns? Nope, it doesn't
by the magic of NRVO (Named Return Value Optimization).
The `s` is constructed directly into the target.

This is maximally efficent as there are no extra copies.

## Move Assignment

Move assignment is a bit more complex. A member function is defined:

```
struct S
{
    void opMoveAssign(ref S s);
}
```
where `this` represents the target (unlike opMove) and the parameter `s`
represents the source. The function body looks like:
```
struct S
{
    void opMoveAssign(ref S s)
    {
        this.__dtor();                                     (1)
        this.payload.opMoveAssign(s.payload);              (2)
        memcpy(&s, __traits(initSymbol, S).ptr, S.sizeof); (3)
    }

}
```
Stepping through the function:

1. destroy the previous contents of the target by running the destructor on it
2. move assign each field from source to target
3. default initialize the source

The astute reader will notice that the big difference between `opMove` and
`opMoveAssign` is the latter has to destruct the target rather than default
initialize it. Destructing the target is less efficient than default initializing
it, so why is opMove calling `opMoveAssign` for its fields rather than `opMove`?
The answer is the function prototype for `opMove` is not conducive to its use on
a field rather than a variable.

But, since this code is written by the user, it can be hand optimized as needed.

## Putting it together

```
import core.stdc.stdio;
import core.stdc.string;

struct T
{
    int i = 2;

    ~this() { printf("T.~this()\n"); }

    this(ref T t) { i = t.i; printf("T.copy\n"); }

    void opAssign(T t) { printf("T.opAssign()\n"); }

    void opMoveAssign(ref T r)
    {
        printf("S.opMoveAssign\n");
        __dtor();       // destruct destination first
        i = r.i;

        // reset source to .init value
        memcpy(&r, __traits(initSymbol, T).ptr, T.sizeof);
    }

    T opMove()
    {
        printf("T.opMove %d\n", i);
        T t;
        t.i = i;

        // reset source to .init value
        memcpy(&this, __traits(initSymbol, T).ptr, T.sizeof);
        return t;
    }
}

struct S
{
    T payload;

    ~this() { printf("S.~this()\n"); }

    this(ref S s) { payload = s.payload; printf("S.copy\n"); }

    void opAssign(S s) { printf("S.opAssign()\n"); }

    void opMoveAssign(ref S r)
    {
        printf("S.opMoveAssign\n");
        __dtor();       // destruct destination first

        payload.opMoveAssign(r.payload);

        // reset source to .init value
        memcpy(&r, __traits(initSymbol, S).ptr, S.sizeof);
    }

    S opMove()
    {
        printf("S.opMove %d\n", payload.i);
        S s;
        s.payload.opMoveAssign(payload);

        // reset source to .init value
        memcpy(&this, __traits(initSymbol, S).ptr, S.sizeof);
        return s;
    }
}

S foo(ref S r)
{
    S s = r.opMove();
    s.payload.i += 5;
    return s;
}

void main()
{
  printf("test1\n");
  {
    S s;
    S r;
    r.opMoveAssign(s);
    r.payload.i = 3;
    printf("payload: %d\n", r.payload.i);
  }
  printf("test2\n");
  {
    S r;
    S t = foo(r);
    printf("payload: %d\n", t.payload.i);
  }
}

```

## Future Directions

1. have the compiler generate default versions of these functions, as it does
for `opAssign`.

2. have the compiler do Data Flow Analysis to determine last use, and hence be able to
(sometimes) elide the eventual destructor call on the source.

3. invent a new operator for `move`, so the `opMove` and `opMoveAssignment` calls can be
replaced with syntactic sugar.

## References:

* [NRVO](https://en.wikipedia.org/wiki/Copy_elision) and [Glossary](https://www.digitalmars.com/d/2.0/glossary.html)
