# Go 2 - Immutability
This document describes a Go 2 (*Go > 1.11*) language feature proposal to
immutability.

Author: [Roman Sharkov](https://github.com/romshark) (<roman.sharkov@qbeon.com>)

**Table of Contents**
- [Go 2 - Immutability](#go-2---immutability)
	- [1. Introduction](#1-introduction)
		- [1.1. Current Problems](#11-current-problems)
			- [1.1.1. Slow Code](#111-slow-code)
			- [1.1.2. Vague Documentation](#112-vague-documentation)
			- [1.1.3. Potentially Undefined Behavior](#113-potentially-undefined-behavior)
		- [1.2. Benefits](#12-benefits)
			- [1.2.1. Fast Code](#121-fast-code)
			- [1.2.2. Safe Code](#122-safe-code)
			- [1.2.3. Self-Explaining Code](#123-self-explaining-code)
	- [2. Proposed Language Changes](#2-proposed-language-changes)
		- [2.1. Immutable Fields](#21-immutable-fields)
		- [2.2. Immutable Methods](#22-immutable-methods)
		- [2.3. Immutable Arguments](#23-immutable-arguments)
		- [2.4. Immutable Return Values](#24-immutable-return-values)
		- [2.5. Immutable Variables](#25-immutable-variables)
	- [3. FAQ](#3-faq)
		- [3.1. Are the items within immutable slices/maps also immutable?](#31-are-the-items-within-immutable-slicesmaps-also-immutable)
		- [3.2. Go is all about simplicity, so why make the language even more complicated?](#32-go-is-all-about-simplicity-so-why-make-the-language-even-more-complicated)
		- [3.3. Aren't other features such as generics and better error handling not more important right now?](#33-arent-other-features-such-as-generics-and-better-error-handling-not-more-important-right-now)

## 1. Introduction
A Go 1 developer's current approach to immutability is copying because Go 1.x
doesn't currently provide any immutability annotations.

### 1.1. Current Problems
The current approach to immutability has a number of serious disadvantages
listed below

#### 1.1.1. Slow Code
Copies slow down our code and thus encourage us to write unsafe mutable APIs
when targeting optimal performance, you have to choose between performance or
safety.

#### 1.1.2. Vague Documentation
We have to manually describe what can be mutated and what the code user **must
not** mutate. This unnecessarily complicates the documentation and makes it
error prone.

#### 1.1.3. Potentially Undefined Behavior
Mutating what shouldn't be mutated can cause undefined behavior.

### 1.2. Benefits
Immutability can give us the following benefits:

#### 1.2.1. Fast Code
No need to make unnecessary copies and compromises between safety and
performance.

#### 1.2.2. Safe Code
A compile-time guarantee to avoid any undefined behavior that could have been
caused by violating immutability recommendations.

#### 1.2.3. Self-Explaining Code
No need to explicitly describe mutability recommendations in the documentation.

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
	ImmutableField const *Object // Immutable
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
.example.go:9:23 cannot assign to immutable field `Object.ImmutableField` of type `const *Object`
.example.go:21:25 cannot assign to immutable field `Object.ImmutableField` of type `const *Object`
.example.go:22:40 cannot assign to immutable field `Object.ImmutableField` of type `const *Object`
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
func (o *Object) MutatingMethod() const *Object {
    o.mutableField = &Object{}
    return o.ImmutableMethod()
}

// ImmutableMethod is a const method.
// It's illegal to mutate any fields of the receiver.
// It's illegal to call mutating methods of the receiver
func (o const *Object) ImmutableMethod() const *Object {
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
.example.go:15:7 cannot call mutating method `Object.MutatingMethod` on immutable receiver `o` of type `const *Object`
.example.go:16:21 cannot assign to contextually immutable field `Object.mutableField` of type `*Object`
.example.go:23:9 cannot call mutating method `Object.MutatingMethod` on contextually immutable variable `obj` of type `const *Object`
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
func (o const *Object) ImmutableMethod() {}

// MutateObject mutates the given object
func MutateObject(obj *Object) {
	obj.MutableField = &Object{}
}

// ReadObj is guaranteed to only read the object passed by the argument
// without mutating it in any way
func ReadObj(
	obj const *Object // Immutable
) ([]BankAccounts, error) {
	MutateObject(obj)            // Compile-time error
	obj.MutatingMethod()         // Compile-time error
	obj.MutableField = &Object{} // Compile-time error
}
```
**Expected compilation errors:**
```
.example.go:21:19 cannot use obj (type const *Object) as type *Object in argument to MutateObject
.example.go:22:9 cannot call mutating method `Object.MutatingMethod` on immutable variable `obj` of type `const *Object`
.example.go:23:23 cannot assign to contextually immutable field `Object.MutableField` of type `*Object`
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
func ReturnImmutable() const *Object {
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
.example.go:20:37 cannot assign to field `Object.MutableField` of contextually immutable variable `immutableVariable` of type `const *Object`
.example.go:21:23 cannot call function `Object.MutatingMethod` with non-const receiver on contextually immutable variable `immutableVariable` of type `const *Object`
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
func (o const *Object) ImmutableMethod() {}

// NewObject creates and returns a new mutable object instance
func NewObject() *Object {
	return &Object{}
}

// MutateObject mutates the given object
func MutateObject(obj *Object) {
	obj.MutableField = &Object{}
}

func main() {
	const obj = NewObject()
	obj.MutableField = &Object{} // Compile-time error
	obj.MutatingMethod()         // Compile-time error
	MutateObject(obj)            // Compile-time error
}
```
**Expected compilation errors:**
```
.example.go:23:23 cannot assign to contextually immutable field `Object.MutableField` of type `*Object`
.example.go:24:9 cannot call mutating method `Object.MutatingMethod` on immutable variable `obj` of type `const *Object`
.example.go:25:19 cannot use obj (type const *Object) as type *Object in argument to MutateObject
```

## 3. FAQ

### 3.1. Are the items within immutable slices/maps also immutable?
**No**, they're not! An immutable slice/map of mutable objects is declared this way:
```go
type ImmutableSlice const []*Object
type ImmutableMap   const map[*Object]*Object
```
If you want the items of an immutable slice/map to be immutable as well, you'll
need to declare them using the `const` qualifier:
```go
type ImmutableSlice     const [] const *Object
type ImmutableMap       const map[*Object] const *Object
type ImmutableMapAndKey const map[const *Object] const *Object
```
A deeply-immutable matrix could therefore be declared the following way:
```go
type ImmutableMatrix const [] const [] int
```

---

### 3.2. Go is all about simplicity, so why make the language even more complicated?
The `const` qualifier **doesn't** make Go "more complicated", instead it's
**just the opposite**! The complexity created by unsafe mutable references
**is** what makes
Go more complicated to work with, because you always have to remember to copy
stuff that you don't want others to be able to mutate, or at least explicitly
advise to "not mutate certain stuff" in the documentation running the risk of
breaking your inattentive colleague's code:

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
func (s const *Server) ConnectedClients() const []Client {
	return s.clients
}
```
**Conclusion**: Compiler-enforced restrictions to what is expected to be mutable
and what is not makes your code more readable for both you and other developers
working with your code/library and potentially saves you hours of debugging
broken code caused by mutable references. Essentially immutability is a strict
contract enforced by the compiler, and Go encourages compile-time safety by its
nature.

---

### 3.3. Aren't other features such as generics and better error handling not more important right now?
**No**, absolutely not! Unlike with topics such as *"generics"* and *"how to
handle errors more elegantly"* there's really not much to argue about in case of
immutability. It should be clear that it makes code both safer and easier to
make sense of. It doesn't require any breaking changes and it doesn't
even require a single new language keyword. Therefore immutability should be
considered of higher priority compared to other previously mentioned topics.

----
Copyright © 2018 [Roman Sharkov](https://github.com/romshark)
(<roman.sharkov@qbeon.com>)
