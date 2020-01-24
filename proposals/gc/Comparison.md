# Proposal Comparison

This is a comparison of the new GC proposal and the preexisting proposal.
This document is written with the expectation that the reader has first read the Overview and the Design Rationale of the new proposal.

# Nominal vs. Structural

The most fundamental difference is that the new proposal uses a primarily *nominal* type system whereas the preexisting proposal uses a primarily *structural* type system.
In a nominal system, types can be distinct even if they have same operational structure&mdash;types are actually names with operational structure attributed to those names.
In a structural system, types are defined solely by their operational structure&mdash;thus types with the same operational structure are necessarily identical.

Nominal types are in every major typed programming language.
In object-oriented languages, they take the form of classes and interfaces.
In functional languages, they take the form of algebraic data types and type classes.
While that suggests that WebAssembly should likely support some form of nominality, that does not mean that WebAssembly should itself be nominal.
The design critera for low-level languages are very different than those for high-level langauges.
In fact, even though functional languages have nominal algebraic data types and type classes, these constructs are often quickly lowered to a structural representation in the compiler.
Furthermore, many major typed languages have structural types as well, such as tuples, functions, and unions and intersections.
(Meta-note: union types should not be confused with sum types. Union types are coproducts with respect to subtyping, whereas sum types are coproducts with respect to "pure" programs. For historical reasons regarding their relationship with set theory, sum types are often referred to as (disjoint) union types, which causes significant confusion.)

## Inter-Module Sharing

For modules to be able to share data, they need to first agree upon the types of the data, and in particular they need to have a common language for describing these types.
In a structural system, one simply describes the structure of the data using a standardized grammar of structural types.
In a nominal language, though, the modules first have to agree upon the names to use, and only then can they describe the structure of the data in terms of those names.

This might lead one to believe that structural types are the better approach for a low-level language for which so much modularity is needed and dynamic linking/sharing is expected.
However, agreeing on a language of types is a much easier problem than agreeing on a structure for data.
Consider the simplest example: pairs.
If you look at the case studies for [Nominal OO](NomOO.md) and [Typed Functional](TypedFun.md) languages, you will see that the two systems represent pairs very differently.
The reason is that language implementations rely on all values satisfying certain invariants.
For nominal OO languages, all values need a v-table for supporting dynamic dispatch, whereas for OCaml, all values need traversal data for supporting polymorphic structural comparison.
So, even if these modules agreed on a language of types, that does not really solve the problem of exchanging pairs of values between these modules.

