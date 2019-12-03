# MVP Support for Typed Functional Languages

Here we illustrate how the MVP would support a typed functional language.


## Uniform Representation

In order to support parametric polymorphism, the MVP needs to support an efficient uniform representation of the values of a typed functional language. Note, though, that this does not the MVP needs to provide an efficient uniform representation across all languages&mdash;the uniform representation can vary by language because polymorphic functions will only be applied to values from that language. Furthermore, values do not always need to be manipulated in the uniform representation&mdash;many implementations have found it more efficient to manipulate values in specialized representations based on static type information and then explicitly convert to and from the uniform representation when using polymorphic functions.

That said, it is still important that all values be expressible in some uniform representation.
But in designing this uniform representation there is something beyond efficiency that we should consider.
Just as WebAssembly works hard to develop a design that prevents websites from becoming reliant upon implementation details (like endianness), languages compiling to WebAssembly also do not want programs interacting with their compiled modules to become dependent on implementation details.
For example, reordering the cases of an algebraic data type is a semantics-preserving transformation in many languages, as is renaming a case (provided all match expressions and constructing expressions are similarly renamed), and so a language compiler wants to ensure that the resulting change in the compiled WebAssembly module is similarly semantics preserving.
If values can exit and enter the module, through say `anyref` in imports and exports, then this semantics includes the casting behavior of values.
One can prove that this has significant implications on casting between different algebraic data types in particular.

