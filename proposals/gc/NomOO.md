# MVP Support for Nominal OO Languages

Here we illustrate how the MVP would support a nominal object-oriented language.
Here we use Java as an example, specifically discussing the `Object`, `String`, and `Class` classes and the `Comparable` and `CharSequence` interfaces.


## Object

```
$Object
:= scheme.new
              (field vtable (gcref $Object_vtable)
                     readable immutable)
              (extensible hierarchical)
              nullable
              (equatable identity)
```

A Java object has a virtual-method table (the `vtable` field), supports reference equality (the `equatable identity` attribute), permits multi-tiered single class inheritance (the `extensible hierarchical` attribute), and has a `null` pointer (the `nullable` attribute).
Because `Object` in Java is not an abstract class, we need another scheme for actually allocating (exactly) `Object`s:

```
$object
:= scheme.new (parent implicit $Object)
              constructible
              (equatable identity)
```

This is a common pattern, so it might be worthwhile to develop a shorthand for it, but at this stage in the design it seems better to avoid confusions shorthands could cause.
For now, think of `$Object` as describing references with *static* type `Object`, and think of `$object` as describing references with *dynamic* type `Object`.


## Class Methods

This design implements dynamic dispatch on *class* methods using the standard technique of a virtual-method table (v-table).
The guaranteed contents of an object's v-table depends on that object's *class* type.
The more precise the class type is, the more contents that are guaranteed to be in the v-table.
Below is an illustration of what the v-table is guaranteed to have for any object (or for any object of any class that is a subclass of `Object`):

```
$Object_vtable
:= scheme.new
              (field itable (gcref $itable)
                     readable immutable)
              (field class (gcref $class_class)
                     readable immutable)
              (field getClass (func (param (gcref $Object)) (result (gcnref $Class)))
                     readable immutable)
              (field equals (func (param (gcref $Object) (gcnref $Object)) (result i32))
                     readable immutable)
              (field toString (func (param (gcref $Object)) (result (gcnref $String)))
                     readable immutable)
              ...
              extensible
```

Note that this scheme has no parent.
This indicates that v-tables are completely separate from objects.
As such, an engine might manage them very differently, as is common.
For example, its extensibility attribute is not castable, indicating that the engine does not need to store any casting information in the metadata.

The `itable` field references the interface table for the class (discussed below).
The `class` field references the reflection information for the class (discussed far below).

The `getClass`, `equals`, and `toString` fields each provide this class's implementation of the corresponding method of `Object`.
Each method's first parameter is a `gcref $Object`, which is used to provide the method implementation with the value of `this`.
Note that the parameter cannot be `null` because the class-method invocation process already guarantees that the value is non-null.

Lastly, every field in the v-table is `immutable`, enabling a number of optimizations and providing a reliably clean semantics for operations that are critical to the language runtime.
In some variants some fields might be `initializable` for boot-strapping purposes.
Or they might be `mutable` to enable some run-time optimizations, like optimized jitting or on-demand code loading.
Each of these use cases are worth considering in more detail, but we focus on the simplest case here.


## Interfaces and Interface Methods

This design implements dynamic dispatch on *interface* methods using the once-standard-but-now-outdated technique of an interface table (i-table).
An interface table is an array of interface-method tables, one for each interface implemented by the class.
An interface-method table (im-table) provides references to the class's implementations of the methods of the interface.
A dispatch of an interface method is implemented by looking up the i-table of the receiver, iterating through the array until the interface-method table for the given interface is found, looking up the corresponding method implementation from the table, and then passing the value of the receiver along with the other arguments to the method implementation.

```
$itable
:= scheme.new
              (field length interface_count (unsigned 8)
                     readable immutable)
              (field indexed interface (gcref $imtable)
                     readable immutable)
              constructible
```

The definition of `$itable` makes it very clear that this is an internal data structure: it has no parent, it is not extensible, and it is fully immutable.
It is also our first example of an array-like structure, with the `interface_count` field specifying the size of the array, and with the indexed `interface` field indicating the (in-this-case unique) indexed contents of the array.
The `unsigned 8` type of the length field indicates that the array can have at most 254 elements, thereby limiting classes in the source code to implementing at most 254 interfaces.
This small maximum length increases the chance that the length can be packed with the gc-metadata for instances of the scheme.

```
$imtable
:= scheme.new
              (extensible flat)
```

