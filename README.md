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
	- [2. Proposed Language Features](#2-proposed-language-features)
		- [2.1. Immutable Fields](#21-immutable-fields)
		- [2.2. Immutable Methods](#22-immutable-methods)
		- [2.3. Immutable Arguments](#23-immutable-arguments)
		- [2.4. Immutable Return Values](#24-immutable-return-values)
		- [2.5. Immutable Variables](#25-immutable-variables)

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

## 2. Proposed Language Features

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
    o.MutatingMethod()         // Compile-time method
    o.mutableField = &Object{} // Compile-time method
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

----
Copyright © 2018 [Roman Sharkov](https://github.com/romshark)
(<roman.sharkov@qbeon.com>)