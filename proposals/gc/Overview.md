# GC Extension

## Introduction

Note: Basic support for simple [reference types](https://github.com/WebAssembly/reference-types/blob/master/proposals/reference-types/Overview.md), for [typed function references proposal](https://github.com/WebAssembly/function-references/blob/master/proposals/function-references/Overview.md), and for [type imports](https://github.com/WebAssembly/proposal-type-imports/blob/master/proposals/type-imports/Overview.md) have been carved out into separate proposals which should become the future basis for this proposal.

See [MVP](MVP.md) for a more formal v1 proposal.


### Motivation

* Efficient support for high-level languages
  - faster execution
  - smaller modules
  - the vast majority of modern languages need it

* Provide access to industrial-strength GCs
  - at least on the web, VMs already have high performance GCs

* Non-goal: seamless interoperability between multiple languages


### Requirements

* Allocation of data structures that are garbage collected
* Allocation of byte arrays that are garbage collected
* Allow heap values from the embedder (e.g. JavaScript objects) that are garbage collected
* Down casts as an escape hatch for the low-level type system
* Modular (no need for shared type definitions etc.)


### Desirables

* Unboxing of small scalar values
* Explicit low-level control over runtime behaviour (no implicit allocation, no implicit runtime types)


### Challenges

* Fast but type-safe
* Lean but sufficiently universal
* Language-independent
* Trade-off triangle between simplicity, expressiveness and performance
* Interaction with threads


### Approach

* Independent from linear memory
* Basic but general structure: *data-representation schemes*, not high-level language types or object model
* Describe schemes at suitable level of abstraction to permit adaptation to engine designs and specialization to hardware platforms when possible
* Make explicit when schemes must be compatible so that unrelated schemes can be optimized separately rather than requiring a universal representation that works across all schemes
* Accept minimal amount of dynamic overhead (checked casts) as price for simplicity/universality
* Pay as you go; in particular, no effect on code not using GC, no runtime type information unless requested
* Don't introduce dependencies on GC for other features (e.g., using resources through tables)
* Extend the design iteratively, ship a minimal set of functionality fast

The proposal relies on the distinction between three concepts, which we refer to as references, addresses, and pointers.
References are the abstraction provided by WebAssembly.
Conceptually speaking, they denote an address in memory where the contents of the referenced instance can be found.
In reality, a reference denotes an engine-specific representation of the data, which for lack of a better term we will call a pointer.
For example, a pointer implementing a reference to an immutable `float64` on a 64-bit engine employing NaN boxing would often simply be the 64 bits of the `float64` rather than an address.
On the other hand, a pointer implementing a reference to an immutable `float64` on a 32-bit engine would typically be the address at which the 64 bits of the `float64` can be found in the heap.
Many languages design their pointers specialized packing schemes, but these schemes are often dependent on the hardware and the garbage collector at hand, which WebAssembly have no control over.
As such, this proposal is designed to give the engine the information it needs in order to develop a packing scheme well-suited for that language on the given hardware for the given garbage collector.


### Types

The sole purpose of the Wasm type system is to describe low-level data layout, in order to aid the engine compiling its access efficiently.
It is *not* designed or intended to catch errors in a producer or reflect richer semantic behaviours of a source language's type system.

This is true for the types in this proposal as well.
The introduction of managed data adds new forms of types that describe the layout of memory blocks on the heap so that the engine knows, for example, the type of an instance being accessed, avoiding any run-time check or dispatch.
Likewise, it knows the result type of this access, enabling subsequent uses of the result to also be check-free.
For that purpose, the type system does little more than describing the *shape* of such data.


### Potential Extensions

* Safe interaction with threads (sharing, atomic access)
* Direct support for strings?
* Defining, allocating, and indexing structures as extensions to imported types?


### Efficiency Considerations

GC support should maintain Wasm's efficiency properties as much as possible, namely:

* all operations are reliably cheap, ideally constant time
* structures are contiguous, dense chunks of memory
* field accesses are single-indirection loads and stores
* allocation is fast
* references should be boxable whenever possible in the high-level language
* allows ahead-of-time compilation and code caching


### Evaluation

Example languages from three categories should be successfully implemented:

* an [object-oriented language with nominal subtyping](NomOO.md) (e.g., a subset of Java, with classes, inheritance, interfaces)
* a typed functional language (e.g., a subset of ML, with closures, polymorphism, variant types)
* an untyped language (e.g., a subset of Scheme or Python or something else)


## Use Cases

### Support for GC Languages

* an [object-oriented language with nominal subtyping](NomOO.md) (e.g., a subset of Java, with classes, inheritance, interfaces)

### Inter-Module Sharing (Interface Types)

TBD

### JavaScript Interop

TBD

### Capabilities (WASI)

TBD


## Basic Functionality

* Extend the Wasm type section with schemes to express simple data structures and compatibility requirements
* Extend the value types with new constructors for references to instances of schemes


### Schemes

Every scheme specifies a number of attributes, such as the fields describing the data in the scheme, compatibility requirements with other schemes, and the ability to extend the scheme.
From this information, the engine generates an in-memory representation for the scheme and a packed-pointer representation for the scheme.
The values of type `gcref $scheme` are packed-pointer representations of instances of `$scheme`.
If the scheme is `nullable`, then the values of type `ngcref $scheme` are packed-pointer representations of instances of `$scheme` or of `null`.


### Compatibility

A scheme can specify a `parent` that it must be compatible with.
This parent can be an `explicit` parent, meaning only the in-memory representations must be compatible.
Or this parent can be an `implicit` parent, meaning also the packed-pointer representations must be compatible so that `gcref $child` is a subtype of `gcref $parent`.
There are a number of restrictions relating the attributes of a child with its parent to ensure that compatibility is possible to begin with.


### Fields

A scheme can specify a number of fields.

Each field has name, which is only used for compile-time coordination and has no other run-time significance.

Each field has a type that its corresponding value in any given instance must have.

Each field specifies whether it is `readable` and whether it is `writable`.

A field might specify a mutation attribute that specifies how the field can change over time.
Currently the options are `mutable`, `immutable`, and `initializable`.
The last indicates that the field can only be changed from its default value&mdash;once it has a non-default value, it cannot be changed.

A scheme can have one appropriately typed `immutable` field specified as its `length`, thereby making the scheme represent an array rather than just a tuple.
A field can be declared to be `indexed`, in which case it has a value for each non-negative index strictly less than the value of the `length` field.
The accessor and mutator instructions for `indexed` fields corresponding require an additional run-time argument indicating the index to access or mutate.


### Casting and Extension

Due to compatibility, an instance can belong to multiple schemes.
If the relevant schemes permit, the instance has run-time information that can be used to cast an instance from a less-precise scheme to a more-precise scheme.
Each scheme can describe how it can be cast and extended.
Every castable scheme has some corresponding run-time indicator, which might be part of the packed-pointer representation or part of the in-memory representation.
How casting is implemented is significantly affected by how schemes can be extended:
* An `explicit` extensible scheme can only have `explicit` children, which means that no child can influence the packed-pointer representation of the scheme.
* A `flat` extensible scheme limits casting so that casting can be implemented by just an arithmetic comparison to the run-time indicator of the target scheme.
* A `hierarchical` extensible scheme includes a casting table in the in-memory representation to enable arbitrary constant-time casts.
* A `cases $child*` extensible scheme explicitly lists its only permitted child schemes.


### Nullability & Defaultability

These notions are already introduced by [typed function references](https://github.com/WebAssembly/function-references/blob/master/proposals/function-references/Overview.md) and carry over to the new forms of reference types in this proposal.

Plain references cannot be null, avoiding any runtime overhead for null checks when accessing a struct or array.
Nullable references are available via a separate type constructor called `ngcref`, which is only permitted for schemes that are declared to be nullable.

Most value types, including all numeric types and nullable references are *defaultable*, which means that they have 0 or null as a default value.
Other reference types are not defaultable.


### Reference Equality

A scheme can define the notions of equality it supports.
`identity` equality is used to enable address comparison.
`deep` equality is used to enable packing instances of small schemes into the packed-pointer representation.
`case` equality is used to define equality case-wise according to its `cases`.
In the common case, equality can be checked by just shallowly comparing the packed-pointer representations without requiring any loads from memory.


### Sharing

TODO (post-MVP): Distinguish types safe to share between threads in the type system. See OOPSLA paper.


## Other Reference Types

### Universal Type

This type is already introduced by the [reference types proposal](https://github.com/WebAssembly/reference-types/blob/master/proposals/reference-types/Overview.md).

The type `anyref` can hold references of any reference type.
It can be formed via [up casts](#casting),
and the original type can be recovered via [down casts](#casting).


### Host Types

These are enabled by the [type imports proposal](https://github.com/WebAssembly/proposal-type-imports/blob/master/proposals/type-imports/Overview.md).

The embedder may define its own set of types (such as DOM objects) or allow the user to create their own types using the embedder API (including a subtype relation between them).
Such *host types* can be [imported](import-and-export) into a module, where they are treated as opaque data types.

There are no operations to manipulate such types, but a WebAssembly program can receive references to them as parameters or results of imported/exported Wasm functions. Such "foreign" references may point to objects on the _embedder_'s heap. Yet, they can safely be stored in or round-trip through Wasm code.

(type $Foreign (import "env" "Foreign"))
(type $s (struct (field $a i32) (field $x (ref $Foreign)))

(func (export "f") (param $x (ref $Foreign))
  ...
)


### Function References

Function references are already introduced by the [typed function references proposal](https://github.com/WebAssembly/function-references/blob/master/proposals/function-references/Overview.md).

References can also be formed to function types, thereby introducing the notion of _typed function pointer_.

Function references can be called through `call_ref` instruction:
```
(type $t (func (param i32))

(func $f (param $x (ref $t))
  (call_ref (i32.const 5) (local.get $x))
)
```
Unlike `call_indirect`, this instruction is statically typed and does not involve any runtime check.

Values of function reference type are formed with the `ref.func` operator:
```
(func $g (param $x (ref $t))
  (call $f (ref.func $h))
)

(func $h (param i32) ...)
```


## Type Structure

### Type Grammar

The overall type syntax is extended with three types: `gcref <schemeidx>`, `ngcref <schemeidx>`, and `nullref` (from the Reference Types proposal).
These types reference schemes, which are defined through a series of attributes using a syntax defined in the [MVP](MVP.md).
Note that there is no need for type recursion in this proposal.
And while schemes can reference each other recursively, this recursion is nominal, which greatly simplifies the definition of and algorithms for the type system.


### Subtyping

Subtyping is designed to be _non-coercive_, i.e., never requires any underlying value conversion.

The subtyping relation is the reflexive transitive closure of a few basic rules:
1. `gcref $scheme1` is a subtype of `gcref $scheme2` if `$scheme2` is an `implicit` parent of `$scheme1`.
  - Note: there is no subtyping relation between a child and its `explicit` parent.
2. `ngcref $scheme1` is a subtype of `ngcref $scheme2` if `$scheme2` is an `implicit` parent of `$scheme1` (presuming both types are `ok`.
  - Note: there is no subtyping relation between a child and its `explicit` parent.
3. `gcref $scheme` is a subtype of `ngcref $scheme` (presuming the latter is `ok`)
4. `nullref` is a subtype of `ngcref $scheme` (presuming the latter is `ok`)


### Import and Export

To do in detail. The high-level idea is one can import/export a scheme with required/exposed attributes. Such attributes include `parent` attributes, which would enable one to import/export multiple schemes and demand/ensure that they are related.


## Possible Extension: Nesting

* Want to represent structures embedding arrays contiguously.
* Want to represent arrays of structures contiguously (and maintaining locality).
* Access to nested data structures needs to be decomposable.
* Too much implementation complexity should be avoided.

Examples are e.g. the value types in C#, where structures can be unboxed members of arrays, or a language like Go.

Example (C-ish syntax with GC):
```
struct A {
  char x;
  int y[30];
  float z;
}

// Iterating over an (inner) array
A aa[20];
for (int i = 0..19) {
  A* a = aa[i];
  print(a->x);
  for (int j = 0..29) {
    print(a->y[j]);
  }
}
```

Needs:

* incremental access to substructures,
* interior references.

Two main challenges arise:

* Interior pointers, in full generality, introduce significant complications to GC. This can be avoided by distinguishing interior references from regular ones. That way, engines can choose to represent interior pointers as _fat pointers_ without complicating the GC, and their use is mostly pay-as-you-go.

* Aggregate objects, especially arrays, can nest arbitrarily. At each nesting level, they may introduce arbitrary mixes of pointer and non-pointer representations that the GC must know about. An efficient solution requires that the GC interprets (an abstraction of) the type structure. More advanced optimisations involve dynamic code generation.


## Possible Extension: Type Parameters and Fields

TODO
```
func_type   ::=  (func <type_param>* <param_type>* <result_type>*)
type_param  ::=  (typeparam $x <typeuse>?)

data_type   ::=  (struct <type_field>* <field_type>*) | ...
type_field  ::=  (typefield $x <typeuse>?)

def_type    ::=  ... | (typefunc $x <typeuse>? <def_type>)
```


## Possible Extension: Weak References and Finalisation

Binding to external libraries sometimes requires the use of *weak references* or *finalizers*.
They also exist in the libraries of various languages in myriads of forms.
Consequently, it would be desirable for Wasm to support them.

The main challenge is the large variety of different semantics that existing languages provide.
Clearly, Wasm cannot build in all of them, so we are looking for a mechanism that can emulate most of them with acceptable performance loss. Unfortunately, it is not clear at this point what a sufficiently simple and efficient account could be.
