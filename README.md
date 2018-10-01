![Immutability - A Go Language Feature Proposal](https://github.com/romshark/Go-2-Proposal---Immutability/blob/master/document_header.png "Immutability - A Go Language Feature Proposal")

# Go - Immutability
This document describes a language feature proposal to immutability for the [Go
programming language](https://golang.org). The proposed feature targets the
current [Go 1.x (> 1.11) language specification](https://golang.org/ref/spec)
and doesn't violate [the Go 1 compatibility
promise](https://golang.org/doc/go1compat). It also describes [an even better
approach to immutability](#3-immutability-by-default-go--2x) for a hypothetical, backward-incompatible [Go 2
language specification](https://blog.golang.org/toward-go2).

<table>
	<tr>
		<td><b>Author</b></td>
		<td>
			<a href="https://github.com/romshark">Roman Sharkov</a> (<a href="mailto:roman.sharkov@qbeon.com">roman.sharkov@qbeon.com</a>)
		</td>
	</tr>
	<tr>
		<td><b>Status</b></td>
		<td>Draft</td>
	</tr>
	<tr>
		<td><b>Version</b></td>
		<td>v0.1.0</td>
	</tr>
</table>

**Table of Contents**
- [Go - Immutability](#go---immutability)
	- [1. Introduction](#1-introduction)
		- [1.1. Current Problems](#11-current-problems)
			- [1.1.1. Ambiguous Code and Dangerous Bugs](#111-ambiguous-code-and-dangerous-bugs)
			- [1.1.2. Vague and Bloated Documentation](#112-vague-and-bloated-documentation)
			- [1.1.3. The - Slow but Safe vs Dangerous but Fast - Dilemma](#113-the---slow-but-safe-vs-dangerous-but-fast---dilemma)
			- [1.1.4. Inconsistent Concept of Constants](#114-inconsistent-concept-of-constants)
		- [1.2. Benefits](#12-benefits)
			- [1.2.1. Safe and Precise Code](#121-safe-and-precise-code)
			- [1.2.2. Self-Explaining Code](#122-self-explaining-code)
			- [1.2.3. Increased Runtime Performance](#123-increased-runtime-performance)
	- [2. Proposed Language Changes](#2-proposed-language-changes)
		- [2.1. Immutable Fields](#21-immutable-fields)
		- [2.2. Immutable Methods](#22-immutable-methods)
		- [2.3. Immutable Arguments](#23-immutable-arguments)
		- [2.4. Immutable Return Values](#24-immutable-return-values)
		- [2.5. Immutable Variables](#25-immutable-variables)
		- [2.6. Immutable Interface Methods](#26-immutable-interface-methods)
		- [2.7. Slice Aliasing](#27-slice-aliasing)
		- [2.8. Address Operators](#28-address-operators)
			- [2.8.1. Taking the address of a variable](#281-taking-the-address-of-a-variable)
			- [2.8.2. Dereferencing a pointer](#282-dereferencing-a-pointer)
		- [2.9. Immutable Reference Types](#29-immutable-reference-types)
			- [2.9.1. Pointer Examples](#291-pointer-examples)
				- [2.9.1.1. Immutable pointer to a mutable object](#2911-immutable-pointer-to-a-mutable-object)
				- [2.9.1.2. Mutable pointer to an immutable object](#2912-mutable-pointer-to-an-immutable-object)
				- [2.9.1.3. Immutable pointer to an immutable object](#2913-immutable-pointer-to-an-immutable-object)
			- [2.9.2. Slice Examples](#292-slice-examples)
				- [2.9.2.1. Immutable slice of immutable objects](#2921-immutable-slice-of-immutable-objects)
				- [2.9.2.2. Mutable slice of immutable objects](#2922-mutable-slice-of-immutable-objects)
				- [2.9.2.3. Immutable slice of mutable objects](#2923-immutable-slice-of-mutable-objects)
				- [2.9.2.4. Mutable slice of mutable objects](#2924-mutable-slice-of-mutable-objects)
			- [2.9.3. Map Examples](#293-map-examples)
				- [2.9.3.1. Mutable map of immutable keys to mutable objects](#2931-mutable-map-of-immutable-keys-to-mutable-objects)
				- [2.9.3.2. Mutable map of mutable keys to immutable objects](#2932-mutable-map-of-mutable-keys-to-immutable-objects)
				- [2.9.3.3. Mutable map of immutable keys to immutable objects](#2933-mutable-map-of-immutable-keys-to-immutable-objects)
				- [2.9.3.4. Immutable map of immutable keys to immutable objects](#2934-immutable-map-of-immutable-keys-to-immutable-objects)
			- [2.9.4. Channel Examples](#294-channel-examples)
				- [2.9.4.1. Immutable channels of immutable objects](#2941-immutable-channels-of-immutable-objects)
				- [2.9.4.2. Immutable channels of mutable objects](#2942-immutable-channels-of-mutable-objects)
				- [2.9.4.3. Mutable channels of immutable objects](#2943-mutable-channels-of-immutable-objects)
		- [2.10. Type Casting](#210-type-casting)
			- [2.10.1. Simple Casting](#2101-simple-casting)
			- [2.10.2. Literal Type Casting](#2102-literal-type-casting)
			- [2.10.3. Prohibition of Casting Immutable- to Mutable Types](#2103-prohibition-of-casting-immutable--to-mutable-types)
		- [2.11. Implicit Casting](#211-implicit-casting)
			- [2.11.1. Implicit Casting of Pointer Receivers](#2111-implicit-casting-of-pointer-receivers)
	- [3. Immutability by Default (Go >= 2.x)](#3-immutability-by-default-go--2x)
		- [3.1. Benefits](#31-benefits)
			- [3.1.1. Safety by Default](#311-safety-by-default)
			- [3.1.2. No Explicit Casting](#312-no-explicit-casting)
			- [3.1.3. Less Code](#313-less-code)
			- [3.1.4. No `const` Keyword Overloading](#314-no-const-keyword-overloading)
	- [4. FAQ](#4-faq)
		- [4.1. Are the items within immutable slices/maps also immutable?](#41-are-the-items-within-immutable-slicesmaps-also-immutable)
		- [4.2. Go is all about simplicity, so why make the language more complicated?](#42-go-is-all-about-simplicity-so-why-make-the-language-more-complicated)
		- [4.3. Aren't other features such as generics and better error handling not more important right now?](#43-arent-other-features-such-as-generics-and-better-error-handling-not-more-important-right-now)
		- [4.4. Why overload the `const` keyword instead of introducing a new keyword like `immutable` etc.?](#44-why-overload-the-const-keyword-instead-of-introducing-a-new-keyword-like-immutable-etc)
		- [4.5. How are constants different from immutables?](#45-how-are-constants-different-from-immutables)
		- [4.6. Why do we need immutable receivers if we already have copy-receivers?](#46-why-do-we-need-immutable-receivers-if-we-already-have-copy-receivers)
		- [4.7. Why do we need immutable interface types?](#47-why-do-we-need-immutable-interface-types)
		- [4.8. Doesn't the `const` qualifier add boilerplate and make code harder to read?](#48-doesnt-the-const-qualifier-add-boilerplate-and-make-code-harder-to-read)
		- [4.9. Why do we need the distinction between immutable and mutable reference types?](#49-why-do-we-need-the-distinction-between-immutable-and-mutable-reference-types)
		- [4.10. Why not implicitly convert mutable to immutable types?](#410-why-not-implicitly-convert-mutable-to-immutable-types)
	- [5. Other Proposals](#5-other-proposals)
		- [5.1. proposal: spec: add read-only slices and maps as function arguments #20443](#51-proposal-spec-add-read-only-slices-and-maps-as-function-arguments-20443)
			- [5.1.1. Disadvantages](#511-disadvantages)
			- [5.1.2. Similarities](#512-similarities)
		- [5.2. proposal: Go 2: read-only types #22876](#52-proposal-go-2-read-only-types-22876)
			- [5.2.1. Disadvantages](#521-disadvantages)
			- [5.2.2. Differences](#522-differences)
			- [5.2.3. Similarities](#523-similarities)

## 1. Introduction
Immutability is a technique used to prevent mutable shared state, which is a
very common source of nasty, hard to find and hard to even identify bugs,
especially in concurrent environments. It can be achieved through **immutable
types**. This proposal is based on 5 fundamental rules:

- **I.** Each and every type has an immutable counterpart.
- **II.** Assignments to objects of an immutable type are illegal.
- **III.** Calls to methods with a mutable receiver type on objects of an
  immutable type are illegal.
- **IV.** Mutable types can be casted to their immutable counterparts, but not
  the other way around.
- **V.** Immutable interface methods must be implemented by a method with an
  immutable receiver type.

These rules can be enforced by making the compiler scan all objects of immutable
types for modification attempts such as assignments and calls to mutating
methods and fail the compilation if any illegal mutation attempts were
identified. The compiler would also need to check, whether types correctly
implement immutable interface methods.

A Go 1.x developer's current approach to immutability is manual copying because
Go 1.x doesn't currently provide any compiler-enforced immutable types. This
makes Go harder to work with because mutable shared state can't be easily dealt
with.

Ideally, a safe programming language should enforce [immutability by
default](#3-immutability-by-default-go--2x) where all types are immutable unless
they're explicitly annotated as mutable. This concept would require significant,
backward-incompatible language changes breaking existing Go 1.x code, thus such
an approach to immutability would only be possible in a new
backward-incompatible Go 2.x language specification.

To prevent breaking Go 1.x compatibility this document describes a
backward-compatible approach to adding support for immutable types by
overloading the `const` keyword ([see here for more
details](#44-why-overload-the-const-keyword-instead-of-introducing-a-new-keyword-like-immutable-etc))
to act as an immutable type qualifier. There is no runtime cost to this
approach, but a negligible compile-time cost is still required.

### 1.1. Current Problems

The current approach to immutability (namely copying) has a number of
disadvantages listed below and sorted by importance in descending order.

#### 1.1.1. Ambiguous Code and Dangerous Bugs
The absence of immutable types can lead to ambiguous code that results
in dangerous, hard to find bugs. Consider the following method definition:

```go
// Method ...
func (r *T) Method(
	a *T,
	b *T,
	v []*T,
) (rv *T) {/*...*/}
```
The above code is ambiguous, it doesn't represent the intentions of its original
author:
- Will it produce any side-effects on `r`?
- Will it mutate the `T`s referenced by `a` and `b`?
- Will it mutate `v`?
- Will it mutate any `T` referenced by any item of `v`?
- Is the `T` referenced by `rv` allowed to be mutated?

All those questions can lead to bugs if they're not properly answered, and
[documentation never answers them
reliably](#112-vague-and-bloated-documentation))

If the above function is exported from a 3-rd party package `xyz` that's
imported to a project `P` as an external dependency and the documentation
promises (or "claims") to not mutate any of the symbols, the code in `P` will be
written with this assumption in mind.

At any time the vendor of `xyz` might change its behavior, either intentionally
or unintentionally, which will introduce bugs and data corruption:
- `a`, `b`, `v` or any items of `v` might get aliased and mutated either
  directly (in the scope of `xyz.T.Method`) or indirectly (at an unspecified
  point in time).
- New side-effects could be introduced to `r`.
- `rv` might get aliased and introduce unwanted side-effects when mutated by the
  `xyz.T.Method` caller.

In the worst case, the maintainers of `P` won't even be informed about the
mutations that were unintentionally introduced to a newer version of
`xyz.T.Method` through a bug in its implementation. But even if the vendors of
`xyz` correctly update the changelog and the documentation introducing new
intentional side-effects, chances are high that the maintainers of `P` miss the
changes in the documentation and fail to react accordingly. `P` will continue to
compile, but its **outputs will become corrupted** which can't always be easily
detected even in the presence of extensive automated testing.

#### 1.1.2. Vague and Bloated Documentation
Documentation never represents **actual** intentions, it represents **claimed**
intentions. Claimed intentions can't be relied on, because claims are not
guaranteed to remain in sync with actual code behavior.

To avoid ambiguous code developers often describe *mutability recommendations*
of variables, fields, arguments, methods and return values. Not only does this
unnecessarily complicate and bloat up the documentation, but it also makes it
error-prone and redundant. Documentation can easily get out of sync with the
actual code because it can't be verified algorithmically.

#### 1.1.3. The - Slow but Safe vs Dangerous but Fast - Dilemma
Copies are the only way to achieve immutability in Go 1.x, but copies inevitably
degrade runtime performance. This dilemma encourages Go 1.x developers to either
write unsafe mutable APIs when targeting optimal runtime performance or safe but
slow and copy-code bloated ones.

Optional performance and code safety are, currently, mutually exclusive, even
though having both would be possible with compiler-enforced immutable types at
the cost of a slightly decreased compilation time.

#### 1.1.4. Inconsistent Concept of Constants
Currently, Go 1.x won't allow non-scalar constants such as constant slices:
```
const each2 = []byte{'e', 'a', 'c', 'h'} // Compile-time error
```

[Robert Griesemer](https://github.com/griesemer) stated in his comment to
[proposal #6386](https://github.com/golang/go/issues/6386) that this is by
language design, quote:
> This is neither a defect of the language nor the design. The language was
_deliberately_ designed to only permit constant of basic types.

But many developers still claim it to be a major design flaw because exclusive
immutability for scalar types only leads to inconsistency in the language
design. What most developers really need is not *constants of arbitrary types*
but rather **immutable package-scope variables**, which can be implemented
consistently with the help of immutable types:
```
var each2 const []byte = const([]byte{'e', 'a', 'c', 'h'})
```
Even though technically `each2` is not a *constant* but an *immutable
package-scope variable* - it solves the mutability problem.

### 1.2. Benefits
Support for immutable types would provide the benefits listed below and sorted
by importance in descending order.

#### 1.2.1. Safe and Precise Code
Immutable types make APIs less ambiguous.

With immutable types the situations described in the [previous
section](#111-ambiguous-code-and-dangerous-bugs) wouldn't even be possible,
because the author of the function of the external package would need to
explicitly denote immutable types as such to make the compiler enforce the
guarantee:

```go
// Method ...
func (r * const T) Method(
	a * const T,
	b * T,
	const v [] * const T,
) (
	rv * const T,
	rv2 * T,
) {
	/*...*/
}
```
The above code is unambiguous and precise. It clearly represent the intentions
of its original author and answers all critical questions reliably:
- Will it produce any side-effects on `r`?
    - **No, it can't. It's receiver is immutable**
- Will it mutate the `T` referenced by `a`?
    - **No, it can't. The `T` referenced by `a` is immutable**
- Will it mutate `v`?
	- **No, it can't. `v` is immutable**
- Will it mutate any `T` referenced by any item of `v`?
    - **No, it can't. The `T`s referenced by any item of `v` are immutable**
- Is the `T` referenced by `rv` allowed to be mutated?
	- **No, it's not. The `T` referenced by `a` is immutable**
- Will it mutate the `T` referenced by `b`?
	- **Yes, it potentially will!**
- Is the `T` referenced by `rv2` allowed to be mutated?
	- **Yes, it won't lead to unwanted side-effects**

Whenever a mutable type is taken, returned or provided it's assumed that its
state will potentially be mutated.

The user of this function would make decisions based on the actual function
definition in the code instead of relying on the potentially inconsistent
documentation.

If the vendors of this function decide to change the mutability of either an
input or output type or the mutability of the object the method operates on -
they will have to change the type introducing breaking API changes causing
compiler errors and making the user pay attention to whether or not everything's
right.

The vendors won't be able to just silently introduce mutations causing bugs! The
compiler will prevent this from happening either before the vendors release the
update (assuming that the code is compiled by a CI/CD system before publication)
or during the user's local build in the worst case.

#### 1.2.2. Self-Explaining Code
With immutable types, there's no need to explicitly describe mutability
recommendations in the documentation. When immutable types are declared as such
then the code becomes self-explaining:
- An **argument** or a **variable** of an immutable type can be relied on not
  being changed neither inside the scope it's declared in, nor in the scopes
  it's passed to.
- An immutable **method** (or a "function with a **receiver** of an immutable
  type" if you will) - can be relied on not changing the object it operates on.
- A **return value** of an immutable type can be relied on not being changed by
  the function caller.
- A **field** of an immutable type can be relied on not being changed as soon as
  the object is initialized, even inside the scope of its origin package.

#### 1.2.3. Increased Runtime Performance
Immutability provides a way to safely avoid unnecessary copying as well as
unnecessary indirections through mutable and immutable interfaces (because
interfaces do have a cost).

Immutability also makes specific compiler optimizations possible. Whether or not
those optimization opportunities are exploited later on is rather irrelevant to
this particular proposal.

## 2. Proposed Language Changes
The language must be adjusted to support the `const` qualifier
inside type definitions to denote certain types as immutable.

The compiler must enforce the following rules:
- Assignments to objects of an immutable type are illegal.
- Calls to methods with a mutable receiver type on objects of an immutable type
  are illegal.
- Immutable types cannot be casted to their mutable counterparts.
- Types must implement immutable interface methods using an immutable receiver
  type.
- Mutable types must be explicitly casted to their immutable counterparts.
- During method calls - pointer receivers must be implicitly casted in both
  directions (mutable to immutable and vice-versa) if the types of the objects
  they're pointing to are equal.

It is to be noted, that all proposed changes are fully backward-compatible and
don't require any breaking changes to be introduced to the Go 1.x language
specification. Code written in previous versions of Go 1.x will continue to
compile and work as usual.

### 2.1. Immutable Fields
Immutable struct fields are declared using the `const` qualifier. Immutable
fields can only be set during the definition of the object and are then
immutable for the entire lifetime of the object within any context.

```go
type Object struct {
	ImmutableField const * const Object // Immutable
	MutableField   *Object       // Mutable
}

// MutatingMethod is a non-const method
func (o *Object) MutatingMethod() {
	o.MutableField = &Object{}
	o.ImmutableField = &Object{} // Compile-time error
}

func main() {
	// Immutable fields are immutable once the object is initialized
	obj := Object{
		ImmutableField: &Object{
			ImmutableField: nil,
		},
	}

	obj.MutableField = &Object{}
	obj.ImmutableField = nil                      // Compile-time error
	obj.ImmutableField.ImmutableField = &Object{} // Compile-time error
}
```
**Expected compilation errors:**
```
.example.go:9:23 cannot assign to immutable field `Object.ImmutableField` of type `const * const Object`
.example.go:21:25 cannot assign to immutable field `Object.ImmutableField` of type `const * const Object`
.example.go:22:40 cannot assign to immutable field `Object.ImmutableField` of type `const * const Object`
```

----

### 2.2. Immutable Methods
Immutable methods are declared using the `const` qualifier on the
function-receiver and guarantee to not mutate the receiver in any way when
called. They can safely be used in immutable contexts, such as within other
immutables methods and/or on immutable objects.

Technically, this feature should rather be called "immutable function
receivers".

```go
type Object struct {
    mutableField *Object // Mutable
}

// MutatingMethod is a non-const method.
func (o *Object) MutatingMethod() const * const Object {
    o.mutableField = &Object{}
    return o.ImmutableMethod()
}

// ImmutableMethod is a const method.
// It's illegal to mutate any fields of the receiver.
// It's illegal to call mutating methods of the receiver
func (o * const Object) ImmutableMethod() const * const Object {
    o.MutatingMethod()                      // Compile-time error
    o.mutableField = &Object{}              // Compile-time error
    o.mutableField.mutableField = &Object{} // Compile-time error
    return const * const Object(o.mutableField)
}

func main() {
    obj := * const Object (&Object{})
    obj.ImmutableMethod()
    obj.MutatingMethod() // Compile-time error
}
```
**Expected compilation errors:**
```
.example.go:15:7 cannot call mutating method (Object.MutatingMethod) on immutable receiver `o` (* const Object)
.example.go:16:21 cannot assign to contextually immutable field Object.mutableField
.example.go:17:34 cannot assign to contextually immutable field Object.mutableField
.example.go:24:9 cannot call mutating method (Object.MutatingMethod) on immutable variable `obj` of type `* const Object`
```

----

### 2.3. Immutable Arguments
Immutable arguments are declared using the `const` qualifier and guaranteed to
not be mutated by the receiving function. Mutating any mutable fields of the
object is illegal within the recursive context of the receiving function.
Calling non-const mutating methods on immutable arguments is illegal as well.

```go
type Object struct {
	MutableField *Object // Mutable
}

// MutatingMethod is a non-const method
func (o *Object) MutatingMethod() {}

// ImmutableMethod is a const method
func (o * const Object) ImmutableMethod() {}

// MutateObject mutates the given object
func MutateObject(obj *Object) {
	obj.MutableField = &Object{}
}

// ReadObj is guaranteed to only read the object passed by the argument
// without mutating it in any way
func ReadObj(
	obj * const Object // Mutable reference to immutable object
) {
	obj = nil // fine, because the pointer is mutable

	MutateObject(obj)            // Compile-time error
	obj.MutatingMethod()         // Compile-time error
	obj.MutableField = &Object{} // Compile-time error
}
```
**Expected compilation errors:**
```
.example.go:23:19 cannot use obj (type * const Object) as type *Object in argument to MutateObject
.example.go:24:9 cannot call mutating method (Object.MutatingMethod) on immutable variable `obj` of type `* const Object`
.example.go:25:23 cannot assign to contextually immutable field (Object.MutableField) of type `*Object`
```

----

### 2.4. Immutable Return Values
Immutable return values are declared using the `const` qualifier and guarantee
that the returned objects will be immutable in any receiving context.

```go
type Object struct {
	MutableField *Object // Mutable
}

// MutatingMethod is a non-const method
func (p *Object) MutatingMethod() {
	p.MutableField = &Object{}
}

// ReturnImmutable returns an immutable value
func ReturnImmutable() const * const Object {
	return const * const Object(&Object{
		MutableField: &Object{
			MutableField: &Object{},
		},
	})
}

func main() {
	immutableVariable := ReturnImmutable()
	
	immutableVariable.MutableField = &Object{} // Compile-time error
	immutableVariable.MutatingMethod()         // Compile-time error
}
```
**Expected compilation errors:**
```
.example.go:22:37 cannot assign to contextually immutable field (Object.MutableField) of immutable variable `immutableVariable` of type `const * const Object`
.example.go:23:23 cannot call mutating function (Object.MutatingMethod) on contextually immutable variable `immutableVariable` of type `const * const Object`
```

----

### 2.5. Immutable Variables
Immutable variables are declared using the `const` qualifier and guarantee
to not be mutated within any context.

```go
type Object struct {
	MutableField *Object // Mutable
}

// MutatingMethod is a non-const method
func (o *Object) MutatingMethod() {}

// ImmutableMethod is a const method
func (o * const Object) ImmutableMethod() {}

// NewObject creates and returns a new mutable object instance
func NewObject() *Object {
	return &Object{}
}

// MutateObject mutates the given object
func MutateObject(obj *Object) {
	obj.MutableField = &Object{}
}

// ConstRef helps shortening declaration statements
type ConstRef const * const Object

func main() {
	// The definition version:
	// The cast is necessary because NewObject returns a mutable value
	// while we want an immutable variable
	obj := const * const Object (NewObject())

	// The var declaration version:
	// (this statement could be shortened using a type alias)
	var obj_var_long const * const Object = const * const Object(NewObject())
	var obj_var ConstRef = ConstRef(NewObject())

	obj.MutableField = &Object{} // Compile-time error
	obj.MutatingMethod()         // Compile-time error
	MutateObject(obj)            // Compile-time error
}
```
**Expected compilation errors:**
```
.example.go:23:23 cannot assign to contextually immutable field (Object.MutableField) of type `*Object`
.example.go:24:9 cannot call mutating method (Object.MutatingMethod) on immutable variable `obj` of type `const * const Object`
.example.go:25:19 cannot use obj (type const * const Object) as type *Object in argument to MutateObject
```

### 2.6. Immutable Interface Methods
Interfaces can be obliged to require receiver type immutability using the
`const` qualifier in the method declaration to prevent the implementing function
from mutating the object referenced by the interface.
```go
// Interface represents a strict interface with an immutable method
type Interface interface {
	// Read must not mutate the underlying implementation
	const Read() string

	// Write can potentially mutate the underlying implementation
	Write(string)
}

// ValidImplementation represents a correct implementation of Interface
type ValidImplementation struct {
	buffer []byte
}

// Read correctly implements Interface.Read, it has an immutable receiver
func (r * const ValidImplementation) Read() string {
	/*...*/
}

// Write correctly implements Interface.Write,
// even though the receiver is immutable
func (r * const ValidImplementation) Write(s string) {
	/*...*/
}

// InvalidImplementation represents an incorrect implementation of Interface
type InvalidImplementation struct {
	buffer []byte
}

// Read incorrectly implements the immutable Interface.Read,
// the receiver must be of type: * const InvalidImplementation
func (r * InvalidImplementation) Read() string {
	/*...*/
}

// Write correctly implements Interface.Write
func (r * InvalidImplementation) Write(s string) {
	/*...*/
}

func main() {
	var iface Interface = &InvalidImplementation // Compile-time error
	iface.Write(0, const([]byte("example")))
}
```
```
.example.go:43:26: cannot use InvalidImplementation literal (type *InvalidImplementation) as type Interface in assignment:
	*InvalidImplementation does not implement Interface (Read method has mutable pointer receiver, expected an immutable receiver type)
```

----

### 2.7. Slice Aliasing
Immutability of slices is always inherited from their parent slice. Sub-slicing
immutable slices results in new immutable slices:
```go
func ReturnConstSlice() const []int {
	return const([]int {1, 2, 3})
}

func main() {
	originalSlice := const []int{1, 2, 3}
	subSlice := originalSlice[:1]

	originalSlice[0] = 4 // Compile-time error
	subSlice[0] = 4      // Compile-time error

	anotherSubSlice := ReturnConstSlice()[:1]
	anotherSubSlice[0] = 4 // Compile-time error
}
```
```
.example.go:9:23 cannot assign to immutable variable of type `const []int`
.example.go:10:18 cannot assign to immutable variable of type `const []int`
.example.go:13:25 cannot assign to immutable variable of type `const []int`
```

### 2.8. Address Operators

#### 2.8.1. Taking the address of a variable
Taking the address of an *immutable* variable results in a *mutable* pointer to
an *immutable* object:

```go
var t const T = const(T{})
t_pointer := &t // * const T
```

To take an *immutable* pointer from an *immutable* variable explicit casting is
needed:
```go
var t const T = const(T{})
t_pointer := const(&t) // const * const T
```

#### 2.8.2. Dereferencing a pointer
Dereferencing an *immutable* pointer to an *immutable* object:
```go
t := const * const T (&T{})
*t // const T
```

Dereferencing a mutable pointer to an *immutable* object:
```go
t := * const T (&T{})
*t // const T
```

Dereferencing an *immutable* pointer to a *mutable* object:
```go
t := const * T (&T{})
*t // T
```

### 2.9. Immutable Reference Types
Reference types such as [slices](https://golang.org/ref/spec#Slice_types),
[maps](https://golang.org/ref/spec#Map_types),
[channels](https://golang.org/ref/spec#Channel_types) and
[pointers](https://golang.org/ref/spec#Pointer_types) can also be declared
immutable using the `const` qualifier just like any other type. But the objects
/ items referenced by immutable reference types **don't inherit their
immutability!** Reference types can point to both mutable and immutable types,
this makes the type system very versatile and flexible.

The examples below demonstrate a few possible combinations:

#### 2.9.1. Pointer Examples

##### 2.9.1.1. Immutable pointer to a mutable object

```go
var immut2mut const *Object = const(&Object{})

immut2mut = &Object{} // Compile-time error
immut2mut.Field = 42  // fine
immut2mut.Mutation()  // fine
```

##### 2.9.1.2. Mutable pointer to an immutable object

```go
var mut2immut * const Object = * const Object(&Object{})

mut2immut = &Object{} // fine
mut2immut.Field = 42  // Compile-time error
mut2immut.Mutation()  // Compile-time error
```

##### 2.9.1.3. Immutable pointer to an immutable object
```go
var immut2immut const * const Object = const * const Object(&Object{})

immut2immut = &Object{} // Compile-time error
immut2immut.Field = 42  // Compile-time error
immut2immut.Mutation()  // Compile-time error
```

#### 2.9.2. Slice Examples

##### 2.9.2.1. Immutable slice of immutable objects

```go
var immut2immut const [] const Object
immut2immut = append(immut2immut, Object{}) // Compile-time error
immut2immut[0] = Object{}                   // Compile-time error

obj := immut2immut[0]
obj.Mutation() // Compile-time error
```

##### 2.9.2.2. Mutable slice of immutable objects

```go
var mut2immut [] const Object
mut2immut = append(mut2immut, Object{}) // fine
mut2immut[0] = Object{}                 // fine

obj := mut2immut[0]
obj.Mutation() // Compile-time error
```

##### 2.9.2.3. Immutable slice of mutable objects

```go
var immut2mut const [] Object
immut2mut = append(immut2mut, Object{}) // Compile-time error
immut2mut[0] = Object{}                 // Compile-time error

obj := immut2mut[0]
obj.Mutation() // fine
```

##### 2.9.2.4. Mutable slice of mutable objects

```go
var mut2mut [] Object
mut2mut = append(mut2mut, Object{}) // fine
mut2mut[0] = Object{}               // fine

obj := mut2mut[0]
obj.Mutation() // fine
```

#### 2.9.3. Map Examples

##### 2.9.3.1. Mutable map of immutable keys to mutable objects

```go
var mut_immut2mut map[const Object] Object

newKey := const(Object{})
mut_immut2mut[newKey] = Object{} // fine
delete(mut_immut2mut, newKey)    // fine

for key, value := range mut_immut2mut {
	key.Mutation()   // Compile-time error
	value.Mutation() // fine
}
```

##### 2.9.3.2. Mutable map of mutable keys to immutable objects

```go
var mut_mut2immut map[Object] const Object

newKey := Object{}
mut_mut2immut[newKey] = const(Object{}) // fine
delete(mut_mut2immut, newKey)           // fine

for key, value := range mut_mut2immut {
	key.Mutation()   // fine
	value.Mutation() // Compile-time error
}
```

##### 2.9.3.3. Mutable map of immutable keys to immutable objects

```go
var immut_immut2immut map[const Object] const Object

newKey := const(Object{})
immut_immut2immut[newKey] = const(Object{}) // fine
delete(immut_immut2immut, newKey)           // fine

for key, value := range immut_immut2immut {
	key.Mutation()   // Compile-time error
	value.Mutation() // Compile-time error
}
```

##### 2.9.3.4. Immutable map of immutable keys to immutable objects

```go
var m const map[const Object] const Object

newKey := const(Object{})
m[newKey] = const(Object{}) // Compile-time error
delete(m, newKey)           // Compile-time error

for key, value := range m {
	key.Mutation()   // Compile-time error
	value.Mutation() // Compile-time error
}
```

#### 2.9.4. Channel Examples

##### 2.9.4.1. Immutable channels of immutable objects

```go
func main() {
	ch := ConstReadOnlyChannel()
	ch = AnotherChannelOfSameType() // Compile-time error

	immutObj := <-ch
	immutObj.Field = 42  // Compile-time error
	immutObj.Mutation()  // Compile-time error
}

// ConstReadOnlyChannel returns an immutable read only channel
// of immutable objects
func ConstReadOnlyChannel() const <-chan const Object {
	ch := make(const chan const Object)
	go func() {
		ch <- const(Object{})
	}()
	return ch
}
```

##### 2.9.4.2. Immutable channels of mutable objects

```go
func main() {
	ch := ConstReadOnlyChannel()
	ch = AnotherChannelOfSameType() // Compile-time error

	mutObj := <-ch
	mutObj.Field = 42  // fine
	mutObj.Mutation()  // fine
}

// ConstReadOnlyChannel returns an immutable read only channel
// of mutable objects
func ConstReadOnlyChannel() const <-chan Object {
	ch := make(const chan Object)
	go func() {
		ch <- Object{}
	}()
	return ch
}
```

##### 2.9.4.3. Mutable channels of immutable objects

```go
func main() {
	ch := MutReadOnlyChannel()
	ch = MutReadOnlyChannel() // fine

	immutObj := <-ch
	immutObj.Field = 42  // Compile-time error
	immutObj.Mutation()  // Compile-time error
}

// MutReadOnlyChannel returns a mutable read only channel of immutable objects
func MutReadOnlyChannel() <-chan const Object {
	ch := make(chan const Object)
	go func() {
		ch <- const(Object{})
	}()
	return ch
}
```

### 2.10. Type Casting
Mutable types need to be explicitly casted to immutables types to be used as
such because implicit casting in Go is only used in exceptional cases like type
to interface conversions.

#### 2.10.1. Simple Casting
Most of the times simple `const` casting is enough like in the examples below:

```go
// ReturnConstString returns an immutable string value
func ReturnConstString() const string {
	var test string

	return "test" // Compile-time error
	return test   // Compile-time error

	// Simple non-const to const casting
	return const("test")
	return const(test)
}
```

```go
// ReturnConstSlice returns an immutable slice
func ReturnConstSlice() const [] int {
	test := make([] int, 3)

	return make([] int, 3)    // Compile-time error
	return [] int {1, 2, 3}   // Compile-time error
	return test               // Compile-time error

	return make(const [] int, 3)

	// Simple non-const to const casting
	return const([] int {1, 2, 3})
	return const(test)
}
```

Simple typecasting `const(o)` inverts the type of the given object `o` to its
immutable counterpart for `o` to be treated as immutable. Applying simple
typecasting to already immutable types has no effect.

#### 2.10.2. Literal Type Casting
There also are more complex situations where simple `const` casting is
insufficient. In those situations, the mutable type needs to be type-casted
`immutable type (symbol)` to an immutable type.

```go
// ReturnConstTPointerMatrix returns an immutable slice of immutable slices of
// pointers to immutable instances of T
func ReturnConstTPointerMatrix() const [] const [] * const T {
    var test [] [] *T

    return make([] [] *T, 3)   // Compile-time error
    return [] [] *T            // Compile-time error
    return test                // Compile-time error
    return const [][]*T (test) // Compile-time error

    // cast the deeply mutable to a deeply immutable type
    return const [] const [] * const T (test)

    return make(const [] const [] * const T, 3)
    return const [] const [] * const T {
        const [] * const T {* const T ( &T{} )},
        const [] * const T {* const T ( &T{} )},
        const [] * const T {* const T ( &T{} )},
    }
}
```

```go
// ReturnMap returns a mutable map of
// immutable T to pointer of immutable T
func ReturnMap() map[const T] * const T {
	var test map[T] *T

	return make(map[T] *T, 3) // Compile-time error
	return map[T] *T {}       // Compile-time error
	return test               // Compile-time error
	return const map[T] *T    // Compile-time error

	return make(map[const T] * const T, 3)
	return map[const T] * const T {nil, nil, nil}

	// const-type-cast
	return map[const T] * const T (test)
}
```

```go
func main() {
	// Declare mutable matrix of pointers to mutable structs
	var mutable [][]*T

	// Cast to immutable matrix of pointers to mutable structs
	immutable_variant1 := const [] const [] *T (mutable)

	// Cast to immutable slice of mutable slices of pointers to mutable structs
	immutable_variant2 := const [] [] *T (mutable)

	// Cast to mutable matrix of pointers to immutable structs
	immutable_variant3 := [][] * const T (mutable)
}
```

#### 2.10.3. Prohibition of Casting Immutable- to Mutable Types
Casting immutable types to mutable types is forbidden because it would make it
possible to silently void the immutability guarantee breaking the entire concept
of immutability.

### 2.11. Implicit Casting

#### 2.11.1. Implicit Casting of Pointer Receivers
Pointer receivers are implicitly casted in both directions (mutable to immutable
and vice-versa) when the types they're pointing to match.

Methods:
```go
type T struct {/*...*/}
func (r1 *T) M1() {/*...*/}
func (r2 const * T) M2() {/*...*/}
func (r3 * const T) M3() {/*...*/}
func (r4 const * const T) M4() {/*...*/}
```

Variables:
```go
// mutable pointer to mutable T
t1 := &T{}

// immutable pointer to mutable T
t2 := const * T(&T{})

// mutable pointer to immutable T
t3 := * const T(&T{})

// immutable pointer to immutable T
t4 := const * const T(&T{})
```

| Combination | Compile-time Result | Reason |
|-|-|-|
| `t1.M1()` | legal | types match. |
| `t2.M1()` | **implicit cast** |  `const * T` (`t2`) is implicitly casted to `* T` (`r1`) because in both cases `T` is mutable. |
| `t3.M1()` | illegal | `T` referenced by `t3` is immutable, but `M1` is a mutating method. |
| `t4.M1()` | illegal | `T` referenced by `t4` is immutable, but `M1` is a mutating method. |
| `t1.M2()` | **implicit cast** | `* T` (`t1`) is implicitly casted to `const * T` (`r2`) because in both cases `T` is mutable. |
| `t2.M2()` | legal | types match. |
| `t3.M2()` | illegal | `T` referenced by `t3` is immutable, but `M2` is a mutating method. |
| `t4.M2()` | illegal | `T` referenced by `t4` is immutable, but `M2` is a mutating method. |
| `t1.M3()` | **implicit cast**  | `* T` (`t1`) is implicitly casted to `* const T` (`r3`) because `T` is mutable and `M3` is a non-mutating method. |
| `t2.M3()` | illegal | `T` referenced by `t2` is mutable, but `M3` is a mutating method. |
| `t3.M3()` | legal | types match. |
| `t4.M3()` | **implicit cast** | `const * const T` (`t4`) is implicitly casted to `* const T` (`r3`) because in both cases `T` is immutable. |
| `t1.M4()` | **implicit cast** | `* T` (`t1`) is implicitly casted to `const * const T` (`r4`) because `T` is mutable and `M3` is a non-mutating method. |
| `t2.M4()` | **implicit cast** | `const * T` (`t2`) is implicitly casted to `const * const T` (`r4`) because `T` is mutable and `M4` is a non-mutating method. |
| `t3.M4()` | **implicit cast** | `* const T` (`t3`) is implicitly casted to `const * const T` (`r4`) because in both cases `T` is immutable. |
| `t4.M4()` | legal | types match. |

## 3. Immutability by Default (Go >= 2.x)
If we were to think of an immutability proposal for the backward-incompatible Go
2 language specification, then making all types immutable by default and
introducing a special keyword `mut` for mutability annotation would be a better
option.

```go
// Object implements the ObjectInterface interface
type Object struct {
	Immutable_str string
	Mutable_str   mut string

	Immutable_immutRef_to_immutObj * Object
	Mutable_mutRef_to_immutObj     mut * Object
	Mutable_mutRef_to_mutObj       mut * mut Object

	Immutable_immutSlice_of_immutObj [] Object
	Mutable_mutSlice_of_immutObj     mut [] Object
	Mutable_mutSlice_of_mutObj       mut [] mut Object

	Immutable_immutMap_of_immutObj map[Object] Object          // immutable key
	Mutable_mutMap_of_immutObj     mut [Object] Object         // immutable key
	Mutable_mutMap_of_mutObj       mut [mut Object] mut Object // mutable key
}

// MutableMethod implements ObjectInterface.MutableMethod
func (mutableReceiver * mut Object) MutableMethod(
	mutableArgument mut * Object, // mutable reference to immutable object
) (
	mutableReturnValue mut * mut Object, // mutable reference to mutable object
) {
	var mutRef_to_mutObj mut * mut Object
	var mutRef_to_immutObj mut * Object
	var immutRef_to_immutObj * Object

	return nil
}

// ImmutableMethod implements ObjectInterface.ImmutableMethod
func (immutableReceiver *Object) ImmutableMethod(
	immutableArgument * Object, // immutable reference to immutable object
) (
	immutableReturnValue * Object // immutable reference to immutable object
) {
	var mutRef_to_mutObj mut * mut Object
	var mutRef_to_immutObj mut * Object
	var immutRef_to_immutObj * Object

	return nil
}

type ObjectInterface interface {
	mut MutableMethod(arg mut * Object) (returnValue mut * mut Object)
	ImmutableMethod(arg *Object) (returnValue *Object)
}
```

### 3.1. Benefits

#### 3.1.1. Safety by Default
It's easy to forget to add the `const` qualifier and accidentally make something
mutable. But when mutable types need to be explicitly declared mutable using the
`mut` qualifier writing code becomes even safer.

#### 3.1.2. No Explicit Casting
When all types are mutable by default then they need to be casted to *immutable*
types before they can be used as such. But when *all* types are **immutable by
default** then no casting is ever necessary.

#### 3.1.3. Less Code
Statistically, Most of the variables, arguments, fields, return values and
methods are immutable, thus the frequent `const` qualifiers can be replaced by
fewer `mut` qualifiers, which improves both readability and coding speed. The
`mut` keyword is also shorter than `const`.

#### 3.1.4. No `const` Keyword Overloading
The need for overloading of the `const` keyword would vanish, which would
improve semantic language consistency.

## 4. FAQ

### 4.1. Are the items within immutable slices/maps also immutable?
**No**, they're not! As stated in [Section 2.9.](#29-immutable-reference-types),
an immutable slice/map of mutable objects is declared this way:
```go
type ImmutableSlice const []*Object
type ImmutableMap   const map[*Object]*Object
```
If you want the items of an immutable slice/map to be immutable as well, you'll
need to declare them using the `const` qualifier:
```go
type ImmutableSlice     const [] const * const Object
type ImmutableMap       const map[*Object] const * const Object
type ImmutableMapAndKey const map[const * const Object] const * const Object
```
A deeply-immutable matrix could, therefore, be declared the following way:
```go
type ImmutableMatrix const [] const [] int
```

---

### 4.2. Go is all about simplicity, so why make the language more complicated?
The `const` qualifier adds only a little cognitive overhead:
- When declaring a **function argument** we have to know whether we want to be
  able to change its state and make it immutable if we don't.
- When declaring a **struct field** we have to know, whether we want the state
  of this field to remain unchangeable, during the lifetime of an object
  instantiated from this struct, as soon as it's initialized.
- When declaring a **return value** we have to know whether we want to give the
  caller the permission to modify the object we returned.
- When declaring a **variable** we have to know, whether we want to change it
  in this context.
- When declaring a **function-receiver** we have to know, whether this function
  will change anything inside the receiver.
- When declaring an **interface method** we have to know, whether this method
  should not change the state of the object implementing this interface.
- When declaring a **reference type** such as a pointer, a slice a map or a
  channel we have to know whether we want to:
	- make the object changeable, but not its reference
	- make the actual reference changeable, but not the object it references
	- make both the reference and the object changeable

This additional cognitive overhead prevents us from introducing the complexity
created by mutable shared state. Bugs introduced through mutable shared state
are very dangerous, hard to notice, hard to identify and pretty hard to fix.
Justifying the simplicity of a language which can lead to very complex bugs is
rather incorrect when considering the insignificant overhead of the `const`
qualifier. Thus, immutability is a feature, the overhead of which outweighs the
disadvantages of not having it.

**Example:** You always have to remember to copy stuff that you don't want
others to be able to mutate, or at least explicitly advise to "not mutate
certain stuff" in the documentation running the risk of breaking your
inattentive colleague's code:

```go
// ConnectedClients returns the list of all currently connected clients.
// DO NOT mutate the returned slice, this could break the server!
func (s *Server) ConnectedClients() []Client {
	return s.clients
}
```
But even if the people working with your code follow your advices, they could
still mess it up:
```go
// ThisWontMutateIt verbally promises to
// not mutate the given slice of clients
func ThisWontMutateIt(clients []Client) {
	// You know what? to hell with the promise!
	// I don't know where this slice originated from, so why care?
	clients[len(clients) - 1] = nil
}

clients = server.ConnectedClients()
// It promised not to mutate it, so it's safe, right? right!?
ThisWontMutateIt(clients)
```

To improve the safety of the code above, we'd usually return a copy instead:
```go
func (s *Server) ConnectedClients() []Client {
	// Copy the client list to avoid returning an unsafe mutable reference
	clients := make([]Client, len(s.clients))
	for i, clt := range s.clients {
		clients[i] = clt
	}
	return clients
}
```
This certainly makes both code and documentation more complicated and error
prone (and slow) than it could be with immutability:
```go
// ConnectedClients returns the list of all currently connected clients.
func (s * const Server) ConnectedClients() const []Client {
	return const(s.clients)
}
```

---

### 4.3. Aren't other features such as generics and better error handling not more important right now?
Unlike other language specification issues such as *"generics"* and *"how to
handle errors more elegantly"* there's really not much to argue about in case of
immutability. It should be clear that:
- it makes code both safer and easier to make sense of,
- it doesn't require any breaking changes,
- it doesn't even require a single new language keyword.

Therefore immutability should be considered of higher priority compared to other
language design proposals.

----

### 4.4. Why overload the `const` keyword instead of introducing a new keyword like `immutable` etc.?
**Backwards-compatibility**. Using the const keyword would allow us to introduce
immutability to Go 1.x without having to make breaking changes to the language.
The introduction of a new keyword could potentially break existing Go 1.x code,
where the new keyword might be used for naming symbols causing build conflicts.
`const` on the other hand is already a [reserved language
keyword](https://golang.org/ref/spec#Keywords) which doesn't interfere with the
proposed language changes and verbally comes close to the desired meaning (for
example, C++ uses the `const` keyword to do just that).

----

### 4.5. How are constants different from immutables?
**Short:** Constants are static in memory, while immutables are just
write-protected references to mutable memory.

**Long:** The value of a constant is defined during the compilation and remains
a static piece of memory for the entire lifetime of your program. An immutable
field, argument, return value, receiver or variable, on the other hand, is
**not** static in memory, because it can still be mutated through mutable
references:

```go
// CreateList creates a new slice and returns both, a mutable and an immutable
// reference to it (which is bad! don't ever do that!
// unless you know what you're after!)
func CreateList(size int) (mutable []string, immutable const []string) {
	newSlice := make([]string, size)
	for i := 0; i < size; i++ {
		newSlice[i] = "sample text"
	}
	return newSlice, const(newSlice)
}

func main() {
	mutableReference, immutableReference := CreateList(10)

	// Mutating an immutable return value is illegal
	immutableReference[5] = "mutated" // Compile-time error

	// Mutating the underlying array through a mutable reference is just fine!
	mutableReference[5] = "mutated"

	// You can now observe the mutation from the read-only immutable reference
	immutableReference[5] // "mutated"
}
```

**NOTICE:** the above code is **bad code**! Its purpose was to demonstrate that
immutables are not constants. If you want to prevent immutable objects from
being mutated for sure - drop all mutable references to it as soon as it's
created!

----

### 4.6. Why do we need immutable receivers if we already have copy-receivers?
There are two reasons: safety and performance.

**Copy-receivers don't prevent mutations!** They simply can't because of
[pointer aliasing](https://en.wikipedia.org/wiki/Pointer_aliasing):

```go
type Object struct {
	name     string
	parent   *Object
	children []*Object
}

// SetName is a non-const mutating method
func (o *Object) SetName(newName string) {
	o.name = newName
}

// ReadOnlyMethod is insidious because it verbally "promises"
// to not touch the object while it still can do!
// Even though it has a non-pointer receiver, the copy
// can't get rid of aliasing and can thus mutate internals!
func (o Object) ReadOnlyMethod() {
	if len(o.children) > 0 {
		// Mutating contextually immutable aliases is legal, VERY BAD!
		o.children[0] = nil
	}
	if o.parent != nil {
		// Call non-const method in immutable context is legal, VERY BAD!
		o.parent.SetName("GOTCHA!")
	}
}

func main() {
	obj := &Object{
		name:   "root",
		parent: nil,
	}
	obj.children = []*Object{
		&Object{
			name:   "child_1",
			parent: obj,
		},
		&Object{
			name:   "child_2",
			parent: obj,
		},
	}

	fmt.Println("Root before:  ", obj)
	fmt.Println("Child before: ", obj.children[0])

	firstChild := obj.children[0]
	firstChild.ReadOnlyMethod() // Will mutate parent's name

	obj.ReadOnlyMethod() // Will mutate children list

	fmt.Println("Root after:   ", obj)        // Children list mutated
	fmt.Println("Child after:  ", firstChild) // Root name mutated
}
```
https://play.golang.org/p/0kRSuVFkSMN

**Copy-receivers are slow(er)**. Consider the following benchmark:
```go
type Object struct {
	name    string
	text    []rune
	double  float64
	integer int64
	bytes   []byte
}

// PointerReceiver has a pointer receiver
func (o *Object) PointerReceiver() (string, []rune, float64) {
	return o.name, o.text, o.double
}

// CopyReceiver has a copy receiver
func (o Object) CopyReceiver() (string, []rune, float64) {
	return o.name, o.text, o.double
}

var name string
var text []rune
var double float64

// BenchmarkPointerReceiver benchmarks the pointer-receiver method
func BenchmarkPointerReceiver(b *testing.B) {
	obj := &Object{}
	b.ResetTimer()
	for n := 0; n < b.N; n++ {
		name, text, double = obj.PointerReceiver()
	}
}

// BenchmarkCopyReceiver benchmarks the copy-receiver method
func BenchmarkCopyReceiver(b *testing.B) {
	obj := &Object{}
	b.ResetTimer()
	for n := 0; n < b.N; n++ {
		name, text, double = obj.CopyReceiver()
	}
}
```
https://play.golang.org/p/2xgn7YMosXO

The results should be similar to:
```
goos: windows
goarch: amd64
pkg: benchreceiver
BenchmarkPointerReceiver-12     1000000000               2.12 ns/op
BenchmarkCopyReceiver-12        300000000                4.23 ns/op
PASS
ok      benchreceiver   4.110s
```
_Windows 10; i7 3930K @ 3.80 Ghz_
```
goos: darwin
goarch: amd64
pkg: github.com/romshark/benchreceiver
BenchmarkPointerReceiver-8    1000000000          2.05 ns/op
BenchmarkCopyReceiver-8       300000000           4.62 ns/op
PASS
ok      benchreceiver   4.129s
```
_MacOS High Sierra (10.13); i7-4850HQ @ 2.30 GHz_

Even though ~2 nanoseconds doesn't sound like much it's still twice as
expensive to call.

**Conclusion:** copy-receivers are not a solution, they make your code slower
**without** providing any protection from [pointer
aliasing](https://en.wikipedia.org/wiki/Pointer_aliasing), thus immutable
receivers (be it an immutable copy or an immutable pointer receiver) are
necessary to ensure compiler-enforced safety.

### 4.7. Why do we need immutable interface types?
Immutable interface types allow us to **reuse** interface types disabling their
mutating ability in certain scopes without having to define two separate
interface types (one interface type with read methods only and another one with
both mutating and non-mutating methods)

Passing an immutable interface to a function as an argument while trying to call
a mutating method on it, for example, would generate a compile-time error:
```go
type Interface {
	const Read()
	Write()
}

type Implementation struct {}
func (i * const Implementation) ReadOnly() {}
func (i * Implementation) Write() {}

// ReadInterface will not be able to execute non-const methods of the interface
func ReadInterface(iface const Interface) {
	iface.ReadOnly() // fine
	iface.Write()    // Compile-time error
}

func main() {
	iface := &Implementation{}
	ReadInterface(const(iface))
}
```
```
.example:13:11: cannot call mutating method on immutable interface iface (const Interface)
```

### 4.8. Doesn't the `const` qualifier add boilerplate and make code harder to read?
**Short answer**: No, it doesn't and it can be quite the opposite.

**Long answer**: Let's pretend we need to write a method with the following
constraints:
- It must take a slice of pointers to objects of type `Object` as argument `s`.
- It must return all objects from an internal slice.
- It must use the function `Dependency` that's exported from a third-party
  package `thirdparty` and pass `s` to it.
- The `thirdparty.Dependency` function doesn't specify whether or not it'll
  mutate `s` in the documentation.
- It **must not mutate** `s`, neither the slice nor the referenced objects!
- It must ensure the internal slice **cannot be mutated** from the outside!
- It must ensure, that the receiver **is not mutated** in any way!

Our current approach would be copying because there's no other way to ensure
immutability.
```go
/* WITHOUT IMMUTABILITY */

func (rec *T) OurMethod(s []*Object) [] *Object {
	s_copy := make([] *Object, len(s))
	for i, item := range s {
		// Clone the items to get rid of aliasing
		// Copying an aliased slice wouldn't make any sense otherwise
		s_copy[i] = item.Clone()
	}

	// Pass a copy of "s" to third-party function to ensure it doesn't modify it
	thirdparty.Dependency(s_copy)

	// Now return a deep copy of the internal slice to prevent any mutations
	internal_copy := make([] *Object, len(rec.internal))
	for i, item := range rec.internal {
		// Clone to avoid aliasing
		// Copying an aliased slice wouldn't make any sense otherwise
		internal_copy[i] = item.Clone()
	}
	return internal_copy
}
```

Now feel free to remove the comments and compare the above copy-bloated code
with the code protected by the `const` qualifier:

```go
/* WITH IMMUTABILITY */

func (rec * const T) OurMethod(
    s const [] * const Object,
) const [] * const Object {
  thirdparty.Dependency(s)                     // s is safe
  return const [] * const Object(rec.internal) // rec.internal is safe
}
```

If you don't like the rather verbose type definitions then consider using type
aliasing to shorten the code even more:

```go
type ConstSlice const [] * const Object

func (rec * const T) OurMethod(s ConstSlice) ConstSlice {
  thirdparty.Dependency(s)
  return rec.internal
}
```

### 4.9. Why do we need the distinction between immutable and mutable reference types?
Simply put, the question is: *why do we have to write out the rather verbose
`const * const Object` and `const [] const Object` instead of just
`const *Object` and `const []Object` respectively?*

There are certain situations where mutable references to immutable types are
necessary such as when we want to describe a dynamic, interlinked graph data
structure where the nodes of the graph are immutable.

```go
// GraphNode represents a node with outbound and inbound connections.
// Connections can be changed, but the underlying nodes will remain immutable
type GraphNode struct {
	inbound * const GraphNode
	outbound * const GraphNode
}
```

Contrary, there are other situations where we'd require immutable references to
mutable objects, such as in the case of rather complex functions taking
references to mutable graph nodes:

```go
// MutateGraphNode takes an immutable pointer to a mutable graph node,
// The pointer needs to be immutable so that it behaves like an
// immutable variable so we can't accidentally change it in the scope
// of the function potentially messing up the whole calculation!
func MutateGraphNode(ref const * GraphNode) {
	/*...*/
	ref = &GraphNode{} // Compile-time error
	/*...*/
	ref.outbound = &GraphNode{} // fine
	ref.Mutation()              // fine
}
```

Without this distinction, the above code wouldn't be possible and we'd have to
compromise compile-time safety by **removing immutability** to solve similar
problems. Reference types like pointers, slices, and maps are just regular types
and should be treated as such consistently without any special regulations.

### 4.10. Why not implicitly convert mutable to immutable types?
In Go, types are never implicitly converted and always need to be explicitly
casted. As the immutability operator `const` is just a type qualifier - making
mutable types implicitly convert to immutable types would lead to inconsistency
in the language specification.

## 5. Other Proposals
### 5.1. [proposal: spec: add read-only slices and maps as function arguments #20443](https://github.com/golang/go/issues/20443)
The proposed kind of immutability described in the document above doesn't solve
the mutable shared state problem caused by [pointer
aliasing](https://en.wikipedia.org/wiki/Pointer_aliasing) at all proposing only
exceptional treatment of slices and maps passed as function arguments.

#### 5.1.1. Disadvantages
- **Inconsistent:** it introduces exceptional rules for map- and slice-type
  arguments leading to eventual specification inconsistency.
- **Doesn't solve the root problem:** it doesn't take mutable pointer types into
  account which are prone to [pointer
  aliasing](https://en.wikipedia.org/wiki/Pointer_aliasing) leading to mutable
  shared state just like slices and maps.
- **Very limited:** it doesn't propose immutability for variables, methods,
  fields, return values and arguments of any other type than slices and maps.
- **Leads to performance degradation:** it proposes shallow-copying of slices
  and maps passed to function argument instead of actual compile-time
  immutability errors (even though it mentions it).
- **Unclear:** it doesn't clearly define how to handle slice aliasing.
- **Unclear:** it also doesn't clearly define how to handle nested container
  types.
- **Very limited:** it won't allow different combinations of mutable and
  immutable types (such as passing mutable references inside immutable slices
  and similar).

#### 5.1.2. Similarities
- Similar `const` keyword overloading with the same argumentation with slight
  differences (used as argument field qualifier rather than as argument type
  qualifier).

----

### 5.2. [proposal: Go 2: read-only types #22876](https://github.com/golang/go/issues/22876)
The proposed `ro` qualifier described in the document above is similar to
current proposal but still has some significant differences.

#### 5.2.1. Disadvantages
- **Backward-incompatible:** the proposed feature requires
  backward-incompatible language changes.
- **Limiting and partially pointless:** _`ro` Transitivity_ describes the
  inheritance of immutability by types referenced by immutable references, which
  limits the ability of the developer to describe *immutable references to
  mutable objects* and similar. This limitation will make developers avoid using
  `ro` types alltogether and use unsafe mutable types instead when a mix between
  mutable and immutable references is necessary. An immutable slice of mutable
  slices: `const [] [] int` wouldn't be possible with `ro` transitivity and
  would leave the developer no choice but turning it into a mutable slice of
  mutable slices: `[][]int` making the entire concept of read-only types
  partially pointless.
- **Less advanced:** it doesn't propose immutable struct fields.

#### 5.2.2. Differences
- "Immutability" is called "read-only type permissions" while constants are
  called "immutables".
- Proposes implicit *mutable to immutable* type conversion by default.

#### 5.2.3. Similarities
- The proposed `ro` qualifier is part of the type definition just as the `const`
  qualifier.
- Proposes immutable return values, arguments, interfaces and receivers.
- Describes very similar benefits.

----
Copyright © 2018 [Roman Sharkov](https://github.com/romshark)
(<roman.sharkov@qbeon.com>)
