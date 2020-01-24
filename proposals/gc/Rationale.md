# Rationale

Whereas the [Overview](Overview.md) provides motivations behind the *high-level* facets of the proposed design, this document provides motivations behind the *detailed* facets of the proposed design.
This document is structured to mirror the [MVP](MVP.md) so that when a reader has a question about a particular detail in the MVP, they can look up the corresponding motivation of that detail in the matching section here.
For a more coherent picture of how these details come together to achieve a particular effect, refer to the case studies linked to in the [Overview](Overview.md).
This document is written with the expectation that the reader has first read the [Overview](Overview.md) and is meant to be read alongside the [MVP](MVP.md).

# Schemes

The design has a distinct separation between schemes and types.
Schemes can refer to types (in field attributes), and types can refer to schemes (through `gcref` and the like).
This separation substantially simplifies and accelerates type-checking, particularly subtyping, by separating the process into stages.
First, one loads the schemes, whether imported or `scheme.new`ed.
From the parent declarations, one establishes the `implicit` and `explicit` hierarchy of schemes.
The `implicit` hierarchy is used for subtyping checks; however, subtyping checks do not need to examine the schemes themselves, and as such subtyping between `gcref`s can be done in constant time.
Parent-child and import-export scheme pairs are checked for compatibility, sometimes involving subtyping and equivalence checks between types of fields, but those checks are not recursive.
This makes scheme-compatibility-checking linear time.

## Parent Attribute

Schemes explicitly state their parent, rather than having this be inferred from compatibility of attributes.
This is necessary for the above staging of subtyping and scheme-compatibility-checking.
It is also similarly necessary for keeping run-time casts constant time.
Casts must be relatively frequent in any proposal with simple types, so it is especially important to make them reliably efficient.

### Implicit Parents

Having references to child schemes be able to be subtypes of references to parent schemes is critical for applications of certain schemes.
For example, objects in nominal OO languages have a pointer to a v-table. More specifically, given a class `Foo`, the v-table field for a `Foo` object should be a reference to `Foo`'s v-table scheme so that we can access all the virtual methods of `Foo` without further casting.
But suppose `Foo` extends `Bar`.
Then the scheme for `Foo` objects needs to be compatible with the scheme for `Bar` objects.
In particular, their v-table fields need to be compatible.
Since the v-table field is `readable` but not `writeable`, this is possible iff their respective types are subtypes.
So, to support this pattern, we need the reference type for `Foo`'s' v-table scheme to be a subtype of the reference type for `Bar`'s v-table scheme, which the `implicit` parent-child makes possible.

### Explicit Parents

Pointer-packing is an important optimization for many languages.
But pointers are small, making bits a valuable commodity.
As such, static type information can help improve the pointer representation for a specific scheme.
For example, languages with parametric polymorphism often need a "uniform" representation for their values that polymorphic functions can operate on.
At the same time, monomorphic code knows the exact types of values and so can represent them more efficiently.
The `explicit` parent-child relationship lets register-level representations be optimized based on static type information, while still ensuring that heap-allocated instances can be changed between the two static types without copying or the like.

One particular application of this feature is language interop. A module compiled in one language can import an "interop" scheme and declare it as the `explicit` parent of its "uniform" scheme.
Code that is supposed to operate on values specific to the module's language can use the "uniform" scheme, which can be optimized for representing values of the language.
When values need to be shared with the outside world, they can be converted to the "interop" scheme, which has a more generalized representation that can simultaneously represent values from many languages.
Thus the `explicit` parent-child relationship addresses the tension between wanting to optimize for one's own values and wanting to share values with others.

## Field Attribute

Parent-child schemes will often have much of the same content.
Consider that the Object class in Java has 10 different methods, so every v-table scheme for every Java class will have fields for these 10 methods, generally always with the exact same attributes.
Repeating this content wastes binary size and type-checking time.
For this reason, child schemes automatically inherit all the fields of their parent.

### Field Refinement

At the same time, a child scheme occasionally need to modify an existing field of a parent scheme.
A common example is v-tables, where the scheme for a subclass's objects needs to refine the type of the v-table field to be a referent to the scheme for the subclass's v-table.
For this reason, a field attribute can be declared to `refine` a parent field.
Refinement should not be able change whether a field is used as a length of the array or is itself indexed, which is why `refine` is an alternative to `length` and `indexed`.

