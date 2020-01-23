# GC v1 Extensions

*Note: This design is still in flux!*

See [overview](Overview.md) for background.


## Language


### Schemes

Schemes are a new top-level component.
Each scheme describes a memory layout that the garbage collector is responsible for collecting and the type-checker is responsible for reasoning about.

* `schemetype ::= scheme.new <parentattr>? <fieldattr>* <instantiationattr>? <castattr>? <nullabilityattr>? <equalityattr>?`

We use `new` here to emphasize that, even though schemes are not actually created dynamically within an instance, two new schemes are never the same even if they have the same attributes.
In particular, two module instances can only refer to the same scheme instance if one exports the scheme to the other (though we forgo details of exporting/importing schemes for now).


#### Attributes

Attributes describe memory invariants that a garbage collector needs to know about a scheme in order to determine how to best represent and implement operations on the scheme.

* A parent attribute indicates what existing scheme that instances of this scheme must be compatible with. A scheme can have at most one parent attribute. The specified parent must be extensible.
  - `parentattr ::= parent (implicit | explicit) <schemeidx>`
    + An `implicit` parent means that references of this scheme are subtypes of references of its parent.
    + An `explicit` parent means that references of this scheme must be explicitly converted into references of its parent. This is important for enabling schemes to be packed differently based on static type information, which has been found to improve performance in a number of languages. However, the design ensures that conversion and casting will never require duplicating contents on the heap.
  - We use parent<sup>+</sup> to refer to parents and parents of parents and so on. We use parent<sup>\*</sup> to refer to parents<sup>+</sup> and this scheme. Likewise for child<sup>+</sup> and child<sup>*</sup>.
  