The definition of `$imtable` indicates that this scheme's sole purpose is to establish a `flat` casting scheme.
In fact, ideally this casting scheme would be done using fat pointers so that a reference can be cast without actually needing to be dereferenced.

```
$Comparable
:= scheme.new (parent implicit $imtable)
              (field compareTo (func (param (gcref $Object) (gcnref $Object)) (result i32))
                     readable immutable)
              constructible
$CharSequence
:= scheme.new (parent implicit $imtable)
              (field length (func (param (gcref $Object)) (result i32))
                     readable immutable)
              (field charAt (func (param (gcref $Object) i32) (result i32))
                     readable immutable)
              constructible
```

Every interface in the source language specifies a new child scheme of `$imtable` whose fields are the methods of the interface.
Note that the semantics ensures that this is done *nominally*, meaning an im-table for a given interface cannot be cast to an im-table of another interface that happens to have the same methods.
Also, even if an interface extends another interface, the definition of `$imtable` specifices a `flat` hierarchy in order to provide fast casting tests, so the `$imtable` will need have to distinct entries for the subinterface and the superinterface.
Often the redundancy this imposes is addressed by actually using integers indicating the offset at which the methods can be found in the v-table, but verifying the safety of this pattern requires advanced types that are beyond the scope of the MVP.

It is worth mentioning that more modern implementations of interface-method dispatch often use what is confusingly also called interface-method tables.
Rather than having an interface table, the v-table has some global-constant-prime-number of function pointers to interface-method implementations, which together comprise an interface-method table embedded within the v-table.
Every interface method has a pseudo-randomly predetermined index at which its implementations can be found within this table.
These implementations accept an additional argument that is the unique identifier of the interface method.
When an interface method is invoked, the implementation at the given index is called with the value of the receiver, the arguments, *and* the unique identifier of the method.
When the interface-method table is generated, if multiple distinct implemented interface methods happen to be assigned the same index, a conflict resolver is generated, and it examines this unique identifier to determine which implementation to tail-call to.
Unfortunately, because methods with completely different type signatures can collide, verifying this pattern requires advanced types that are beyond the scope of the MVP.


## Inheritance

`String` is an important subclass of `Object`.
In Java, strings are often implemented using some internal data structure, which we refer to as a `$charblock`.

```
$String
:= scheme.new (parent implicit $Object)
              (field refine vtable (gcref $String_vtable)
                     readable immutable)
              (field char_array (gcref $charblock)
                     readable immutable)
              (field start (unsigned 32)
                     readable immutable)
              (field end (unsigned 32)
                     readable immutable)
              castable
              constructible
              nullable
              (equatable identity)
$charblock
:= scheme.new
              (field length numchars (unsigned 32)
                     readable immutable)
              (field indexed char (unsigned 16)
                     readable immutable)
              constructible
$String_vtable
:= scheme.new (parent implicit $Object_vtable)
              (field length (func (param (gcref $String)) (result i32))
                     readable immutable)
              (field charAt (func (param (gcref $String) i32) (result i32))
                     readable immutable)
              ...
              constructible unique
```

The most important detail to observe is that the `vtable` field is *refined* to be a `gcref $String_vtable`, taking advantage of the covariance permitted by the fact that `vtable` is only `readable` and not `writeable`.
This in particular means that if a `gcref $Object` is cast to a `gcref $String`, then we automatically learn that its v-table has the additional methods associated with strings.
(Note, though, that if the v-table has already been loaded from the object, then the MVP's type system is not intelligent enough to recognize that this learned information extends to that previously loaded reference.
For that we need fancy types that are beyond the scope of the MVP.)
Of course, because `String` is a final class in Java, methods on values with static type `String` would never be dispatched indirectly through the v-table but rather directly to the corresponding implementation.
Nonetheless, this ability to refine the v-table is important for good performance of class-method dispatches on non-final classes.

The second-most important detail to observe is that the function fields inherited from `$Object_vtable` are *not* refined to accept a `gcref $String` as the `this` parameter.
Even though the source language can ensure this invariant, verifying the safety of this pattern requires fancy types that are beyond the scope of the MVP.
As such, the `String`'s implementations for these inherited methods will generally first need to cast the `this` parameter to a `gcref $String`, which is why it is important we ensure that such casts can be performed efficiently.
On the other hand, the signatures for non-inherited methods like `length` and `charAt` can use the more precise type `gcref $String` for the `this` parameter.