### Length Fields

In order to implement bounds checks on array accesses to preserve memory safety, the semantics needs to know which field specifies the length of the array.
The garbage collector also often needs to know this field so that it can walk the array.
The garbage collector might even want to store the field in the object's header in order to support a more uniform object-walking implemention, which is why a non-`length` field cannot be refined into a `length` field.

Once allocated, the length of an array cannot change, which is why a `length` field must be immutable. Also, the semantics needs to be able to know how to interpret the value of this field as a length, which is why a `length` field must have an `unsigned` type. There are language features, such as OCaml's structural comparison, which rely on using small arrays to represent values. This is why `unsigned` is used rather than `i32` or `i64`. Similarly, knowing the maximum number of bits can enable engine implementations to pack this field into the object's header, as is done in OCaml.

### Indexed Fields

An indexed field specifies how it is indexed: by some dynamically determined length that can be found in a particular field, or by some statically determined length.
The latter is useful for internal data structures; for example, v-tables in some VMs use an `indexed 19` field for storing interface-method pointers.
The former is useful for both internal and surface-level data structures.
The reason a specific `length` field must be named is to be forwards-compatible with the possibility of multiple forms of dynamic indexing.
For example, some languages use negatively-growing arrays for internal data structures.

### Integer Types

In addition to the `unsigned` field type being useful for array lengths, integer types are useful for packing.
In particular, some language implementations pointer-pack "small" immutable integers.
One issue is that what constitutes as "small" dependes on whether the integer is `signed` or `unsigned`, which is why the integer type explicitly states which signing to use.
This also informs how to sign-extend values when reading from (or writing to) such fields, reducing instructions associated with this pattern.

### Field Views

The flags `readable` and `writeable` indicate the scheme's view of an instance.
Having the v-table field of an object be `readable` and not `writeable` is important for permitting the covariant refinement of v-tables described above.
Permitting `writeable` fields that are not `readable` could be useful for efficient secure interop; i.e. one module could give another module a write-only array to fill without worrying about needing to clear the current contents of that array.
There does not seem to be any application for fields that are neither `readable` nor `writeable`, but the possibility does not seem to place any burden on any aspect of the system; except for scheme-compatibility-checking, a type-checker and runtime can just pretend such field attributes were not declared.

### Field Mutability

Mutability specifies global guarantees about how a field's value can change.
Both `initializable` and `immutable` are useful for optimization.
In particular, they can be used to eliminate redundant accesses of a field.
Such eliminations might be done by a wasm-to-wasm optimizer or by an engine.
One such use case is class-method dispatch.
With `immutable`, a wasm generator could emit the instructions for fetching an object's v-table and then a function pointer within that v-table *every time* a class method is dynamically dispatched upon&mdash;relying upon a subsequent optimization pass to eliminate redundant `immutable` accesses, thereby separating concerns and simplifying the code base.
Nearly the same optimization would work for `initializable`, since execution can only continue after fetching the function pointer from the v-table if the v-table is non-null, i.e. non-default, and therefore the field is guaranteed to no longer be changed.
This would enable languages that need to dynamically generate internal structures like v-tables to still benefit from the many optimizations that `immutable` provides.
As for engine-level optimizations, `immutable` and `initializable` could be used to optimize inlined functions or to enable inline caching, although a hint that the scheme for v-tables is expected to have few instantiations might be useful for guiding engines when to do the latter.

## Instantiation Attribute

An instantiable scheme is either `constructible`, meaning one can construct (i.e. allocate) instances using that scheme, or `extensible`, meaning one can specify children of the scheme that may themselves be instantiable.
A scheme can never be both `constructible` and `extensible` simply because we found the separation keeps the specification significantly simpler and clearer.
For example, we do not need to distinguish between "exact" instances of a scheme and instances of child schemes.
This separation has no run-time impact, and its potential verbosity is addressed by the fact that the only complex attributes&mdash;field attributes&mdash;are automatically inherited from parents.

Arguably, every `scheme.new` scheme should have an instantiation attribute, otherwise there is no need for the scheme to exist.
We elide that requirement for now simply because no other attribute is required and to emphasize that when schemes are imported or exported one should not need to import or export their instantiability, enabling a module to export a scheme while retaining guarantees that no other module can create instances of that scheme.