In the end, we consider three designs, one that supports OCaml's polymorphic structural and physical equality, one that supports OCaml's polymorphic physical equality, and one without polymorphic equality (such as Haskell and F#, which impose an additional constraint on the type parameter).
The last of these three is the least imposing upon the various features of typed functional languages, so we consider that case here in order to consider each of those features more independently of equality.
We will consider the other two designs after reviewing the major features.
Note, though, that we do not consider the challenges of supporting a lazy language in this example.

```
$uniform
:= scheme.new
              (extensible (cases $closure $other))
$other
:= scheme.new (parent implicit $uniform)
              (extensible flat)
```

This design says that everything is either a `$closure` or is efficiently castable to some other scheme (the cases of which are unknown and unbounded).
Closures are specially distinguished because they need to be cast in two ways: they need to be cast from the uniform representation to an arbitrary closure of a given surface-level function type, and they need to be cast from that surface-level function type to a particular kind of closure (e.g. one that has closed specifically over a surface-level integer variable).
The `flat` extensibility attribute is unable to support this particular form of two-tier casting, so `$closure` must use a different form of extensibility, which is why it is distinguished from `$other`.


## Closures

Closures are the instances inhabiting function types in functional languages.
They are generally represented with a pointer to the assembly code implementing the function being represented followed by the values of the local variables that were closed-over by the closure.
The code pointer generally accepts the closure instance itself as its first argument (in order to get access to the value of the closed-over local variables) followed by the arguments expected by the function type.

One challenge with closures is arity, a challenge that is exacerbated by the fact that WebAssembly is not expected to support mixed-arity code pointers anytime soon.
There are a few viable solutions to this challenge, but in the following we focus on the solution that bakes in a few arities (in this case 0, 1, and 2, though in reality it would likely be up through 6) and then has a miscellaneous case for arities 3 and up.

```
$closure
:= scheme.new (parent implicit $uniform)
              (extensible (cases $closure0 $closure1 $closure2 $closure3plus))
$closure0
:= scheme.new (parent explicit $closure)
              (field implementation (func (param (gcref $closure0)) (result (gcref $uniform)))
                     readable immutable)
              castable
              (extensible flat)
$closure1
:= scheme.new (parent explicit $closure)
              (field implementation (func (param (gcref $closure1) (gcref $uniform)) (result (gcref $uniform)))
                     readable immutable)
              castable
              (extensible flat)
$closure2
:= scheme.new (parent explicit $closure)
              (field implementation (func (param (gcref $closure2) (gcref $uniform) (gcref $uniform)) (result (gcref $uniform)))
                     readable immutable)
              castable
              (extensible flat)
$closure3plus
:= scheme.new (parent explicit $closure)
              (field arity (unsigned 8)
                     readable immutable)
              (field implementation (func (param (gcref $closure3plus) (gcref $uniform) (gcref $uniform) (gcref $uniform) (gcref $paramplus)) (result (gcref $uniform)))
                     readable immutable)
              castable
              (extensible flat)
$paramplus
:= scheme.new
              (field length arity (unsigned 8)
                     readable immutable)
              (field (indexed arity) param (gcref $uniform)
                     readable immutable)
              constructible
```

Notice that in all cases the `implementation` field is a function pointer whose first parameter is a `gcref` to a closure of the same arity.
The number of additional `gcref $uniform` parameters depends on the arity.
The `$closure3plus` scheme has an additional `arity` field that is used to manually cast to a specific arity so that the program would still behave the same even if the number of baked-in arities were to be changed.

The `implementation` of `$closure3plus` also accepts a `$paramplus` instance.
Because the `$paramplus` scheme has no parent or children, is entirely immutable, and has no notion of equality, the engine has a lot of freedom to optimize it.
For example, the pointer `0` could be used to indicate the empty array, and the low-bit of the pointer could be used to indicate if the array is singleton (in which case the high bits can be the value of the sole `param`) or is actually allocated on the heap.

Lastly, each of the closure schemes is `extensible flat`.
This enables each closure-allocation site in the source code to specify its own child scheme of the appropriate arity.
The function pointer in the `implementation` field can then cast the given closure argument to its specific associated scheme.
This both enforces the expectation that the `implementation` is only given associated closure instances, and grants the code referenced by `implementation` access to the fields of the closure that are specfic to that `implementation` (i.e. to the values of the specific local variables that are closed over in the corresponding source code).
Eventually we hope to remove the need for this dynamic cast, but verifying the safety of this pattern requires fancy types that are beyond the scope of the MVP.


## Parametric Polymorphism

There is a particular important detail in how the above design for closures interacts with parametric polymorphism and currying.
In this design, the surface-level type `int -> int -> int` is expected to be represented by a `gcref $closure2` because it has arity 2.
However, an expression of this type is a valid argument to a top-level polymorphic function of type `forall a. (int -> a) -> a`.
From that function's perspective, its input is a function of arity 1, not 2.
Thus this design expects the compiler to construct a `gcref $closure1` that wraps the `gcref $closure2`.

Although this design imposes overhead when switching between monomorphic and polymorphic code, many language implementations have found that the overhead is more than made up for by letting most code use specialized representations of values.
For example, a top-level function of type `float -> float` can use `float64`s in registers or on the stack rather than references to heap-allocated instances of the `$uniform` scheme.


## Primitive Types

Of course, because a function of type `float -> float` can be used as an argument to a polymorphic function of type `forall a. (a -> a) -> (a -> a)`, it is important that primitive types like `float` are indeed expressible in the `$uniform` scheme.
The following shows how to do so for both `int` and `float`:

```
$int
:= scheme.new (parent implicit $other)
              (field value (signed 32)
                     readable immutable)
              castable
              constructible
$float
:= scheme.new (parent implicit $other)
              (field value float64
                     readable immutable)
              castable
              constructible
```

Because both of these schemes are entirely `immutable` and permit no notion of equality, the engine can choose to pack common instances of these schemes into the pointer rather than allocate them on the heap.
Which of the two schemes gets packed will likely depend on the architecture (i.e. 32 vs. 64 bit) and the engine (e.g. pointer flagging vs. NaN boxing).
Thus this design enables uniform unboxed primitives where the boxing strategy adapts to the circumstances as appropriate.


## Algebraic Data Types

Typed functional languages are generally heavy users of algebraic data types, i.e. named sums of products.
The best known example of this is (linked) `list`:

```
$list
:= scheme.new (parent explicit $other)
              castable
              (extensible (cases $nil $cons))
$nil
:= scheme.new (parent implicit $list)
              castable
              constructible
$cons
:= scheme.new (parent implicit $list)
              (field head (gcref $uniform)
                     readable immutable)
              (field tail (gcref $list)
                     readable immutable)
              castable
              constructible
```

Note that the `head` field of `$cons` uses the `$uniform` scheme.
This permits polymorphic functions over the elements of `list`.
However, `$list` is specified to be an `explicit` child of `$other` (and consequently `$uniform`).
This means the engine can develop an especially packed representation for `$list` if it wants.
For example, the `0` pointer could denote `$nil`, a bit flag could be used to denote a `$cons` with a `$nil` tail (with the remaining bits denoting the head), and the non-singleton list could be denoted with an address of a `$cons` instance.
Notice that the `tail` field is specifically a `gcref $list`, so lists themselves would be able to internally use this packed representation.

When one `gcref.convert`s such a pointer to `$uniform`, the `nil` could still be packed, but now as some constant that distinguishes from nullary constructors of other algebraic data types, and the singleton `$cons` could be boxed onto the heap.
The heap representation of `$cons` would need to have a header with a constant that distinguishes it from binary constructors of other algebraic data types.


## References

So far we have no notion of equality.
But even Haskell permits equality on its mutable references.
As such, the scheme for references is designed as follows:

```
$ref
:= scheme.new (parent implicit $other)
              (field referent (gcref $uniform)
                     readable writeable mutable)
              castable
              constructible
              (equatable identity)
```

(As a side note, schemes for certain algebraic data types could use `case` equality along with `deep` equality for each case in order to utilize a more efficient equality check that the engine might provide.)


## Type Classes

It is worth noting that type-class evidence does not need to be representable within the uniform representation.
Instead, each type class can have its own associated scheme.
This scheme would have a field for each component provided by the type class.
In the case where one type class extends another, the scheme of the subclass would be a child of the superclass.
Because there is never a need to downcast type-class evidence, these schemes would be simply `extensible` and never `castable`.


## Polymorphic Physical Equality

OCaml provides polymorphic physical equality, i.e. an operator of type `forall a. a -> a -> bool`.
The idea is that this compares the packed-pointer representation of two values as an efficient sound (but incomplete) equality check.
Because packed-point representations vary by language implementation, the semantics of this equality is allowed to be implementation dependent.
The [documentation](http://caml.inria.fr/pub/docs/manual-ocaml/libref/Stdlib.html) of `Stdlib.(==)` specifies physical equality (refering to polymorphic structural comparison, which we discuss next):

> `e1 == e2` tests for physical equality of `e1` and `e2`.
> On mutable types such as references, arrays, byte sequences, records with mutable fields and objects with mutable instance variables, `e1 == e2` is true if and only if physical modification of `e1` also affects `e2`.
> On non-mutable types, the behavior of `( == )` is implementation-dependent; however, it is guaranteed that `e1 == e2` implies `compare e1 e2 = 0`.

The problem, of course, is that WebAssembly is specifically designed to ensure consistent semantics across platforms.
So we *cannot* expose the sort of packed-pointer comparison that this operator was designed for.
Instead, we must fix a semantics of equality satisfying the above specification that should be implementable fairly efficiently across all platforms.

In order to do so, we first take advantage of a specific detail in OCaml's polymorphic structural comparison.
In particular, structural comparison of functional values (i.e. closures) is *required* to throw an error.
We choose to extend this requirement to physical comparison in order to avoid exposing implementation details, such as the choice of whether or not to wrap closures in polymorphic function calls that affect the perceived arity (as discussed above).
In order to provide this semantics, we modify the `$closure` and `$other` schemes to be `castable` so that we can first case on whether a given `$uniform` instance is a closure or not.

We then need to modify the `$other` scheme.
The issue is that we do not want to use `identity` equality for all values as this would both prevent packing primitive values and expose implementation details such as when primivite values were newly allocated versus reused.
So we need to change `$other` to case on whether the value is primitive or not:

```
$other
:= scheme.new (parent implicit $uniform)
              castable
              (extensible (cases $primitive $nonprimitive))
              (equatable case)
$primitive
:= scheme.new (parent implicit $other)
              castable
              (extensible (cases $int $float ...))
              (equatable case)
$nonprimitive
:= scheme.new (parent implicit $other)
              castable
              (extensible flat)
              (equatable identity)
```

Note that the `$primitive` needs to enumerate all the primitive types in its `cases` clause so that the engine's implementation of equality on `$other` can case on the primitive type and adapt its implementation of `deep` equality in each case (since some primitive instances might be packed in the pointer versus allocated on the heap).
On the other hand, `$nonprimitive` simply imposes `identity` equality upon all of its instances, and as such at can have an arbitrary number of child schemes.

The primary downside of this design is that even nullary constructors of algebraic data types use `identity` equality.
The translator to WebAssembly can mitigate this problem by allocating each nullary constructor once as a global constant and then having all subsequent uses of that nullary constructor refer to that global constant.
This ensures instances of nullary constructors are unique, making `identity` equality essentially behave like `deep` equality. 
However, the WebAssembly engine does not know about this invariant, and as such packed-pointer representations cannot be optimized to the same way degree they could be for `$list` above.
This problem could be addressed by adding a `constructible unique` instantiation attribute that would ensure/enforce uniqueness, but this introduces a number of complexities to the design of WebAssembly (such as how to ensure/enforce uniqueness), and it is only useful in this particular case for just OCaml, which gets subsumed by OCaml's need for polymorphic structural comparison anyways.


## Polymorphic Structural Comparison

OCaml also provides a polymorphic structural-comparison operator.
This operator recurses through two data structures.
It is explicitly stated to have the potential to run forever due to cyclic data structures.
The intent here is *not* to make it so that the WebAssembly engine provides structural equality.
Instead, the intent here is to explore how to enable the operator to be compiled to WebAssembly.

The issue is that this operator was designed to piggyback upon a particular garbage-collection strategy.
This strategy relies on all data always being in a uniform representation so that everything in the heap is essentially an array of `gcref $uniform`.
This enables the heap to be recursively crawled, and its contents compared, without knowing what those contents actually describe.
However, there is no expectation that the MVP will provide such crawling functionality, and there are good reasons to never provide such crawling functionality (such as it potentially locking engines into a particular garbage-collection strategy).

So instead we must modify the `$nonprimitive` scheme to provide enough structure for the implementation of structural comparison to manually walk through the data itself:

```
$nonprimitive
:= scheme.new (parent implicit $other)
              castable
              (extensible (cases $record $adt))
              (equatable case)
```

This redefinition states that non-primitives are records or algebraic data types.
Records are OCaml's generalization of mutable references.

```
$record
:= scheme.new (parent implicit $nonprimitive)
              (field type_tag (gcref $record_type)
                     readable immutable)
              (field (length inclusive) count (unsigned 8)
                     readable immutable)
              (field (indexed count) member (gcref $uniform)
                     readable writeable mutable)
              castable
              constructible
              (equatable identity)
$record_type
:= scheme.new
              constructible
              (equatable identity)
```

Although ostensibly a record has fields, in order to be traversable a record is simply a small mutable array (structural comparison does in fact expose the order of fields).
To illustrate a possible extension, the `length` field is marked as `inclusive`, meaning the value of the `length` field is included in the set of value indices.
This in particular implies the array is non-empty and has a maximum of 256 elements(rather than 255).

For each record type that is declared in the source language, one allocates an instance of `$record_type`.
This instance is than used as the `type_tag` in order to be able to dynamically check that the record was allocated as the expected record type.
This way the behavior of external code using a module with this design cannot accidentally depend on two different record types having the same internal representation.
Note that the `$record_type` scheme is so simple that its packed-pointer representation could simply be an integer, with each ``allocation'' simply incrementing the current maximum integer rather than actually putting anything on the heap.

```
$adt
:= scheme.new (parent implicit $nonprimitive)
              (field type_tag (gcnref $adt_type)
                     readable immutable)
              (field case_tag (unsigned 8)
                     readable immutable)
              castable
              (extensible (cases $nullary $constructor))
              (equatable case)
$nullary
:= scheme.new (parent implicit $adt)
              castable
              constructible
              (equatable deep)
$constructor
:= scheme.new (parent implicit $adt)
              (field (length inclusive) arity (unsigned 8)
                     readable immutable)
              (field (indexed arity) member (gcref $uniform)
                     readable immutable)
              castable
              constructible
              (equatable identity)
$adt_type
:= scheme.new
              constructible
              nullable
              (equatable identity)
```

For algebraic data types, we separate the nullary constructors from the rest (just as OCaml does).
In this way we can use `deep` equality on nullary constructors in order to make them packable.

Just as with records, every `$adt` instance has a `type_tag` to prevent exposing when cases in different algebraic types happen to have the same internal representation.
The `type_tag` is `null` when the instance represents `unit` or a tuple.

The `case_tag` is used to distinguish between cases *of the same arity* (structural comparison exposes the order in which cases of the same arity are declared).
This design imposes a maximum of 256 cases of the same arity within any particular algebraic data type.

For `$nullary`, note that the `type_tag` field can often be packed (using the same incrementing trick as `$record_type`), as can the `case_tag` field, meaning `$nullary` instances will rarely need to be allocated on the heap.
For `$constructor`, note that again the `length` field is `inclusive`, as the empty case is already handled by `$nullary`, capping the arity of any constructor at 256 (inclusive).

With all this structure in place, polymorphic structural comparison can be implemented in WebAssembly by first switching on all the possible `cases` for the two instances and then structurally traversing and recursing as appropriate for the case at hand.
