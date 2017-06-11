# Flattening Void

* Proposal: [SE-NNNN](NNNN-flatening-void.md)
* Authors: [Jérémie Girault](https://github.com/jeremiegirault)
* Review Manager: TBD
* Status: **Awaiting review**

## Introduction

[SE-0110](https://github.com/apple/swift-evolution/blob/master/proposals/0110-distingish-single-tuple-arg.md) in its current form improved the compiler consistency and reliability but caused regressions for people using generic-style swift especially around the `Void` type.
The goal of this proposal is to alter the `Void` type definition so as to preserve source compatibility and avoid generics regressions.

Swift-evolution thread: [Discussion thread topic for that proposal](https://lists.swift.org/pipermail/swift-evolution/)

## Motivation

`Void` was initially defined as the type of the empty tuple in swift because arguments of functions were still tuples at that time. This definition is now obsoleted by the changes made by in [SE-0110](https://github.com/apple/swift-evolution/blob/master/proposals/0110-distingish-single-tuple-arg.md) implementation which clearly differentiates arguments and tuples. This obsolescence causes multiple types of regressions / inconsistencies and is source-breaking for users of generic swift.

```swift
func test() { // equivalent to "func test() -> Void"
	// why don't I have to return () ? magic !
}
test()
```

or

```swift
typealias Callback<T> = (T) -> Void
let c1: Callback<Void> = { print("Callback<Void>") } 
// why don't I have to use "_ in" in c1 closure ? magic !
let c2: Callback<Int> = { print("Callback<Int>(\($0)") }

c2(42)
c1(()) // I have to use a double parenthesis where a single pair was valid in swift3. not magic ?
```

`Void` already contains some magic but not consistently.
There is a clear drawback in the clarity and usability of the new code syntax. The generic code is less concise and this new syntax does not express the intent (calling a Callback without arguments).
Due to the strength of the type-checking system, it can easily be problematic for the caller to find the correct arguments when calling a function when using Void arguments and/or generics.

## Proposed solution

This proposition considers that if arguments-as-tuple is obsolete, then Void-as-tuple is also obsolete.

In particular, `Void` should not mean an empty tuple argument (which made sense were arguments were tuples). `Void` is the **absence** of an argument.

Therefore this solution proposes that `Void` is detached from its initial tuple definition and attached a new a special behavior like `Never` is.
Void is proposed to become an intrinsic type with no possible value from code or possible extensions. It should not be instantiable from code.

The meaning of `Void` would be to be removed from the canonical signature of the function, which, with their, is the identity of the function. e.g.

```
func test<T>(t: T) {}
// becomes (in pseudo-code for comprehension)
test<Void>() {}
```

While being possible to remove `Void` from the caller perspective, it is a bit trickier from the callee side if the argument is used. Let's consider :

```
func test<T>(t: T) {
	print(t)
}
```

What should happen when T becomes `Void` ? Every pseudo-instance use should also be striped. In pseudo-language the above becomes :

```
func test<Void>() {
	print()
}
```

Actually in generics, if the type is not constrained (which would prevent `Void` to be used), either the `Void` type and its instances would be stripped from the code.


## Detailed design

- *(1)* Create `Void` intrinsic which can be used as a type. This type will have a special meaning
- *(2)* At the function signature site: from the initial function signature, create a "canonical" version of the signature where all `Void` parameters are removed. This canonical signature would be the identity of the function signature everywhere else in the code (especially when checking for conflicting signatures).
- *(3)* Strip all pseudo-instances of `Void`
- At the call site: the only signature to be used would be the "canonical" one (striped from `Void` arguments).
- In generics: `Void` can be used normally and generates types and functions as usual. Generated functions should respect rules *(2)* and *(3)*

## Source compatibility

Should improve swift3 to swift4 source compatibility

## Effect on ABI stability

Should not affect ABI (need confirmation)

## Effect on API resilience

Should not affect API (need confirmation)

## Alternatives considered

### No change

If no change happens, users will have to use a specific type for the neutral `Void` argument :

```
typealias CallbackOf<T> = (T) -> Void
typealias Callback = () -> Void
```

This would diminish the generic abilities of swift, because these types cannot be coerced and it would require two different functions to take a full callback parameter, which is a language regression.