### Extensibility

Most extensible attributes are "castable", meaning they permit instances of the given extensible scheme to be cast to certain child schemes.
But enabling such casts often requires certain meta-information to be associated with instances, requiring space for this meta-information to be reserved somewhere even if this meta-information is just a single bit signifying that the particular instance can never be cast.
As such, we permit a scheme to be simply `extensible` without specifying any casting structure.
For example, v-tables generally do not need their own casting structure because the run-time type of the v-table can be discerned from the run-time type of the object the v-table belongs to (since the v-table field is refined by each class's scheme).

As for castable extensibilities, there a variety of casting mechanisms each well-suited to different tasks.
The proposal provides a castable extensibility for each of the casting mechanisms we found most common, and which altogether seem to cover the needs of the in-depth case studies we analyzed.
Because there are multiple mechanisms, the proposal also ensures that every cast instruction can statically determine which casting mechanism it must use.

#### Explicit Extensibility

`explicit` extensibility permits arbitrary child<sup>+</sup> schemes so long as all child schemes are `explicit` children.
This feature is intended for multi-language schemes, akin to, say, `anyref`.
For example, one could define a universal scheme `$unischeme` via `scheme.new (extensible explicit) nullable unequatable`, and then any module that wants its value-level instances to be exchangable with other wasm modules via `$unischeme` could simply import `$unischeme` and designate it as the `explicit` parent of the "root" scheme for its value-level schemes.
Because all child schemes are `explicit`, the `$unischeme` could be represented at the register level through a fat pointer; one word would be an identifier of the specific child scheme the instance belongs to, and the other word would be the register-level representation of that child scheme.
This would impose no constraint on how instances of a given module are represented, while still enabling efficient communication between modules through `$unischeme`.

#### Flat Extensibility

`flat` extensibility is primarily designed for the situation where one expects there to simply be a number of `constructible` child schemes, but that number is large or unknown.
The implementation is expected to associate a (locally) unique identifier with each child scheme, and tag every instance with this identifier.
For some garbage collectors, this unique identifer might simply be the address of the descriptor used to walk the scheme (assuming equivalent descriptors are not merged together).
A cast is then implemented by comparing this tag with the target scheme&mdash;a very quick operation.

A major use case for this feature is closures.
The scheme for the function type of a typed functional language can be declared flatly extensible, with every closure scheme declared to be a constructible child.
The code implementing a closure will then first cast the provided closure environment from the generic scheme for functions to the scheme for the specific closure being implemented.
Yes, the high-level language can guarantee this cast is unnecessary, but the proof of that guarantee cannot be communicated to wasm (yet), and so this cast needs to be performed.
But this design ensures the cast is extremely low overhead&mdash;just a memory load (likely to be cached or to prepare the cache for subsequent reads from the closure environment) followed by an equality check with a compile-time constant.

A child of a flatly extensible scheme can itself be extensible *provided* it uses `cases` extensibility.
Casts can still be implemented efficiently in two ways.
One way is to have an identifier for each case and ensure that these identifiers are dense, meaning no other identifier will be arithmetically between these identifiers.
Then, rather than performing an equality check against an instance's tag, one checks whether the tag is between the lowest identifier and the highest identifier.
The other way is to have one shared identifier for all cases and to simply embed the specific case number somewhere in the instance.
Either way, this feature does not impose overhead on flat casts to simply `constructible` children.

The primary use case of this provision is algebraic data types (ADTs).
A typed functional language might want a unified scheme for all ADT instances.
But the language would also like to be able to cast from that scheme to a specific ADT (but not a specific case of the ADT!) without having to fix all the ADTs at compile time (say because more modules might be dynamically loaded).
With this provision, the unified scheme can have `flat` extensibility and the scheme for an ADT can have `cases` extensibility, with one `constructible` child for each case of the ADT.

#### Hierarchical Extensibility

`hierarchical` extensibility is intended for settings where an instance might be cast to many targets due to varying degrees of static type precision (in the surface language).
Conceptually speaking, this is implemented by tagging each instance with a pointer to a table of the various `castable` schemes it belongs to.
Because schemes can have at most one parent, each `castable` scheme is at a fixed depth in the casting hierarchy, which determines the unique offset in this table that its identifier might be found it.
Thus a cast can be implemented in constant time: load the table, load the size of the table, check that the depth is within that size, load the identifier at the depth-offset, then check that the identifiers match.

A major use case is classes with single class-inheritance in nominal OO languages.
The design has the added benefit that a successful cast simultaneously ensures the safety of subsequent memory options *and* ensures a cast in the surface-level language would succeed in common semantics, thereby avoiding the need to have a wasm-level cast and a surface-level cast.
Note that this benefit only applies to casts to class types&mdash;wasm-level casts would not be used for casts to interface types.

#### Cases Extensibility

`cases` extensible is for when a scheme has a predetermined finite number of child schemes (though those children might have an unbounded number of children).
This feature enables a number of optimizations.
In terms of representation, it hints that these cases should be encoded as bits in the pointer rather than a tag in the heap.
In terms of instructions, it suggests that branching casts to multiple target schemes will be common and enables jump tables rather than chained branches for this scenario.
The feature also combines well with other features, such as `flat` extensibility above and `case` equality below.

## Cast Attribute

This attribute simply states that a scheme can be cast *to* (whereas castable extensibilities permit casts *from* a scheme).
Like the optionality of the instantiation attribute, the value of the optionality of the cast attribute lies mostly in importing and exporting.
A module can export a castable scheme without exporting the ability to cast to that scheme.
This is important for security applications of types, since unmitigated casting can be used to circumvent the security guarantees normally granted by type abstraction.
For this same reason, a cast is only permitted in a module if the module can *statically* guarantee that the target is a child<sup>\*</sup> of the source and that the source has a castable extensibility and that the target is `castable`.
Besides importing and exporting, `castable` schemes can impose some slight costs.
In particular, in a `hierarchical` setting, every `castable` scheme requires a slot in the cast table, increasing the size of these tables.

## Nullability Attribute

To clarify first, types distinguish between non-null references, possibly-null references, and definitely-null reference.
`nullable` indicates that the possibly-null (and definitely-null) reference types are permitted for a scheme.
Knowing that a scheme is not `nullable` can enable better packed representations.
For example, the `list` and `option` ADTs respectively have a `nil` and `none` constants.
In order to provide *semantic* abstraction of types, these two constants need to be distinguishable when in their "uniform" representation, meaning they cannot both be represented by the `null` value.
But when in the representation specialized to, say, `option`, the `none` case can be represented by the `0` address, and the `some` case can simply be the value of the contained element (i.e. unboxed), *provided* neither the `option` scheme nor the uniform scheme are `nullable`.

## Equality Attribute

Many languages rely on reference equality for semantics and for performant data structures and algorithms.
But many optimizations and some garbage-collection techniques do not preserve reference equality.
As an example, pointer-packing small integers forgets the location the integer had been allocated at.
So the equality attribute indicates what notion of equality on instances of the scheme, if any, needs to be supported and preserved by the engine.

### Unequatable

`unequatable` is designed primarily with importing and exporting in mind.
In particular, there is a difference between having no notion of equality (`unequatable`) and having an unknown notion of equality (the absence of an equality attribute).
One cannot endow a notion of equality on a space that (possibly) already has a notion of equality.
So `unequatable` is used to explicitly indicate that this scheme has no notion of equality, enabling child schemes to each specify their own notions of equality.

### Identity Equality

`identity` equality is primarily available to support languages with reference equality.

### Deep Equality

`deep` equality is primarily intended to enable pointer packing *and* to enable language-level equality implementations to take advantage of the fact that pointer-packed values can sometimes be compared by simply comparing the pointers *without* language-level equality implementations needing to know whether or not the values are pointer-packed.

### Case Equality

`case` equality is intended for languages that mix reference equality and deep equality.
In particular, a language might use deep equality for "primitive" types so that they can be pointer-packed, and use identity equality for "complex" types for efficient shallow comparisons.

The reason `case` equality is restricted to schemes with `cases` extensibility is because otherwise engines would need a way to determine at run time whether an instances belongs to a scheme with `identity` equality or `deep` equality.

### Equality-Dependence

Schemes with `deep` equality can have fields whose types use `deep` equality.
This can enable deep forms of pointer packing, but if unrestricted it can also make equality checks fail to terminate.
As with the restriction of `case` to `cases`, dependence is designed to ensure that engines can statically guarantee termination of equality checks and statically compile&mdash;without functions or loops&mdash;the instructions for implementing an equality comparison.

# Types

The proposed types distinguish between definite references (`gcref`) and possibly-null references (`gcnref`). While memory operations do not incur a performance cast when the reference is possibly null, knowing that a memory operation is guaranteed to succeed is useful for reasoning about and optimizing programs.
Furthermore, as discussed above, there is a representation cost to the possibility of `null`, and `gcref` enables us to have schemes that are not `nullable`.

The type `gcnull $scheme` represents solely the value `null`.
The reason that it is parameterized by a scheme is that the value `null` might be represented differently by different schemes.
For example, if an engine chooses to represent a particular nullable scheme using fat pointers, then `null` is represented in that scheme using a double-word value (most likely all 0s).
If `gcnull` were unparameterized, as `nullref` is, then nullable schemes could never be represented using fat pointers solely because their common subtype `gcnull` would presumably be a single-word type.

# Subtyping

First, some meta-level discussion of the role of subtyping in low-level languages:
* Extra typing information in instructions can be redundant when value types must be available from the context during type-checking anyway. Encoding the types into the instruction wastes space and type-checking time. Take a possible instruction design like `struct.set <typeidx> <fieldidx>`. One might think that the `<typeidx>` annotation improves type-checking, but it in fact burdens the type-checker. First, the annotation takes space in the binary while only providing information that can already be deduced from the type of the operand on the stack. Second, the annotation forces the type-checker to check if the type of the operand on the stack is compatible with the annotation. [Research on low-level type systems](http://www.cs.cornell.edu/~ross/publications/italx/) has found that making instruction typing be [monotonic with respect to subtyping](#instructions) can substantially reduce annotation size and improve type-checking performance by eliminating these overheads.
* A well-designed subtyping system can enable type inference that can eliminate even the need for type annotations on control-flow labels. Even though the type inference should likely be done offline, so that engines only need to type-check, [research](http://www.cs.cornell.edu/~ross/publications/italx/) has found that omitting these types can vastly simplify emission and optimization of WebAssembly programs. In particular, if one ensures subtyping is well-founded and has joins, i.e. least upper bounds, then tools can rearrange the control-flow graph and rely on a separate inference tool to automatically recover the critical type annotations at merge points in the control flow.
* Both of the above imply that one should design subtyping for a low-level language extremely conservatively, only making two types be subtypes if there is a valuable application.
* One non-application of subtyping in low-level languages is implicit coercion from one type to another. Low-level programs are not generally written by people; they are primarily generated by tools. These tools can easily generate an instruction that explicitly coerces a value from one type to another. This explicit coercion even enables engines to use static type information to optimize the representations of values, in which case the coercion instruction would translate to assembly that converts between these representations. And in cases when the engine does no such optimization of representations, then this instruction can simply be erased by the engine. If at some point a particular explicit coercion is discovered to be a frequent cause of binary bloat with no performance benefit, one can always extend subtyping to make the explicit coercions unnecessary. But such a change to the standard *is only possible in one direction*, so it is best to start conservatively.
* The primary application of subtyping in low-level languages is variance. For example, multiple case studies for this proposal rely on the ability to refine the type of a read-only field in a child scheme. These refinements mean that the value of that field for a particular instance will be perceived as having different types by different parts of the program, so these types need to be (covariantly) related by subtyping to ensure that those perceptions treat the value consistently. Similarly, a function might be defined for a relatively imprecise input type and yet also be used as a function that is guaranteed to be given only more precise inputs---a scenario that is safe provided the relevant input types are (contravariantly) related by subtyping. For example, the same function might be referenced by both a v-table and an interface-method table even though the former can often guarantee a more precise type for the "this" pointer than the latter.

## Implicit vs. Explicit

Implicit parent-child relationships are reflected in subtyping so that applications of variance, such as field refinement and functions, can take advantage of the compatibility between instances of these schemes.
However, this prevents engines from utilizing more precise static type information to better represent values at the register level.
For this reason, explicit parent-child relationships are *not* reflected in subtyping, instead using explicit coercions to take advantage of the heap-level compatibility between instances of these schemes.

## Null Subtyping

Some languages have a `Null`-like type that is used as a (near-)bottom type in the subtyping hierarchy, which is important for supporting various patterns and libraries.
For this reason, `gcnull` is both covariant *and contravariant* with respect to implicit parent-child relationships.
Supposing a surface-level class type `Foo` translates to a wasm-level type `gcnref $foo`, where `$foo` is an implicit child<sup>\*</sup> of `$object`, then translating a surface-level type like `Null` to `gcnull $object` ensures that this translation preserves subyping.
For this same use case, `gcnull $scheme` needs to be a subtype of, not just convertible to, `gcnref $scheme`.

# Instructions

Many instructions are given multiple typings.
This is done to make these instructions monotonic with respect to subtyping, for reasons explained above in [Subtyping](#subtyping).
In particular, the typings are designed so that, given more precise input types (with respect to subtyping), the instruction has more precise output types (and is more likely to have its side conditions satisfied).

## Construction

We chose `construct` rather than `new` to emphasize that the process might not actually allocate a new structure on the heap.
In particular, the appropriate equality attribute might permit the structure to be packed in the pointer rather than allocated on the heap.

### Construct Indexed

The design permits schemes to define immutable arrays.
For example, interface tables are immutable, which enables optimizations such as inline caches.
As another example, OCaml represents its ADTs using small immutable arrays in order to support polymorphic structural comparison.
This means we need a way to actually construct such immutable arrays, since we cannot simply allocate the array and subsequently mutate its elements.
The `scheme.construct_indexed` is the most straightforward to provide this functionality, though more elaborate instructions might be called for in the future.

The reason why `$field` is required is to be future-compatible with supporting multiple forms of indexing in a scheme.

### Construct Default

Many languages initialize values by first zeroing out a newly allocated block of memory and then initializing the fields one by one.
`scheme.construct_default` is provided to support this pattern, with a slight tweak to support `immutable` fields (such as v-tables in Java objects).

### Construct Copy

Although `scheme.construct_copy` might provide better performance for some common patterns, we included it in the MVP as a means to construct instances of schemes *without* knowing all the fields of the scheme.
In particular, a scheme might designate an imported scheme as its parent.
This instruction would enable one to allocate instances of that scheme without knowing anything about the imported scheme.
One pattern we are considering how to support is where some module is responsible solely for providing a language's runtime system, with wasm programs compiled from that language dynamically linking with that runtime module.
This would help keep wasm programs small, make for faster loads due to the shared runtime in the cache, and facilitate the sharing of data between wasm programs compiled from the same language.
One issue, though, is that the maintainer of the runtime needs to be able to make improvements without the programs being recompiled to wasm to support these improvements, so the question is how can we abstract as many details of the runtime module as possible.
This instruction is one idea to that end.

On a more technical note, the reason that the instruction only copies `immutable` fields is to ensure that the order of reads and writes used to implement the copying is invisible even in a multithreaded setting.

## (Indexed) Access and Mutation

These instructions seem self-explanatory.

## Equality

The equality attribute is already carefully designed to keep the instructions for equality simple.
The only interesting question is how to handle `null`.
For supporting languages in which `null` is a proper value, which is by far the most common use case, `gcnref.eq` treats `null` as equal to itself.
For languages, or more likely language implementations, in which `null` really denotes the *absence* of a value, `gcnref.eq_distinct` treats `null` as unequal to itself.
As an example, this is the semantics of equality on `null` in SQL.

## Casting

For all cast-related instructions, the instruction is valid only if the target scheme is *statically* a `castable` child<sup>\*</sup> of the parent with a castable extensibility.
These requirements ensures that the appropriate run-time information is *dynamically* available, while also providing the engine with enough information to *statically* optimize for the appropriate casting strategy.
Furthermore, requiring these properties to be proven *statically* is important for security applications of types, since unmitigated casting can be used to circumvent the security guarantees that the type abstraction provided by importing and exporting should normally ensure.

### Branch or Cast

`br_or_cast` is specifically designed to fall through when the cast succeeds.
For one, this matches the behavior of `cast`.
For another, multiple `br_or_cast` instructions are more likely to share the same code for the failure case than for the successful cast, so this design permits all those failures to branch to the same label.

### Switch Cast

Although `gcref.switch_cast` could be implemented by successive `gcref.br_or_cast` instructions, consolidating these branches both saves binary space on a common pattern and facilitates optimizations like using a jump table when the various target schemes correspond to the `cases` of the source scheme.