Lastly, making `$String` and `$String_vtable` be `constructible` rather than `extensible` encodes the finality of the `String` class.

Not visible above is the code for allocating Strings, the String v-table, and the String i-table/im-tables.
The String im-tables have the same problem that the `this` parameter must have type `gcref $Object` and therefore the implementations of interface methods must be cast and will likely tail-call to the main implementation for which the `this` pointer already has the more precise type.
The allocation of the String v-table can use the `gcref.construct_copy` to bulk copy the standard implementations of many `Object` methods and initialize the remaining methods to the String-specific implementations.
Note that this instruction would even enable one to construct the String v-table without knowing all the fields of the method v-table, which will be important for modularization and incremental compilation.


## Arrays

Arrays in Java are essentially a special kind of final class.
Importantly, arrays need to have v-tables in order to be treated as Objects.
Besides that, arrays are fairly straightforward, following into two subcategories: primitive arrays and reference arrays.

```
$int_array
:= scheme.new (parent implicit $Object)
              (field length length (unsigned 32)
                     readable immutable)
              (field indexed element i32
                     readable writeable mutable)
              castable
              constructible
```

As an example of a primitive array, `$int_array` defines the scheme for `int[]` Java values.
It has a v-table because its parent scheme, `$Object`, has a v-table.
Its `length` is capped at 32 bits due to the fact that Java's `length` field for arrays is capped at 31 (unsigned) bits.
Its `indexed` field is finally our first example of a `mutable` field, which emphasizes the fact that immutability is extremely common in behind-the-scenes data structures of even languages known for their heavy use of mutability, and as such it is worthwhile to make a design that supports and optimizes for immutability.

```
$reference_array
:= scheme.new (parent implicit $Object)
              (field element_class (gcref $reference_class)
                     readable immutable)
              (field length length (unsigned 32)
                     readable immutable)
              (field indexed element (gcnref $Object)
                     readable writeable mutable)
              castable
              constructible
```

There are two ways in which arrays could be designed.
Unfortunately, due to either Java's use of covariant arrays or Java's feature of generics, one needs fancy types that are beyond the scope of the MVP to verify the pattern that ensures that a `String[]` contains only `String`s.
As such, even though code compiled from Java will ensure this invariant, the MVP will have to operate as if any reference array can always contain an arbitrary `Object`.

This leads us to the `$reference_array` scheme above.
This scheme has an additional (non-indexed) `element_class` field that references a *custom* data structure describing run-time type information that indicates what kind of array the instance is meant to be, e.g. exactly an `Object[]` or exactly a `String[]`.
Typically, when an object gets written into the array in the source code, the compiled WebAssembly code will first use this run-time type information to test whether the object has the appropriate run-time type for the array instance at hand (see below for details as to how this test is done).
Thus the compilation strategy for Java to WebAssembly is responsible for maintaining the invariant that a `String[]` contains only `String`s, rather than the WebAssembly engine being responsible for this invariant.


## Reflection

Java provides a way to reflect upon the classes, interfaces, methods, and other constructs of the program.
The primary gateway into this reflection mechanism is the `Class` class.
This class is final, meaning that users cannot define custom extensions of the class, but behind the scenes it makes sense to implement the class using an internal algebraic data type.

```
$Class
:= scheme.new (parent implicit $Object)
              (field refine vtable (gcref $Class_vtable)
                     readable immutable)
              castable
              (extensible (cases $prim_class $void_class $reference_class))
              nullable
$Class_vtable
:= scheme.new (parent implicit $Object_vtable)
              (field isInstance (func (param (gcref $Class) (gcnref $Object)) (result i32))
                     readable immutable)
              ...
              constructible
$prim_class := ...
$void_class := ...
```

In this design, the `$Class` scheme has three cases: primitives (e.g. int, double, boolean&mdash;*not* `Integer`, `Double`, `Boolean`), void, and references (i.e. classes, interfaces, and arrays).
The `Class` class also provides one particularly important method: `isInstance`.
The implementation of this method takes a `$Class` instance and an `$Object` instance and returns a boolean indicating whether the object is an instance of the run-time type represented by the class.
This is also precisely the sort of code we need for implementing the test conducted before assigning an object into an array.
As such, we focus exclusively on how to implement `isInstance`, putting aside the many other facets of reflection here.
Note that this is trivial for primitives and void, since no object has those types, which is why we skip the definitions of those schemes above.
Similarly, the case where the value being cast is `null` is trivial, so here on out we assume that case has been addressed.

