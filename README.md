![Immutability - A Go 2 Language Feature Proposal](https://github.com/romshark/Go-2-Proposal---Immutability/blob/master/document_header.png "Immutability - A Go 2 Language Feature Proposal")

# Go 2 - Immutability
This document describes a Go 2 (*Go > 1.11*) language feature proposal to
immutability.

Author: [Roman Sharkov](https://github.com/romshark) (<roman.sharkov@qbeon.com>)

**Table of Contents**
- [Go 2 - Immutability](#go-2---immutability)
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
		- [2.6. Slice Aliasing](#26-slice-aliasing)
	- [3. FAQ](#3-faq)
		- [3.1. Are the items within immutable slices/maps also immutable?](#31-are-the-items-within-immutable-slicesmaps-also-immutable)
		- [3.2. Go is all about simplicity, so why make the language more complicated?](#32-go-is-all-about-simplicity-so-why-make-the-language-more-complicated)
		- [3.3. Aren't other features such as generics and better error handling not more important right now?](#33-arent-other-features-such-as-generics-and-better-error-handling-not-more-important-right-now)
		- [3.4. Why overload the `const` keyword instead of introducing a new keyword like `immutable` etc.?](#34-why-overload-the-const-keyword-instead-of-introducing-a-new-keyword-like-immutable-etc)
		- [3.5. How are constants different from immutables?](#35-how-are-constants-different-from-immutables)
		- [3.6. Why do we need immutable receivers if we already have copy-receivers?](#36-why-do-we-need-immutable-receivers-if-we-already-have-copy-receivers)

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
) {
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

### 2.6. Slice Aliasing
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

### 3.2. Go is all about simplicity, so why make the language more complicated?
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
func (s const *Server) ConnectedClients() const []Client {
	return s.clients
}
```

---

### 3.3. Aren't other features such as generics and better error handling not more important right now?
**No**, absolutely not! Unlike with topics such as *"generics"* and *"how to
handle errors more elegantly"* there's really not much to argue about in case of
immutability. It should be clear that it makes code both safer and easier to
make sense of. It doesn't require any breaking changes and it doesn't
even require a single new language keyword. Therefore immutability should be
considered of higher priority compared to other previously mentioned topics.

----

### 3.4. Why overload the `const` keyword instead of introducing a new keyword like `immutable` etc.?
**Backwards-compatibility**. Using the const keyword would allow us to introduce
immutability to Go 1.x without having to make breaking changes to the language.
The introduction of a new keyword could potentially break existing Go 1.x code,
where the new keyword might be used for naming symbols causing build conflicts.
`const` on the other hand is already a [reserved language
keyword](https://golang.org/ref/spec#Keywords) which doesn't interfere with the
proposed language changes and verbally comes close to the desired meaning (for
example, C++ uses the `const` keyword to do just that).

----

### 3.5. How are constants different from immutables?
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

### 3.6. Why do we need immutable receivers if we already have copy-receivers?
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

----
Copyright © 2018 [Roman Sharkov](https://github.com/romshark)
(<roman.sharkov@qbeon.com>)