Because of this, this value of structural types seems more applicable to [Interface Types](https://github.com/WebAssembly/interface-types/tree/master/proposals/interface-types) than to garbage collection.
The interface type could be a structural tuple, and then adapters could be responsible for converting between that simple structural tuple and the language-implementations own representation of tuples.
For more advanced inter-module sharing, one could use importing and exporting of (nominal) schemes, with the host-environment dynamic linker (e.g. written in JS) being responsible for hooking up exports to the corresponding imports.

## Type-Checking

In order to be at least reasonably expressive, the preexisting proposal uses *regularly coinductive* deterministic structural types.
This means that structural types can be defined through fixpoints provided those fixpoints are unparameterized.
Most well-known applications of fixpoints use sum types, e.g. `List(E) = fix T. 1 + E * T`, though the preexisting proposal does not currently directly support sum types (unlike the new proposal).
One lesser-known application to low-level languages is presented in the overview though.
Fixpoints can be used to ensure that the "this" pointer of a v-table method newly introduced by a class necessarily belongs to an instance of that class.
(Methods declared by superclasses still can only guarantee a less precise "this" pointer. Both these points apply to both proposals.)
Note that the preexisting proposal also uses fixpoints to define closures, but in a way that provides no real value, e.g. it does not eliminate any casts.
More importantly, fixpoints are all-but-necessary to keep importing and exporting of types tractable in a structural setting.

Both proposals rely on subtyping to support many of the same patterns.
With regularly coinductive *determinstic* structural types (as in the preexisting proposal), subtyping is known to be O(n<sup>2</sup>).
With *irregularly* coinductive (deterministic/non-deterministic) structural types, i.e. where fixpoints can be parameterized, subtyping is known to be undecidable.
The latter two results are important for knowing the limitations on how this approach can be extended.

Because of the first result, it seems the plan is to rely on two things.
First, structural types have a canonical form, so through cons-hashing one can check type equivalence in constant time.
(Even without that trick, type *equivalence* can be determine in near-linear time.)
Second, those cons-hashes can be used to cache subtyping results.
Of course, how well caching works always depends on the actual case at hand, and without any benchmarks to test on for a while, this approach just has to keep its fingers crossed.
Plus, type-checking might be used as a DOS attack vector against some applications of WebAssembly, in which case an attacker can specifically ensure caches miss.

The new proposal avoids these problems by staging scheme validation and subtyping.
Schemes explicitly state their (implicit) parent-child relationships; subtyping reflects this relationship *without digging deeper into the definitions of schemes*; scheme compatiblity is then checked in linear time.
This makes subtyping linear time (due to function types&mdash;reference subtyping is constant time), even though scheme definitions can be mutually recursive.
This should extend to importing and exporting by the same reasoning.

## Casting Performance and Security

Both proposals use "simple" types, which fundamentally implies that casting will be much more frequent than in a low-level language that does not need to be concerned about malicious programs.
The preexisting proposal uses structural casts to support certain patterns.
The same complexity issues described above apply here, except now the issues affect run-time performance rather than compile-time performance.
Even at present, engines rely on cons-hashing to efficiently support `call_indirect`.
But as WebAssembly expands to more advanced languages with more advanced libraries, casts (and indirect calls) will grow less and less between equivalent types and more and more between subtypes.
So relying on cons-hashing, caching, and speculation seems a risky venture for an irrevocable web standard facing such uncertainty.

Besides performance, structural casts are problematic for security.
If a module exports an abstract type, a malicious/aggressive module can simply bypass the abstraction by casting values of that type to their expected structural type.
Similarly, if WASI provides a capability to a specific set of functions, a module can cast that capability to an expected larger set of functions, meaning that WASI cannot rely on types to narrow capabilities and instead must regularly dynamically allocate capabilities.

The new proposal, on the other hand, guarantees that all casts are efficiently implementable in constant time, and only permits casts between types that the module has both been given access to (through explicit exports) and been informed are related (again through explicit exports).

## Semantic Abstraction

Structural casts cause problems for type abstraction, but WebAssembly should also be concerned about semantic abstraction.
A compiler to WebAssembly wants to have the property that semantically equivalent surface-level programs compile to semantically equivalent wasm programs.
For example, reordering the cases of an algebraic data type is a semantics-preserving transformation in many languages (besides OCaml), as is renaming a case (provided all match expressions and constructing expressions are similarly renamed), and so a language compiler wants to ensure that the resulting change in the compiled WebAssembly module is similarly semantics preserving.
If values can exit and enter the module, through say `anyref`, then this semantics includes the casting behavior of values.

One can prove that this has significant implications on casting between different algebraic data types in particular.
Suppose `Foo = A of char | B of char` and `Bar = X of char | Y of char`.
With structural types, both of these would be represented by `char + char` if you have sum types, or by `bool * char` without sum types.
So consider what happens if an `A('a')` gets passed to the outside world as an `anyref`, which later than passes it back to the module as an `anyref` but where the module expects a value of `Bar` and so casts it appropriately.
With the structural encoding, this cast will succeed and the value will now be perceived as `X('a')`.
But now suppose the cases of `Foo` are reordered&mdash;a semantics-preserving transformation&mdash;and the same exchange with the outside world occurs.
This reordering will change the structural encoding of `A('a')`, and thus in the end the value will now be perceived as `Y('a')`, thereby demonstrating that the behavior of the program has changed.
One might consider determining the encoding by using the names of the cases, but case-renaming is also supposed to be semantics preserving, so that still does nothing to actually solve the problem.
(And this is just the easy case; we haven't even considered the problem of the outside world manufacturing its own structural values and attempting to pass them off as `Bar`s).
Nominality is actually *necessary* to provide semantic abstraction.

The new proposal solves the abstraction problem by associating a unique *abstract* name with each case.
It also still permits the packed `bool * char` representation by letting `Foo` and `Bar` each be  `explicit` children of the uniform representation (or of `anyref`) and by using `cases` extensibility (from which the engine determines that a `bool` is sufficient to distinguish the possible cases).
The values are only tagged with the specific abstract name of the case when they are converted into the uniform representation, and even then this tagged representation can likely be pointer-packed.

# Schemes

In order to structure this document, here we go through the design of schemes and discuss the differences of their design and the impact of those differences.

## Pointer-Packing

The second-most significant difference is the approach to pointer-packing.
The preexisting proposal puts the burden of pointer-packing solely on the wasm generator via `i31ref`.
However, pointer-packing strategies are generally designed to particular machine architectures (especially bit-width) and garbage-collection strategies, neither of which the wasm generator has any control over.
Furthermore, in [this discussion](https://github.com/WebAssembly/gc/issues/53) multiple people provided multiple examples of pointer-packing strategies that `i31ref` could not support, as well as multiple garbage-collection strategies that `i31ref` is incompatible with.
And then there is the issue that some intended use cases (e.g. nullary cases of algebraic data types) of `i31ref` fail to ensure semantic abstraction.

The new proposal, on the other hand, splits the burden of pointer-packing across the wasm generator and the wasm engine.
The wasm generator must design their schemes to be packable, say by using solely `immutable` fields, using small integer types (with appropriate sign hints), and using `cases` to suggest useful bit flags.
The wasm engine must recognize packable schemes and optimize their packed-pointer representation appropriately according to the architecture and garbage-collection strategy at hand.
One can even express `i31ref` via a packable `$i31` scheme that the engine can pack as appropriate, though we found in our case studies that its use cases are better served through other features of the new proposal.

## Fields

### Field Refinement

The preexisting proposal has the ability to extend structures, corresponding to the new proposal's notion of automatic field inheritance.
However, the preexisting proposal does not support field refinement, forcing the schemes of classes (but not of class v-tables) to repeat the entire contents of their superclasses, and preventing classes from  extending superclasses defined in separate modules without knowing their entire contents.

### Length

The preexisting proposal fixes the length of arrays at 32 bits.
Our in-depth case study found that OCaml benefits from small arrays, where ideally the length can be packed by the engine into the gc-header.
And, of course, there are applications of large arrays.

### Indexed Fields

The preexisting proposal does not permit arrays to have non-indexed fields.
However, arrays in nominal OO languages are implemented with a header providing a vtable (and in some cases the run-time type information of the element type).
Thus these languages will need to implement arrays by using an object with a pointer to an array, adding a layer of indirection for array reads and writes.

The new proposal also allows multiple fields to be indexed, permitting arrays of unboxed structures.

### (Im)Mutability

This is difficult to compare because the preexisting proposal is inconsistent with its treatment of mutability.
Immutability seems to not be explored much in the preexisting proposal.
For example, for some reason `struct.new` is specified to only allocate structures where every field is mutable, and while immutable array types are expressible there is no way to allocate them.
The new proposal has explored these issues, providing preliminary designs for the instructions necessary to utilize immutability in the case studies.
It also decouples variance (`readable` and `writeable`) from mutability.
And it introduces the `initializable` notion of mutability, as it was found useful for the case studies and for enabling important optimizations like inline caches.

## Instantiation, Extension, and Casting

In the preexisting proposal, subtyping of structs always permits prefix subtyping.
This means that engines can never statically guarantee the size/layout of a referenced struct, forcing all structs to have a header dynamically informing the garbage collector how to walk it.
The new proposal enables engines to guarantee that size/layout of instances of certain schemes can always be determined statically, enabling engines to omit headers in these cases, which implementations of Lisp/Erlang/Prolog have found useful for eliminating a third of space overhead for all cons cells.

Similarly, the preexisting proposal permits all references to be cast, requiring all instances to allocate a casting header (though at least this can be combined with the required gc-header).

The new proposal supports four casting strategies.
One of these, `explicit` extensibility, makes no sense in the preexisting proposal.
Of the remaining three, the preexisting proposal only supports the slowest strategy through its `rtt` values: `hierarchical` extensibility (besides the even slower strategy, structural casting, which the new proposal does not support for reasons [above](#nominal-vs.-structural).
The `flat` and `cases` extensiblities cannot be supported (modulo whole-module analysis) because both rely on guarantees that the casting hierarchy is restricted to limited structures, and the preexisting proposal has no way of preventing instructions from creating a new sub-`rtt` for a given `rtt`.

Even if such restrictions were made possible to impose, by using first-class `rtt` values, engines have no way (modulo whole-module analysis) of ensuring that casts to a given `rtt` will know the appropriate casting strategy for that `rtt`.
Thus there needs to be some way to determine the appropriate strategy dynamically.
Or, more likely, to forgo optimized strategies altogether and support only one strategy, such as class descriptors.
Certainly this strategy is well suited to languages like JavaScript, but WebAssembly should be designed to outperform compilations to JavaScript.
The new proposal supports class descriptors, which is important for quickly integrating the proposal into existing browser implementations, but it also enables engines to provide the more specialized casting strategies that have been found to suit languages besides JavaScript well.
In fact, the `flat` extensibility can even be used to optimize casts using class descriptors, since it ensures that the cast can only succeed if the class descriptor matches exactly.

## Nullability

In the preexisting proposal, everything is nullable, meaning for every type the optional reference to that type is a valid type.
This limits the ability to pointer-pack references, as every pointer-packed representation needs to encode `null` even if the language can guarantee it is unnecessary.
Furthermore, `nullref` is a subtype of every optional-reference type, which is a supertype of every reference type, forcing all reference types to be (presumably) single word.
The new proposal makes nullability a controllable property of schemes, enabling better pointer-packing.
It also parameterizes `gcnull` by a scheme, permitting different `nullable` schemes to have different bit-widths.
Thus the new proposal permits engines to use fat pointers for certain schemes when appropriate.
Similarly, when a scheme is a `cases` of a few nullary child schemes, the new proposal permits references to that scheme to be encoded as a single byte, which in turn can be packed with other bytes when used as a field of other schemes, even if the `cases` scheme has the language's uniform scheme as its `explicit` parent.

## Equality

The new proposal supports multiple notions of equality (including `unequatable`).
The preexisting proposal supports only `identity` equality for structs and arrays.
This prevents all pointer-packing and all optimizations and implementation strategies that do not preserve reference equality.
This is done by imposing a common supertype, `eqref`, for all structs and arrays, again preventing optimized custom-bit-width representations of references.
By encoding equality as a property of schemes, rather than encoding it into subtyping, the new proposal supports a more notions of equality and more optimized register-level representations, including custom-bit-width representations, and furthermore enables engines to statically determine the notion of equality at hand and emit the appropriate optimized implementation of equality.

# Instructions

The new proposal ensures instruction typing is monotonic with respect to subtyping.
As such, its instructions require fewer (effectively redundant) type annotations, enabling smaller binaries and faster type-checking.

## Construction

The new proposal provides instructions for constructing immutable arrays (and structs) that are missing from the preexisting proposal: `scheme.construct_indexed`.
The new proposal provides instructions for constructing instances of a scheme without knowing all of its fields, supporting compilation of distinct surface-level modules to distinct wasm modules: `scheme.construct_default` and `scheme.construct_copy`.

## Initialization

The new proposal provides instructions supporting initializing (indexed) fields via compare(-with-default)-and-swap for multithreaded initialization of language runtimes: `gcref.initialize` and `gcref.initialize_indexed`.

## Casting

The new proposal provides instructions supporting jump tables for enums and algebraic data types: `gcref.switch_cast` and `gcnref.switch_cast`.