```
$reference_class
:= scheme.new (parent explicit $Class)
              castable
              (extensible (cases $class_or_interface_or_prim_array_class $reference_array_class))
```

Notice first that the `$reference_class` scheme declares `$Class` to be an `explicit` parent.
This enables an engine to use an optimized packed representation for `$reference_class`, an optimization that is worthwhile because the `element_class` field of an array is more precisely a `$reference_class`.
Second, the `$reference_class` scheme itself has two cases: classes or interfaces or primitive arrays, and reference arrays.
These two cases were chosen because each case has a distinctively different strategy for implementing `isInstance`.

```
$class_or_interface_or_prim_array_class
:= scheme.new (parent explicit $reference_class)
              (field isInstance (func (param (gcref $Object)) (result i32))
                     readable immutable)
              castable
              (extensible (cases $class_or_interface $prim_array))
$class_or_interface
:= scheme.new (parent explicit $class_or_interface_or_prim_array_class)
              ...
              castable
              extensible (cases $class_class $interface_class)
$class_class := ...
$interface_class := ...
$prim_array_class := ...
```

The `$class_or_interface_or_prim_array_class` scheme introduces an `isInstance` field (rather than method).
This field references a function that takes an object (but no `Class`) and returns what is ostensibly a boolean.
What that function does, though, depends on whether the type being represented is a class, interface, or primitive array.

For both classes and primitive arrays, all the `isInstance` function does is apply the `gcref.test` instruction instantiated with the scheme of the given class or primitive array.
One might consider providing a first-class value that represents schemes for this purpose, but such a construct would significantly complicate the spec (even more), could impose significant restrictions on engines, and seems like a lot of effort for something that can be easily accomplished using a (presumably quick) function call and which is only used in a corner case to begin with.

For interfaces, all the `isInstance` function does is go through each of the im-tables in the object's i-table and tests whether they are instances of the im-table scheme corresponding to the interface at hand.
This example, along with reference arrays, demonstrates that casts in the source language often do not correspond with casts at the WebAssembly level.
That said, the examples of classes and primitive arrays demonstrate that sometimes source-level and WebAssembly level casts can align, in which case exploiting this alignment enables the engine to provide more efficient implementations of such source-level casts.

```
$reference_array_class
:= scheme.new (parent explicit $Class)
              (field element_class (gcref $reference_class)
                     readable immutable)
              constructible
              castable
```

The `$reference_array_class` scheme introduces an `element_class` field.
If this instance is supposed to denote `String[]`, then `element_class` references the run-time representation of `String`.
In this case, a cast is implemented by checking whether the given object is an instance of `$reference_array` and, if so, checking whether its `element_class` value represents a subtype of this reference-array-class's `element_class` (the implementation of which we do not go into).


## Null

The `null` value in Java is special in that it can be considered to have any reference type in Java.
Some languages even have a type `Null` that is a subtype of any (nullable) reference type.
The proposal provides this functionality through the `scheme.null $Object` instruction and the `gcnull $Object` type.
We found it necessary to parameterize `gcnull` by a scheme to prevent imposing constraints on how unrelated but `nullable` schemes can be represented at the pointer level (e.g. one scheme might be represented using standard pointers while another scheme might be represented using fat pointers).
However, `gcnull` is *bi*variant with respect to `implicit` parent-child relationships (which must share the same pointer represenations).
This means that `gcnull $Object` is a subtype of `gcnull $String`, making it a subtype of any (nullable) reference type of the source language.


## Constructors

Object initialization has been a long-standing pain point for language designers, and some of those pain points translate into challenges for a GC proposal.
In particular, ideally one would like to ensure that every field gets initialized before they are accessed and that immutable fields get initialized only once, while at the same time letting constructors of subclasses focus only on fields specific to the subclass and defer initializing fields of the superclass to its constructor.

The simplest way to solve this problem at the WebAssembly level is just to make all fields mutable and hand constructors a reference whose fields they can mutate to their needs.
Of course, this solution comes at the cost of denying engines all optimization opportunities that should be possible since many of these fields are ostensibly immutable.
Less obviously, this solution requires all fields to be defaultable.
More and more object-oriented languages are eschewing implicit `null` values, and as such they should be able to have non-null fields, but these are not defaultable.