* A field attribute describes how an instance of the scheme can be accessed or mutated.
  - `fieldattr ::= field (length | indexed (<fieldname> | <length>) | refine)? <fieldname> (<type> | <inttype>) readable? writeable? <mutationattr>?`
    + Having a `length` field indicates that the scheme describes an array. For now, there can be at most one `length` field. A `length` field must have an `unsigned` type and be `immutable`.
    + An `indexed` field is one that must be accessed using a non-negative index that is strictly less than the (unsigned) value of the designated field or integer. The designated field must be a `length` field.
    + A scheme with a `refine` field must have a parent<sup>+</sup> scheme with a field of the same `<fieldname>`.
    + The `<fieldname>` is used in place of an index so that child schemes can specify fields without knowing the full contents of their parents. The specific name has no run-time significance; it is effectively an abstract name for an unknown offset to be determined by the engine.
  - `inttype ::= (signed | unsigned) <num-bits>` (where 0 < `<num-bits>` <= 64)
    + The `signed | unsigned` qualifier informs the engine how the integer should be packed, when appropriate.
  - `mutationattr ::= mutable | initializable | immutable` 
    + `initializable` fields must have defaultable type and can only be changed from their default value.
    + A `writeable` field must be either `mutable` or `initializable`.
  - A `refine` field must satisfy the following properties with respect to every parent<sup>+</sup>'s field with the same name:
    + If the parent<sup>+</sup>'s field is `readable`/`writeable`, then this field must be respectively `readable`/`writeable`.
    + If this field is `readable`, then the parent<sup>+</sup>'s field must be of a supertype of this field.
    + If this field is `writeable`, then the parent<sup>-</sup>`s field must be of a subtype of this field.
    + If the parent<sup>+</sup>'s field has a mutation type, then this field must have same mutation type.
  - For now, if a parent<sup>+</sup> scheme has a `length` field, then every field of this scheme must be a `refine` field.

* An instantiation attribute indicates how instances of this scheme can be created. A scheme can have at most one construction attribute.
  - `instantiationattr ::= constructible | extensible (explicit | flat | hierarchical | cases <schemeidx>*)?`
    + `explicit` is a castable extensibility that requires all child schemes to be `explicit` children, but imposes no restriction on how those schemes might be instantiated. This enables the representation of this scheme to be determined independently of the representation of its child schemes, but at the cost of explicit conversion.
    + `flat` is a castable extensibility that requires all `extensible` child<sup>+</sup> schemes to have `cases` extensibility, but ensures casts to child<sup>+</sup> schemes can be performed by an arithmetic range (or equality) check on an instance's run-time scheme identifier.
      - It might be worthwhile to have a flag that indicates to provide very fast casts by fattening the pointer. Probably would have to impose significant restrictions on hierarchy.
    + `hierarchical` is a castable extensibility that is permissive, but imposes some memory overhead to support constant-time downcasts to child<sup>+</sup> schemes via array-lookup.
    + `cases` is a castable extensibility that requires all `castable` child<sup>+</sup> schemes to be a child<sup>\*</sup> of one the schemes in this list.
      - It might be worthwhile to have a flag that indicates to pack the cases into the pointer itself.
    + Question: should we make it possible to restrict the kind of fields children can have?

* A cast attribute indicates whether this scheme can be cast to from its parent. A scheme can have at most one cast attribute.
  - `castattr ::= castable`
    + The scheme must have a parent<sup>+</sup> with a castable extensibility.

* A nullability attribute indicates whether `null` is considered to be a value of this scheme. A scheme can have at most one nullability attribute.
  - `nullabilityattr ::= nullable`
  - If this scheme has a parent, that parent must be `nullable`.

* An equality attribute describes how instances of this scheme (and its children) can and should be compared. A scheme can have at most one equality attribute. This is important for packing instances, since packing necessarily strips identity information.
  - `equalityattr ::= unequatable | equatable (identity | deep | case)`
    + A `constructible` and `equatable` scheme must specify `identity | deep | case`.
    + A scheme with `deep` equality must be `constructible` and have only `readable` and `immutable` fields (including parent<sup>+</sup> fields) whose types all have a notion of equality.
    + A scheme with `case` equality must have `cases` extensibility.
  - A parent of an `unequatable` scheme must be `unequatable`.
  - A child of an `equatable` scheme must be `equatable`.
    + If the parent has `identity` equality, so must the child.
  - Equality-dependence must be well-founded. A scheme with `identity` equality has no equality-dependences. A scheme with `deep` equality has equality-dependence on the schemes occurring in its fields' types. A scheme with `case` equality has equality-dependence on its `cases` schemes.


### Types

#### Value Types

* `gcref $scheme` is a new type. A value of `gcref $scheme` is an instance of a child<sup>\*</sup> of `$scheme`. This value may or may not be stored on the heap depending on the packing scheme the engine chose to use for `gcref $scheme`, which may differ for `explicit` parents and children.
  - `type ::= ... | gcref <schemeidx>`

* `gcnref $scheme` is a new type that is valid iff `$scheme` is `nullable`. A value of `gcnref $scheme` is either an instance of `gcref $scheme` or `null`.
  - `type ::= ... | gcnref <schemeidx>`
  - `gcnref $scheme ok` iff `$scheme` is `nullable`.

* `gcnull $scheme` is a new type that is valid iff `$scheme` is `nullable`. The unique value of `gcnull $scheme` is `null`.


#### Imports

* To do in detail. The high-level idea is one can import a scheme with required attributes. Such attributes include `parent` attributes, which would enable one to import multiple schemes and demand that they are related.


#### Exports

* To do in detail. The high-level idea is one can export a scheme with exposed attributes. Such attributes include `parent` attributes, which would enable one to export multiple schemes and reveal that they are related.


#### Subtyping

In addition to reflexivity and transitivity:

* `gcref $scheme1` is a subtype of `gcref $scheme2` if `$scheme2` is an `implicit` parent of `$scheme1`.
  - Note: there is no subtyping relation between a child and its `explicit` parent.

* `gcnref $scheme1` is a subtype of `gcnref $scheme2` if `$scheme2` is an `implicit` parent of `$scheme1` (presuming both `$scheme1` and `$scheme2` are `nullable`).
  - Note: there is no subtyping relation between a child and its `explicit` parent.

* `gcref $scheme` is a subtype of `gcnref $scheme` (presuming `$scheme` is `nullable`).

* `gcnull $scheme` is a subtype of `gcnref $scheme` (presuming `$scheme` is `nullable`).

* `gcnull $scheme1` is a subtype of `gcnull $scheme2` if `$scheme2` is an `implicit` parent or child of `$scheme1` (presuming both `$scheme1` and `$scheme2` are `nullable`).


### Runtime


#### Values

* In theory, values are all references to structures allocated on the heap with the appropriate data contents and run-time type information (RTTI). The order of fields in memory is suggested to be their order of declaration but might be rearranged for compaction purposes.

* In practice, values are expected to be packed when possible. What gets packed will change as engines develop more advanced packing and garbage-collection techniques. At first, the expectation is that small `immutable` schemes with `deep` (or no) equality will be packed. Soon after, schemes with `cases` extension will have those cases encoded as the low bits of a (aligned) pointer. Eventually, some schemes will no longer need RTTI in the heap, enabling optimizations such as the headerless cons cells often seen in Lisp implementations (where the low bits of the pointer inform the garbage collector that the heap structure is specifically a cons cell).

* A value is considered to be an instance of `$scheme` if it was constructed as some child<sup>\*</sup> scheme of `$scheme`.


### Instructions

All of the typing rules are given under the assumption that WebAssembly truly has subtyping, which significantly reduces annotation burden and eliminates many redundant type-checks.


#### Construction

We choose `construct` rather than `new` to emphasize that the process might not actually allocate a new structure on the heap.
In particular, the appropriate equality attribute might permit the structure to be packed in the pointer rather than allocated on the heap.

In the following, `i32` corresponds to `signed` or `unsigned` fields of `32` bits or less, and `i64` corresponds to `signed` or `unsigned` fields of `64` bits or less.
Question: what should happen when an `i32` or `i64` is inexpressible by the type of the field it is being assigned to?

* `scheme.construct <schemeidx>` constructs an instance of `$scheme` and initializes its non-field-indexed fields with given values
  - `scheme.construct $scheme : [t*] -> [(gcref $scheme)]`
    - iff `$scheme` is `constructible`
    - and `t*` corresponds to the non-field-indexed fields of `$scheme`
    - and all field-indexed fields have defaultable types

* `scheme.construct_indexed <schemeidx> <fieldname> <length>` constructs an instance of `$scheme` and initializes its fields with given values
  - `scheme.construct_indexed $scheme $field n : [t*] -> [(gcref $scheme)]`
    - iff `$scheme` is `constructible`
    - and `$scheme` has a `length` field with name `$field` whose type can express `n`
    - and `t*` corresponds to the non-`$field` non-`$field`-indexed fields of `$scheme` followed by the `$field`-indexed fields of `$scheme`

* `scheme.construct_default <schemeidx> <fieldname>*` constructs an instance of `$scheme` and initializes all fields *not* in `$field*` with default values
  - `scheme.construct_default $scheme $field* : [t*] -> [(gcref $scheme)]`
    - iff `$scheme` is `constructible`
    - and `t*` corresponds to the fields in `$field*`
    - and all fields not in `$field*` have defaultable types
    - and, if a `length` field is in `$field*`, then no fields by that field are in `$field*`

* `scheme.construct_copy <schemeidx> <fieldname>*` constructs an instance of `$scheme` using an instance of a `$source` scheme to initialize the `immutable` fields *not* in `$field*`
  - `scheme.construct_copy $scheme $field* : [(gcref $source) t*] -> [(gcref $scheme)]`
    - iff `$scheme` is `constructible`
    - and `t*` corresponds to the fields in `$field*` plus the non-immutable non-field-indexed fields of `$scheme`
    - and `$source` is a parent<sup>\*</sup> of `$scheme`
    - and every field in `$field*` is `immutable` and non-field-indexed in `$source`
    - and every `immutable` field of `$scheme` not in `$field*` has a corresponding `readable` field in `$source` with an equivalent type
    - and every field-indexed field of `$scheme` either has a defaultable type or is immutable and is in `$source` but not in `$field*` and the corresponding `length` field is in `$source` but not in `$field*`

* `scheme.null <schemeidx>` produces `$scheme`'s representation of the `null` value.
  - `scheme.null $source : [] -> [(gcnull $scheme)]`
    - iff `$scheme` is `nullable`


#### Accessors and Mutators

* `gcref.get <fieldname>` reads from field `x` of an instance
  - `gcref.get x : [(gcref $scheme)] -> [t]` and `: [(gcnref $scheme)] -> [t]`
    - iff `$scheme` has non-indexed `readable` field with name `x` and type `t`
  - traps on `null`

* `gcref.get_<numbits> <fieldname>` reads from field `x` of an instance
  - `gcref.get_nb x : [(gcref $scheme)] -> [inb]` and `: [(gcnref $scheme)] -> [inb]`
    - iff `$scheme` has non-indexed `readable` field with name `x` and type `signed|unsigned nbx` with `nbx <= nb` and `nb` equal to `32` or `64`
  - sign extends as indicated by `signed` or `unsigned` quality of field `x`
  - traps on `null`

* `gcref.set <fieldname>` writes to field `x` of an instance
  - `gcref.set x : [(gcref $scheme) t] -> []` and `: [(gcnref $scheme) t] -> []`
    - iff `$scheme` has non-indexed `writeable`and `mutable`  field with name `x` and type compatible with `t`
  - traps if field `x` has `initializable` mutability and has already been initialized
  - traps on `null`

* `gcref.initialize <fieldname>` attempts to initialize field `x` of an instance and indicates whether the initialization succeeded (where failure indicates `x` is already initialized)
  - `gcref.initialize x : [(gcref $scheme) t] -> [i32]` and `: [(gcnref $scheme) t] -> [i32]`
    - iff `$scheme` has non-indexed `writeable` and `initializble` field with name `x` and type compatible with `t`
  - traps on `null`


#### Indexed Accessors and Mutators

* `gcref.get_indexed <fieldname>` reads from field `x` of an instance
  - `gcref.get x : [(gcref $scheme) ti] -> [t]` and `: [(gcnref $scheme) ti] -> [t]`
    - iff `$scheme` has an indexed `readable` field with name `x` and type `t`
    - and `ti` is compatible with the length field or value of `x`
  - traps on `null` or if the dynamic index is out of bounds

* `gcref.get_indexed_<numbits> <fieldname>` reads from field `x` of an instance
  - `gcref.get_indexed_nb x : [(gcref $scheme) ti] -> [inb]` and `: [(gcnref $scheme) ti] -> [inb]`
    - iff `$scheme` has an indexed `readable` field with name `x` and type `signed|unsigned nbx` with `nbx <= nb`
    - and `ti` is compatible with length field or value of `x`
  - sign extends as indicated by `signed` or `unsigned` quality of field `x`
  - traps on `null` or if the dynamic index is out of bounds

* `gcref.set_indexed <fieldname>` writes to field `x` of an instance
  - `gcref.set_indexed x : [(gcref $scheme) t ti] -> []` and `: [(gcnref $scheme) t ti] -> []`
    - iff `$scheme` has an indexed `writeable` and `mutable` field with name `x` and type compatible with `t`
    - and `ti` is compatible with length field or value of `x`
  - traps if field `x` has `initializable` mutability and has already been initialized
  - traps on `null` or if the dynamic index is out of bounds

* `gcref.initialize_indexed <fieldname>` attempts to initialize field `x` of an instance and indicates whether the initialization succeeded (where failure indicates `x` is already initialized)
  - `gcref.initialize_indexed x : [(gcref $scheme) t ti] -> [i32]` and `: [(gcnref $scheme) t ti] -> [i32]`
    - iff `$scheme` has indexed `writeable` and `initializable` field with name `x` and type compatible with `t`
    - and `ti` is compatible with the length field or value of `x`
  - returns 1 if the field `x` now forever has the value of the second operand, 0 otherwise
  - traps on `null` or if the dynamic index is out of bounds


#### Equality

* `gcref.eq` compares two references whose scheme supports equality
  - `gcref.eq : [(gcref $scheme) (gcref $scheme)] -> [i32]` where `$scheme` is `equatable`

* `gcnref.eq` compares two possibly `null` references whose scheme supports equality, treating `null` as equal to itself
  - `gcnref.eq : [(gcnref $scheme) (gcnref $scheme)] -> [i32]` where `$scheme` is `equatable`

* `gcnref.eq_distinct` compares two possibly `null` references whose scheme supports equality, treating `null` as unequal to itself
  - `gcnref.eq_distinct : [(gcnref $scheme) (gcnref $scheme)] -> [i32]` where `$scheme` is `equatable`


#### Casts

* `gcref.convert <schemeidx>` converts an instance to an instance of `$target`
  - `gcref.convert $target : [gcref $source] -> [gcref $target]` and `: [gcnref $source] -> [gcnref $target]`
    - iff `$target` is a parent<sup>\*</sup> of `$source`

* `gcref.convert <schemeidx>` converts an instance to an instance of `$target`
  - `gcref.convert $target : [gcref $source] -> [gcref $target]`
    - iff `$target` is a parent<sup>\*</sup> of `$source`

* `gcref.test <schemeidx> (0|1)?` tests whether an instance is an instance of `$target`
  - `gcref.test $target : [gcref $source] -> [i32]`
    - iff `$target` is a `castable` child<sup>\*</sup> of `$source`
    - and `$source` has a castable extensibility attribute
  - `gcref.test $target n : [gcnref $source] -> [i32]`
    - iff `$target` is a `castable` child<sup>\*</sup> of `$source`
    - and `$source` has a castable extensibility attribute
  - returns 1 if the operand is an instance of `$target`, `n` if the operand is `null`, 0 otherwise

* `gcref.cast <schemeidx>` casts and converts an instance that was constructed as a child<sup>\*</sup> scheme of `$target`
  - `gcref.cast $target : [gcref $source] -> [gcref $target]` and `: [gcnref $source] -> [gcref $target]`
    - iff `$target` is a `castable` child<sup>\*</sup> of `$source`
    - and `$source` has a castable extensibility attribute
  - traps unless the operand is an instance of `$target`

* `gcnref.cast_null <schemeidx>` casts and converts `null` or an instance that was constructed as a child<sup>\*</sup> scheme of `$target`
  - `gcnref.cast_null $target : [gcnref $source] -> [gcnref $target]`
    - iff `$target` is a `castable` child<sup>\*</sup> of `$source`
    - and `$source` has a castable extensibility attribute
  - traps unless the operand is `null` or an instance of `$target`

* `gcref.br_or_cast <schemeidx> <labelidx> <labelidx>?` branches if an instance is an instance of `$target` (or if a value is `null`)?
  - `gcref.br_or_cast $target $l : [gcref $source] -> [gcref $target]`
    - iff `$target` is a `castable` child<sup>\*</sup> of `$source`
    - and `$source` has a castable extensibility attribute
    - and `$l : [gcref $source]`
  - `gcref.br_or_cast $target $l : [gcnref $source] -> [gcnref $target]`
    - iff `$target` is a `castable` child<sup>\*</sup> of `$source`
    - and `$source` has a castable extensibility attribute
    - and `$target` is `nullable`
    - and `$l : [gcref $source]`
  - `gcref.br_or_cast $target $l $lnull : [gcnref $source] -> [gcref $target]`
    - iff `$target` is a `castable` child<sup>\*</sup> of `$source`
    - and `$source` has a castable extensibility attribute
    - and `$l : [gcref $source]`
    - and `$lnull : [gcnull $target]` (designed so that `$l` and `$lnull` can be same label of type `[gcnref $target]`)
  - branches to `$l` iff the operand is an instance of `$target` or `$lnull` is omitted and the operand is `null`
  - branches to `$lnull` iff `$lnull` is specified and the operand is `null`
  - passes cast operand along with branch

* `gcref.switch_cast (<schemeidx> <labelidx>)? (<schemeidx> <labelidx>)*` branches to the label corresponding to the most precise scheme of an instance (or to `$lnull` if a value is `null`)?
  - `gcref.switch_cast $target1 $l1 ... : [gcref $source] -> [gcref $source]`
    - iff each `$targeti` is a `castable` child<sup>\*</sup> of `$source`
    - and `$source` has a castable extensibility attribute
    - and each `$li : [gcref $targeti]`
    - and if any `$targeti` are `$targetj1` are potentially the same scheme then `i` equals `j`
  - `gcref.switch_cast $target1 $l1 ... : [gcref $source] -> unreachable`
    - iff each `$targeti` is a `castable` child<sup>\*</sup> of `$source`
    - and `$source` has a castable extensibility attribute
    - and each `$li : [gcref $targeti]`
    - and if any `$targeti` are `$targetj1` are potentially the same scheme then `i` equals `j`
    - and every `castable` child<sup>\*</sup> of `$source` is guaranteed to be a child<sup>\*</sup> of some `$targeti` due to `cases` extensibilities
  - `gcref.switch_cast $tnull $lnull $target1 $l1 ... : [gcnref $source] -> [gcref $source]`
    - iff each `$targeti` is a `castable` child<sup>\*</sup> of `$source`
    - and `$source` has a castable extensibility attribute
    - and each `$li : [gcref $targeti]`
    - and if any `$targeti` are `$targetj1` are potentially the same scheme then `i` equals `j`
    - and `$lnull : [gcnull $tnull]`
  - `gcref.switch_cast $tnull $lnull $target1 $l1 ... : [gcnref $source] -> unreachable`
    - iff each `$targeti` is a `castable` child<sup>\*</sup> of `$source`
    - and `$source` has a castable extensibility attribute
    - and each `$li : [gcref $targeti]`
    - and if any `$targeti` are `$targetj1` are potentially the same scheme then `i` equals `j`
    - and every `castable` child<sup>\*</sup> of `$source` is guaranteed to be a child<sup>\*</sup> of some `$targetn` due to `cases` extensibilities
    - and `$lnull : [gcnull $tnull]`
  - branches to `$li` iff the operand is an instance of `$targeti` and furthermore every `targetj` that the operand is an instance of is a parent<sup>\*</sup> of `$targeti`
  - branches to `$lnull` iff `$lnull` is specified and the operand is `null`
  - passes cast operand along with branch


## Binary Format

TODO.


## JS API

See [GC JS API document](MVP-JS.md).
