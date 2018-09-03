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
```
.example.go:9:23 cannot assign to immutable field `Object.ImmutableField` of type `const *Object`
.example.go:21:25 cannot assign to immutable field `Object.ImmutableField` of type `const *Object`
.example.go:22:40 cannot assign to immutable field `Object.ImmutableField` of type `const *Object`
```

----
### 2.2. Immutable Methods
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
```
.example.go:15:7 cannot call mutating method `Object.MutatingMethod` on immutable receiver `o` of type `const *Object`
.example.go:16:21 cannot assign to contextually immutable field `Object.mutableField` of type `*Object`
.example.go:23:9 cannot call mutating method `Object.MutatingMethod` on contextually immutable variable `obj` of type `const *Object`
```

----
### 2.3. Immutable Arguments
```go
type Person struct {
	Name string
}

// SetName has mutable receiver and is therefore a non-const method
func (p *Person) SetName(newName string) {
	p.Name = newName
}

// GetName has immutable receiver and is therefore a const method
func (p const *Person) GetName(newName string) {
	p.Name = newName
}

func ReadList(
	person const *Person // Immutable
	list const []*Person // Immutable
) ([]BankAccounts, error) {
	person.SetName("Joe")          // Compile-time error
	person.GetName()

	list[0] = &Person{}            // Compile-time error
	list = nil                     // Compile-time error
	list = append(list, &Person{}) // Compile-time error

	// Sub-slices inherit immutability from the original slice
	subList := list[1:2]
	subList[0] = &Person{}            // Compile-time error
	subList = nil                     // Compile-time error
	subList = append(list, &Person{}) // Compile-time error

	// Calling non-const methods on slice items is legal
	// because the slice isn't declared as a deeply immutable argument
	p := list[0]
	p.SetName("Joe")
}
```
```
.example.go:19:12 cannot call function `Person.SetName` with non-const receiver of type `*Person` on immutable argument `person` of type `const *Person`
.example.go:22:14 cannot assign to immutable argument `list` of type `const []*Person`
.example.go:23:11 cannot assign to immutable argument `list` of type `const []*Person`
.example.go:24:11 cannot assign to immutable argument `list` of type `const []*Person`
.example.go:28:17 cannot assign to contextually immutable variable `subList` of type `const []*Person`, immutability is inherited from argument `list`
.example.go:29:14 cannot assign to contextually immutable variable `subList` of type `const []*Person`, immutability is inherited from argument `list`
.example.go:30:14 cannot assign to contextually immutable variable `subList` of type `const []*Person`, immutability is inherited from argument `list`
```

----
### 2.4. Immutable Return Values
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
```
.example.go:20:37 cannot assign to field `Object.MutableField` of contextually immutable variable `immutableVariable` of type `const *Object`
.example.go:21:23 cannot call function `Object.MutatingMethod` with non-const receiver on contextually immutable variable `immutableVariable` of type `const *Object`
```

----
Copyright © 2018 [Roman Sharkov](https://github.com/romshark)
(<roman.sharkov@qbeon.com>)