This proposal includes two instructions to try to address some of these problems.
`scheme.construct_default` constructs an instance using default values for all fields but for a few explicit exceptions.
`scheme.construct_copy` constructs an instance by copying the non-explicitly-excluded immutable fields of an instance of a related scheme.
When we introduce scheme imports, both these instructions will enable one to construct instances of a scheme whose parent is imported without requiring the parent to export all its (encapsulated) contents.
And, for use cases still not covered by these instructions, we provide an `initializable` notion of mutability that only lets fields be mutated once&mdash;from their default value.
This enables compare-and-swap implementations on multi-threaded engines and still supports many of the same optimizations that immutability enables.

Of course, the most flexible solution would be to have some way to reason about partially initialized instances.
Unfortunately, verifying the safety of this pattern requires fancy types that are beyond the scope of the MVP.


## Object Identity

This proposal does not address the issue of efficiently providing an integer that denotes the identity of an object, i.e. `System.identityHashcode(Object)` in Java.
So far, the proposed design ensures deterministic behavior, and it is unclear how to address this issue deterministically.
Nonetheless, it is a feature with many important uses, so we should decide upon a solution for it.


## Packing Numeric Classes

Here we consider a slightly different design for Java with respect to primitive types, one that corresponds to more recent object-oriented language designs.
In this design, there is no surface-level distinction between primitive types and reference types.
In other words, the primitive `int` and the class `Integer` are the same.
Instead, one relies on a combination of static type information and fancier low-level representations to avoid, or at least reduce, boxing and unboxing.

One key enabler for this optimization is a corresponding surface-level change to the semantics of equality.
In particular, when comparing the *identities* of two objects, if those objects represent primitives then we need to compare their contents deeply.
That way we get the same behavior regardless of whether those primitive values happen to be boxed or unboxed.
Note that if an engine consistently packs the same primitive values, then it can still implement equality comparison without any memory loads in the common case.

```
$Object
:= scheme.new
              (extensible (cases $primitive $reference))
              nullable
              (equatable case)
```

The above is the redesigned scheme for `$Object`.
Note that it now indicates that its equality is defined `case`-wise, enabling child schemes to have their choice of which notion of equality works best for their needs.
Note also that it no longer has a `vtable` field; this design instead supports method dispatch by case, utilizing the `cases` extensibility guarantee that every instance is either a `$primitive` (which itself is has a finite number of cases) or a `$reference`.
So whenever `toString()` is invoked on a receiver of static type `Object` or unbounded type variable, this will translate to a switch on the primitive case types and the `$reference` scheme.
The same goes for methods of the `Number` class and the `Comparable` interface, since primitive values inhabit those types.
But if the receiver has a static type like `List`, no primitive value implements `List`, and therefore this switch can be elided even for methods like `toString()`.

```
$primitive
:= scheme.new (parent explicit $Object)
              castable
              (extensible (cases $boolean $char $byte $short $int $long $float $double))
              (equatable case)
$boolean
:= scheme.new (parent explicit $primitive)
              (field value (unsigned 1)
                     readable immutable)
              castable
              constructible
              (equatable deep)
$char := ...
$byte := ...
$short := ...
$int := ...
$long := ...
$float := ...
$double := ...
```

The `$primitive` scheme itself is comprised of a variety of cases, one for each primitive type.
It supports only `deep` equality, which forces all of its child schemes to support only `deep` equality, which in turn ensures they have only `immutable` fields whose types have shallowly-checkable notions of equality.
This means that these instances can be packed rather than allocated on the heap when enough bits are available.
The fact that `$primitive` has only 8 cases informs the engine that it needs only 3 bits to determine which kind of primitive value a packed representation denotes.
`$primitive`  is an `explicit` child of `$Object`, enabling a more optimized packed representation.
As an example of a specific primitive case, we show the `$boolean` scheme for boolean values.

```
$reference
:= scheme.new (parent explicit $Object)
              (field vtable (gcref $Object_vtable)
                     readable immutable)
              (extensible hierarchical)
              nullable
              (equatable identity)
```

The `$reference` scheme is essentially the same as the original version of the `$Object` scheme; the only difference is that it is an `explicit` child of the new version of the `$Object` scheme.

