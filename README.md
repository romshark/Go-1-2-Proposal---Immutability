![Immutability - A Go Language Feature Proposal](https://github.com/romshark/Go-2-Proposal---Immutability/blob/master/document_header.png "Immutability - A Go Language Feature Proposal")

# Go - Immutability
This document describes a language feature proposal to immutability for the [Go
programming language](https://golang.org). The proposed feature targets the
current [Go 1.x (> 1.11) language specification](https://golang.org/ref/spec)
and doesn't violate [the Go 1 compatibility
promise](https://golang.org/doc/go1compat). It also describes [an even better
approach to immutability](#3-immutability-by-default-go--2x) for a hypothetical, backwards-incompatible [Go 2
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
			- [1.1.2. Vague Documentation](#112-vague-documentation)
			- [1.1.3. The "Slow but Safe vs Dangerous but Fast" Dilemma](#113-the-%22slow-but-safe-vs-dangerous-but-fast%22-dilemma)
		- [1.2. Benefits](#12-benefits)
			- [1.2.1. Safe Code](#121-safe-code)
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
			- [Taking the address of a variable](#taking-the-address-of-a-variable)
			- [Dereferencing a pointer](#dereferencing-a-pointer)
	- [3. Immutability by Default (Go >= 2.x)](#3-immutability-by-default-go--2x)
		- [3.1. Benefits](#31-benefits)
			- [3.1.1. Safety by Default](#311-safety-by-default)
			- [3.1.2. No Casting](#312-no-casting)
	- [4. FAQ](#4-faq)
		- [4.1. Are the items within immutable slices/maps also immutable?](#41-are-the-items-within-immutable-slicesmaps-also-immutable)
		- [4.2. Go is all about simplicity, so why make the language more complicated?](#42-go-is-all-about-simplicity-so-why-make-the-language-more-complicated)
		- [4.3. Aren't other features such as generics and better error handling not more important right now?](#43-arent-other-features-such-as-generics-and-better-error-handling-not-more-important-right-now)
		- [4.4. Why overload the `const` keyword instead of introducing a new keyword like `immutable` etc.?](#44-why-overload-the-const-keyword-instead-of-introducing-a-new-keyword-like-immutable-etc)
		- [4.5. How are constants different from immutables?](#45-how-are-constants-different-from-immutables)
		- [4.6. Why do we need immutable receivers if we already have copy-receivers?](#46-why-do-we-need-immutable-receivers-if-we-already-have-copy-receivers)
		- [4.7. Why do we need immutable interfaces?](#47-why-do-we-need-immutable-interfaces)
		- [4.8. What's the difference between *immutable references* and *references to immutables*?](#48-whats-the-difference-between-immutable-references-and-references-to-immutables)
			- [Immutable pointer to mutable object](#immutable-pointer-to-mutable-object)
			- [Mutable pointer to immutable object](#mutable-pointer-to-immutable-object)
			- [Immutable pointer to immutable object](#immutable-pointer-to-immutable-object)
			- [Immutable slice of immutable objects](#immutable-slice-of-immutable-objects)
			- [Mutable slice of immutable objects](#mutable-slice-of-immutable-objects)
			- [Immutable slice of mutable objects](#immutable-slice-of-mutable-objects)
			- [Mutable slice of mutable objects](#mutable-slice-of-mutable-objects)
			- [Mutable map of immutable keys to mutable objects](#mutable-map-of-immutable-keys-to-mutable-objects)
			- [Mutable map of mutable keys to immutable objects](#mutable-map-of-mutable-keys-to-immutable-objects)
			- [Mutable map of immutable keys to immutable objects](#mutable-map-of-immutable-keys-to-immutable-objects)
			- [Immutable map of immutable keys to immutable objects](#immutable-map-of-immutable-keys-to-immutable-objects)
		- [4.9. Doesn't the `const` qualifier add boilerplate and make code harder to read?](#49-doesnt-the-const-qualifier-add-boilerplate-and-make-code-harder-to-read)
		- [4.10. Why do we need the distinction between immutable and mutable reference types?](#410-why-do-we-need-the-distinction-between-immutable-and-mutable-reference-types)
	- [5. Other Proposals](#5-other-proposals)
		- [5.1. proposal: spec: add read-only slices and maps as function arguments #20443](#51-proposal-spec-add-read-only-slices-and-maps-as-function-arguments-20443)
			- [Disadvantages](#disadvantages)
			- [Similarities](#similarities)
		- [5.2. proposal: Go 2: read-only types #22876](#52-proposal-go-2-read-only-types-22876)
			- [Disadvantages](#disadvantages)
			- [Differences](#differences)
			- [Similarities](#similarities)

## 1. Introduction
Immutability is a technique to prevent undesired mutations by annotating
immutable symbols like variables, arguments, fields, return values and methods
for both security and performance reasons. A Go 1.11 developer's current
approach to immutability is manual copying because it doesn't currently provide
any compiler-enforced immutability annotations. This makes Go 1.x harder to work
with because immutability isn't rigidly guaranteed by the compiler, which can
lead to hard to find bugs.

Ideally a programming language should enforce immutability by default while the
developers must explicitly annotate mutable symbols as such, but this concept
would require significant, backwards-incompatible language changes breaking
existing Go 1.x code. To prevent breaking Go 1.x compatibility this document
describe an approach to immutability by overloading the `const` keyword ([see
here for more
details](#34-why-overload-the-const-keyword-instead-of-introducing-a-new-keyword-like-immutable-etc))
to act as an immutable type qualifier similar to languages like C++. Symbols
annotated as immutables are checked for mutations during the compilation phase
resulting in compile-time errors. There is no runtime cost to this approach but
a slight compile-time cost is still required.

### 1.1. Current Problems

The current approach to immutability has a number of disadvantages listed below
and sorted by importance in descending order.

#### 1.1.1. Ambiguous Code and Dangerous Bugs
The absence of immutability annotations can lead to ambiguous code that results
in dangerous, hard to find bugs.

Imagine the following situation: you add a new 3-rd party package `xyz` to your
project as an external dependency. This package provides an exported function
`xyz.ReadSlice(slice []string)` the documentation of which promises that the
provided slice of strings will not be mutated, so you write your code with this
assumption in mind.

At any time though the vendor of the external package might change its behavior
either intentionally (updating the documentation) or unintentionally (the slice
might get aliased and thus unintentionally mutate the original slice somewhere
in the depths of the package or it's external dependencies). Chances are high
that either you miss the changed documentation or the behavior of the code
changes without acknowledging you at all in the worst case! This can obviously
lead to serious and hard to find bugs.

#### 1.1.2. Vague Documentation
We have to manually document what can be mutated and what the code user **must
not** mutate. Not only does this unnecessarily complicate the documentation, it
also makes it error prone and redundant.

#### 1.1.3. The "Slow but Safe vs Dangerous but Fast" Dilemma
As previously mentioned, copies are the only way to achieve immutability in Go
1.x, but copies degrade runtime performance. This dilemma encourage us to write
unsafe mutable APIs when targeting optimal runtime performance. We have to
choose between performance and safety even though having both would be possible
with compiler-enforced immutability at the cost of a slightly decreased
compilation time.

### 1.2. Benefits

The addition of immutability support would provide the benefits listed below
and sorted by importance in descending order.

#### 1.2.1. Safe Code
With immutability annotations the situation described in the [previous
section](#113-ambiguous-code-and-dangerous-bugs) wouldn't even be possible,
because the author of the function of the external package would need to
explicitly denote the argument as immutable to make the compiler enforce the
guarantee, while the user of the function would make decisions based on the
actual function declaration instead of relying on the potentially inconsistent
documentation.

This way - when ever you see a mutable reference type argument you'll know you
have to assume that the state of the referenced object will potentially be
mutated. Contrary you can safely assume that the referenced object won't be
mutated if it's declared immutable.

If the vendor of the external function decides to change the mutability of an
argument he/she will have to change the function declaration introducing
breaking API changes causing you to pay attention to whether or not everything's
right. The vendor won't be able to just silently mutate your objects referenced
by immutable references! The compiler will prevent this either before the vendor
releases the update (assuming that the code is compiled before publication by a
CI system) or during your local build (in the worst case) preventing insidious
bugs from being introduces.

#### 1.2.2. Self-Explaining Code
With immutability annotations there's no need to explicitly describe mutability
recommendations in the documentation. When immutable types are declared as such
then the code becomes self-explaining:
- If you see an immutable argument - you can rely on it not being changed
  neither inside the function itself, nor inside any other functions this
  function calls etc.
- If you see an immutable method (or a "function with an immutable receiver" if
  you will) - you can rely on it not changing the object.
- If you return an immutable return value - you can rely on it not being
  mutated by the function caller.
- If you see an immutable field - you can rely on it not being changed as soon
  as the object is initialized, even inside its origin package.
- If you see an immutable variable - you can rely on it not being changed in the
  context it's in as well as the contexts it's passed over to.

#### 1.2.3. Increased Runtime Performance
Immutability provides a way to safely avoid unnecessary copying. The compiler
can also make optimizations based on the immutability of certain objects known
at compile time.

## 2. Proposed Language Changes
The following 5 changes need to be made to the language to fully fulfill this
proposal. The changes involve:
- [struct field declaration statements](#21-immutable-fields)
- [function receiver declaration statements](#22-immutable-methods)
- [function arguments declaration statements](#23-immutable-arguments)
- [function return values declaration statements](#24-immutable-return-values)
- [variable declaration statements](#25-immutable-variables)

All proposed changes are fully backwards-compatible and don't require any
breaking changes to be introduced to the language. Code written in previous
versions of the Go 1.x programming language will continue to compile and work as
expected.

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
Immutable methods are declared using the `const` qualifier on the function
receiver and guarantee to not mutate the receiver in any way when called.
They can safely be used in immutable contexts, such as within other
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
    o.MutatingMethod()         // Compile-time error
    o.mutableField = &Object{} // Compile-time error
    return o.mutableField
}

func main() {
    const obj := Object{}
    obj.ImmutableMethod()
    obj.MutatingMethod() // Compile-time error
}
```
**Expected compilation errors:**
```
.example.go:15:7 cannot call mutating method `Object.MutatingMethod` on immutable receiver `o` of type `const * const Object`
.example.go:16:21 cannot assign to contextually immutable field `Object.mutableField` of type `*Object`
.example.go:23:9 cannot call mutating method `Object.MutatingMethod` on contextually immutable variable `obj` of type `const * const Object`
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
.example.go:24:9 cannot call mutating method `Object.MutatingMethod` on immutable variable `obj` of type `* const Object`
.example.go:25:23 cannot assign to contextually immutable field `Object.MutableField` of type `*Object`
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
	return &Object{MutableField: &Object{
		MutableField: &Object{},
	}}
}

func main() {
	immutableVariable := ReturnImmutable()
	
	immutableVariable.MutableField = &Object{} // Compile-time error
	immutableVariable.MutatingMethod()         // Compile-time error
}
```
**Expected compilation errors:**
```
.example.go:20:37 cannot assign to field `Object.MutableField` of contextually immutable variable `immutableVariable` of type `const * const Object`
.example.go:21:23 cannot call function `Object.MutatingMethod` with non-const receiver on contextually immutable variable `immutableVariable` of type `const * const Object`
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
	var obj_var_long const * const Object = const * const Object (NewObject())
	var obj_var ConstRef = ConstRef(NewObject())

	obj.MutableField = &Object{} // Compile-time error
	obj.MutatingMethod()         // Compile-time error
	MutateObject(obj)            // Compile-time error
}
```
**Expected compilation errors:**
```
.example.go:23:23 cannot assign to contextually immutable field `Object.MutableField` of type `*Object`
.example.go:24:9 cannot call mutating method `Object.MutatingMethod` on immutable variable `obj` of type `const * const Object`
.example.go:25:19 cannot use obj (type const * const Object) as type *Object in argument to MutateObject
```

### 2.6. Immutable Interface Methods
Immutable methods of an interface are declared using the `const` qualifier and
guarantee that const-methods of the object implementing the interface will not
mutate the underlying object.
```go
type Interface interface {
	// Read must not mutate the underlying implementation
	const Read(offset, length const int) const []byte

	// Write can mutate the underlying implementation
	Write(offset const int, data const []byte) int
}

// ValidImplementation represents an correct implementation
// of the Interface interface
type ValidImplementation struct {
	buffer []byte
}

// Read implements the const method, it must have an immutable receiver
func (ci * const ValidImplementation) Read(
	offset, length const int,
) const []byte {
	return const(ci.buffer[offset : offset+length])
}

// Write implements the non-const method,
// it can either have a mutable or an immutable receiver
func (ci *ValidImplementation) Write(
	offset const int, data const []byte,
) int {
	itr := 0
	for ; itr < len(ci.buffer) && itr < len(data); itr++ {
		ci.buffer[itr + offset] = data[itr]
	}
	return itr + 1
}

// InvalidImplementation represents an incorrect implementation
// of the Interface interface
type InvalidImplementation struct {
	buffer []byte
}

// Read tries to implement the const method using a mutable receiver
func (ci *InvalidImplementation) Read(
	offset, length const int,
) const []byte {
	/*...*/
}

func (ci *InvalidImplementation) Write(
	offset const int, data const []byte,
) int {
	/*...*/
	return 0
}

func main() {
	var iface Interface = &InvalidImplementation
	iface.Write(0, const([]byte("example")))
}
```
```
.example.go:55:26: cannot use InvalidImplementation literal (type *InvalidImplementation) as type Interface in assignment:
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

#### Taking the address of a variable
Taking the address of an *immutable* variable results in a *mutable* pointer to
an *immutable* object:

```go
var t const T = T{}
t_pointer := &t // * const T
```

To take an *immutable* pointer from an *immutable* variable explicit casting is
needed:
```go
var t const T = T{}
t_pointer := const(&t) // const * const T
```

#### Dereferencing a pointer
Dereferencing an *immutable* pointer to an *immutable* object:
```go
t := const * const T (&T{})
*t // const T
```

Dereferencing a mutable pointer to an *immutable* object:
```go
t := * const T = (&T{})
*t // const T
```

Dereferencing an *immutable* pointer to a *mutable* object:
```go
t := const * T = &T{}
*t // T
```

## 3. Immutability by Default (Go >= 2.x)
If we were to think of an immutability proposal for the backward-incompatible Go
2 language specification, then making all types immutable by default and
introducing a special keyword `mut` for mutability annotation would be a better
options.

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
Immutability stands for compile-time safety, which would then be default
behavior. The developer will have to explicitly annotate mutable types using the
`mut` modifier preventing types from accidentally being declared mutable by
forgetting to prepend the `const` qualifier.

#### 3.1.2. No Casting
When all types are mutable by default then they need to be casted to *immutable*
types before they can be used as such in case of explicit casting (which is not
the case for implicit `non-const` to `const` casting though). But when all types
are immutable by default then no casting is ever necessary.

## 4. FAQ

### 4.1. Are the items within immutable slices/maps also immutable?
**No**, they're not! An immutable slice/map of mutable objects is declared this way:
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
A deeply-immutable matrix could therefore be declared the following way:
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
- When declaring a **function receiver** we have to know, whether this function
  will change anything inside the receiver.
- When declaring an **interface method** we have to know, whether this method
  should not change the state of the object implementing this interface.
- When declaring a **reference type** such as a pointer, a slice or a map we
  have to know whether we want to:
	- make the object changeable, but not the its reference
	- make the actual reference changeable, but not the object it references
	- make both the reference and the object changeable

This additional cognitive overhead, prevents us from introducing the complexity
created by mutable shared state. Bugs introduces through mutable shared state
are very dangerous, hard to notice, hard to identify and pretty hard to fix.
Justifying the simplicity of a language which can lead to very complex bugs is
rather incorrect when considering the insignificant overhead of the `const`
qualifier. Thus immutability is a feature the overhead of which outweighs the
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
	return s.clients
}
```

---

### 4.3. Aren't other features such as generics and better error handling not more important right now?
**No**, absolutely not! Unlike with topics such as *"generics"* and *"how to
handle errors more elegantly"* there's really not much to argue about in case of
immutability. It should be clear that it makes code both safer and easier to
make sense of. It doesn't require any breaking changes and it doesn't
even require a single new language keyword. Therefore immutability should be
considered of higher priority compared to other previously mentioned topics.

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
field, argument, return value, receiver or variable on the other hand is **not**
static in memory, because it can still be mutated through mutable references:

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
There's two reasons: safety and performance.

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

### 4.7. Why do we need immutable interfaces?
Immutable interfaces guarantee that you can't call mutating methods of the
underlying implementation through it. Passing an immutable interface to a
function as an argument while trying to call a non-const method on it, for
example, would generate a compile-time error:
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
.example:13:11: cannot call method mutating method on immutable interface iface of type `const Interface`
```

----

### 4.8. What's the difference between *immutable references* and *references to immutables*?
Reference-types such as pointers, slices and maps can be immutable while still
referencing mutable objects. This is useful in many cases such as:
- When we need a slice to be mutable so we can add new items to it, remove items
  from it and replace/swap items inside it, but the actual items should remain
  immutable: `var mutableSlice [] const T`
- When we need a pointer to be immutable to prevent it from pointing to anything
  other than a certain object, which should be mutable though:
  `var immutablePointer const * T`
- When we need only the keys of a mutable map to be immutable so we can add /
  remove / replace and mutate values but not mutate the keys:
  `var mutableMap map [const T] T`
- When we need any other possible combination of mutable and immutable types...

#### Immutable pointer to mutable object

```go
var immut2mut const *Object = &Object{}

immut2mut = &Object{} // violation!
immut2mut.Field = 42  // fine
immut2mut.Mutation()  // fine
```

#### Mutable pointer to immutable object

```go
var mut2immut * const Object = &Object{}

mut2immut = &Object{} // fine
mut2immut.Field = 42  // violation!
mut2immut.Mutation()  // violation!
```

#### Immutable pointer to immutable object
```go
var immut2immut const * const Object = const(&Object{})

immut2immut = &Object{} // violation!
immut2immut.Field = 42  // violation!
immut2immut.Mutation()  // violation!
```

#### Immutable slice of immutable objects

```go
var immut2immut const [] const Object
immut2immut = append(immut2immut, Object{}) // violation!
immut2immut[0] = Object{}                   // violation!

obj := immut2immut[0]
obj.Mutation() // violation!
```

#### Mutable slice of immutable objects

```go
var mut2immut [] const Object
mut2immut = append(mut2immut, Object{}) // fine
mut2immut[0] = Object{}                 // fine

obj := mut2immut[0]
obj.Mutation() // violation!
```

#### Immutable slice of mutable objects

```go
var immut2mut const [] Object
immut2mut = append(immut2mut, Object{}) // violation!
immut2mut[0] = Object{}                 // violation!

obj := immut2mut[0]
obj.Mutation() // fine
```

#### Mutable slice of mutable objects

```go
var mut2mut [] Object
mut2mut = append(mut2mut, Object{}) // fine
mut2mut[0] = Object{}               // fine

obj := mut2mut[0]
obj.Mutation() // fine
```

#### Mutable map of immutable keys to mutable objects

```go
var mut_immut2mut map[const Object] Object

newKey := const(Object{})
mut_immut2mut[newKey] = Object{} // fine
delete(mut_immut2mut, newKey)    // fine

for key, value := range mut_immut2mut {
	key.Mutation()   // violation!
	value.Mutation() // fine
}
```

#### Mutable map of mutable keys to immutable objects

```go
var mut_mut2immut map[Object] const Object

newKey := Object{}
mut_mut2immut[newKey] = const(Object{}) // fine
delete(mut_mut2immut, newKey)           // fine

for key, value := range mut_mut2immut {
	key.Mutation()   // fine
	value.Mutation() // violation!
}
```

#### Mutable map of immutable keys to immutable objects

```go
var immut_immut2immut map[const Object] const Object

newKey := const(Object{})
immut_immut2immut[newKey] = const(Object{}) // fine
delete(immut_immut2immut, newKey)           // fine

for key, value := range immut_immut2immut {
	key.Mutation()   // violation!
	value.Mutation() // violation!
}
```

#### Immutable map of immutable keys to immutable objects

```go
var m const map[const Object] const Object

newKey := const(Object{})
m[newKey] = const(Object{}) // violation!
delete(m, newKey)           // violation!

for key, value := range m {
	key.Mutation()   // violation!
	value.Mutation() // violation!
}
```

### 4.9. Doesn't the `const` qualifier add boilerplate and make code harder to read?
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

Our current approach would be copying, because there's no other way to ensure
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
  thirdparty.Dependency(s)   // safe
  return const(rec.internal) // safe
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

### 4.10. Why do we need the distinction between immutable and mutable reference types?
Simply put, the question is: *why do we have to write out the rather verbose
`const * const Object` and `const [] const Object` instead of just
`const *Object` and `const []Object` respectively?*

There are certain situation where mutable references to immutable types are
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
problems. Reference types like pointers, slices and maps are just regular types
and should be treated as such consistently without any special regulations.

## 5. Other Proposals
### 5.1. [proposal: spec: add read-only slices and maps as function arguments #20443](https://github.com/golang/go/issues/20443)
The proposed kind of immutability described in the document above doesn't solve
the mutable shared state problem cause by [pointer
aliasing](https://en.wikipedia.org/wiki/Pointer_aliasing) at all proposing only
exceptional treatment of slices and maps passed as function arguments.

#### Disadvantages
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

#### Similarities
- Similar `const` keyword overloading with the same argumentation with slight
  differences (used as argument field qualifier rather than as argument type
  qualifier).

----

### 5.2. [proposal: Go 2: read-only types #22876](https://github.com/golang/go/issues/22876)
The proposed `ro` qualifier described in the document above is similar to
current proposal but still has some significant differences.

#### Disadvantages
- **Backwards-incompatible:** the proposed feature requires
  backwards-incompatible language changes.
- **Less flexible:** _`ro` Transitivity_ describes the inheritance of
  immutability by types referenced from immutable references which limits the
  ability of the developer to describe *immutable references to mutable objects*
  (like a mutable slice of immutable slices `[] const [] int` etc.) and similar.
- **Less advanced:** it doesn't propose immutable struct fields.

#### Differences
- "Immutability" is called "read-only type permissions" while constants are
  called "immutables".
- Proposes implicit *mutable to immutable* type conversion by default.

#### Similarities
- The proposed `ro` qualifier is part of the type definition just as the `const`
  qualifier.
- Proposes immutable return values, arguments, interfaces and receivers.
- Describes very similar benefits.

----
Copyright © 2018 [Roman Sharkov](https://github.com/romshark)
(<roman.sharkov@qbeon.com>)
