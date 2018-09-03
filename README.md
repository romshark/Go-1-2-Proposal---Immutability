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
type Person struct {
	Name const string                   // Immutable
	BankAccount const *BankAccount      // Immutable
	PhoneNumber string
}

func (p *Person) SomeMethod() {
	person.PhoneNumber = "54321"
	person.BankAccount = &BankAccount{} // Compile-time error
	person.Name = "Bill"                // Compile-time error
}

func main() {
	// Immutable fields are immutable once the object is initialized
	person := Person{
		Name: "Joe",
		PhoneNumber: "12345",
	}
	person.BankAccount = &BankAccount{} // Compile-time error

	person.PhoneNumber = "54321"
	person.Name = "Bill"                // Compile-time error
}
```
```
.example.go:9:25 cannot assign to immutable field `Person.BankAccount` of type `const *BankAccount`
.example.go:10:17 cannot assign to immutable field `Person.Name` of type `const string`
.example.go:19:25 cannot assign to immutable field `Person.BankAccount` of type `const *BankAccount`
.example.go:22:17 cannot assign to immutable field `Person.Name` of type `const string`
```

----
### 2.2. Immutable Methods
```go
type Object struct {
	MutableString string // Mutable field
}

// MutatingMethod is a non-const method
func (o *Object) MutatingMethod() {
	o.MutableString = "new value"
}

// ConstMethod has immutable pointer-receiver and is therefore a const method.
// It's illegal to mutate mutable fields and call mutating methods inside it
func (o const *Object) ConstMethod() {
	o.MutableString = "new value" // Compile-time error
	o.MutatingMethod()            // Compile-time error
}
```
```
.example.go:13:22 cannot assign to field `Object.MutableString` of immutable receiver `o` of type `const *Object`
.example.go:14:7 cannot call mutating method `Object.MutatingMethod` on immutable receiver `o` of type `const *Object`
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
type Person struct {
	Name string
}

// SetName has mutable receiver and is therefore a non-const method
func (p *Person) SetName(newName string) {
	p.Name = newName
}

// Generate returns an immutable slice of mutable objects
func Generate() (persons const []*Person) {
	persons = []*Person{
		&Person{Name: "Joe"},
		&Person{Name: "Jack"},
		&Person{Name: "Jordan"},
	}
}

func main() {
	list := Generate()

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
	p.SetName("Johan")
}
```
```
.example.go:22:14 cannot assign to contextually immutable variable `list` of type `const []*Person`
.example.go:23:11 cannot assign to contextually immutable variable `list` of type `const []*Person`
.example.go:24:11 cannot assign to contextually immutable variable `list` of type `const []*Person`
.example.go:28:17 cannot assign to contextually immutable variable `subList` of type `const []*Person`, immutability is inherited from variable `list`
.example.go:29:14 cannot assign to contextually immutable variable `subList` of type `const []*Person`, immutability is inherited from variable `list`
.example.go:30:14 cannot assign to contextually immutable variable `subList` of type `const []*Person`, immutability is inherited from variable `list`
```

----
Copyright © 2018 [Roman Sharkov](https://github.com/romshark)
(<roman.sharkov@qbeon.com>)