```
$Object_vtable
:= scheme.new
              (field itable (gcref $itable)
                     readable immutable)
              (field class (gcref $class_class)
                     readable immutable)
              (field getClass (func (param (gcref $reference)) (result (gcnref $Class)))
                     readable immutable)
              (field equals (func (param (gcref $reference) (gcnref $Object)) (result i32))
                     readable immutable)
              (field toString (func (param (gcref $reference)) (result (gcnref $String)))
                     readable immutable)
              ...
              extensible
```

The scheme of `$Object_vtable` can be optimized a bit with respect to this redesign.
In particular, the `this` parameter for each method can have type `gcref $reference`, reflecting the guarantee that v-tables are only used to implement class-method dispatch on non-primitive values.
Also, although not apparent here, the `$Class` and `$String` schemes are children of `$reference` since no primitive values inhabit `Class` or `String`.


## Packing Enums

We can take this technique a step further in order to pack Java-style enums.
The main challenge is that Java enums are unbounded (i.e. there are not a finite number of enum types) and can implement interfaces (and so need some way to provide an interface table).
The latter requires us to implement interface-method dispatch for enums, and the former prevents us from doing so via switches, as we did for primitives.
So instead we must have a field indicating which enum type a given enum value belongs to, which we can then use to get an interface table.

```
$Object
:= scheme.new
              (extensible (cases $primitive $nonprimitive))
              nullable
              (equatable case)
$nonprimitive
:= scheme.new (parent explicit $Object)
              (extensible (cases $Enum $reference))
              nullable
              (equatable case)
$Enum
:= scheme.new (parent explicit $nonprimitive)
              (field enum_class (unsigned 8)
                     readable immutable)
              (field enum_case (unsigned 8)
                     readable immutable)
              constructible
              nullable
              (equatable deep)
$enum_itables
:= scheme.new
              (field (indexed 256) itable (gcnref $itable)
                     readable initializable)
              constructible unique
```

This redesign makes `$Object` have two cases: `$primitive` and `$nonprimitive`.
Because enums can implement arbitrary interfaces, `$nonprimitive` is the scheme to use for any reference whose surface-level static type is an interface or `Object`.
If the surface-level static type is `Enum` or any specific enum class, then `$Enum` is the scheme to use.
The schemes of other classes (besides `Number`) should be descendents of `$reference`.

To implement interface-method dispatch in this design, if the instance is an `$Enum`, then one uses the value of `enum_class` to fetch an `$itable` from the indexed `itable` field of some global `gcref $enum_tables` variable.
The reason `enum_class` is not simply a `gcref $itable` is because an 8-bit unsigned integer is more likely to be packed.
(Note that the `itable` field is declared to be `indexed 256`.
This uses an extension to consider for enabling arrays of statically-determined length.)

A major downside of this design is that it caps the number of enum classes to 256.
Similarly, it caps the number of enum cases to 256.
Possibly more importantly, it requires sequencing the initialization of enum classes so that each class gets a unique identifier.
This is not so problematic for self-contained modules, but will be more problematic when one links together modules that each define their own enum classes.
Note that `$enum_itables` makes use of `initializable` mutability to enable enum classes to be initialized incrementally (across modules) while still enabling many immutability optimizations.

These downsides could be avoided by providing some way to specify and ensure that the value of a field is completely determined by the exact scheme of an instance.
Let us call this kind of field a `scheme` field.
Then we could define `vtable` to be a `scheme` field.
These `scheme` fields can be disregarded when packing because an engine can use the packed casting information to fetch the value of these fields when needed.
This functionality is essentially what the above designs for primitives and enums are manually implementing, but which can often be much more efficiently implemented by the engine directly.


## Interop

One recurring question is how the GC proposal can be used for interoperability between different WebAssembly modules and between WebAssembly and external systems such as JavaScript and the DOM.
In the designs above, `$Object` has had no parent.
But one could imagine importing an `extensible explicit` scheme, say `$root`, and declaring `$Object` to have `parent explicit $root`.
This would inform the engine to ensure that all instances allocated on the heap have (room for) associated metainformation required by the `$root` scheme.
At the same time, the `explicit` qualifier enables the engine to still use a specialized packing scheme throughout the internals of the module (and to only generate required metainformation for heap-allocated references on demand).
Obvious examples of `$root` are the scheme for `anyref` and the scheme used for JavaScript objects (providing a path for a [GC JS API](MVP-JS.md), though that file has yet to be updated).
