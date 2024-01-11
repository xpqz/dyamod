# Modules for Standard ML

This is a summary of the ideas presented in the paper [Modules for Standard ML](https://dl.acm.org/doi/pdf/10.1145/800055.802036) by David MacQueen.

[Standard ML](https://en.wikipedia.org/wiki/Standard_ML) is a strongly typed functional programming language, and one of the inspirations for [OCaml](https://ocaml.org/), [Haskell](https://www.haskell.org/) and Microsoft's [F#](https://learn.microsoft.com/en-us/dotnet/fsharp/what-is-fsharp). It uses the [Hindley-Milner](https://en.wikipedia.org/wiki/Hindley%E2%80%93Milner_type_system) type system. 

The Standard ML (SML) module system was cited as an inspiration for that of [Scheme 48](https://github.com/xpqz/dyamod/blob/main/S48M.md).

## Short summary

The paper discusses the design principles and the implementation of modules in ML, focusing on managing environments, inheritance, and sharing. The design is based on treating declarations and environments as "quasi-first-class entities" in the language: *modules* and *instances*. The system supports abstraction with respect to free type and value identifiers and deals with inheritance and sharing between environments. 

The paper also discusses the syntax for signatures, modules, and instances, along with necessary extensions to the ML typing rules. It emphasises the separate definition of interfaces and their implementations and ensures that the defined environment of a module is closed and self-contained. The system is designed to help with managing large-scale program structures and supports type-secure separate compilation of program components.

## Long summary

The design has three stated goals:

1. to facilitate the structuring of large SML programs,
1. to support separate compilation and generic library units; and
1. to employ new ideas in the semantics of data types to extend the power of SML's polymorphic type system.

The design is based on ideas already part of the structure of SML: a *declaration*, its *type signature* and the *environment* it denotes.

The main problem that the authors set out to solve is the first one: large program structure. They write:

> ...as one writes larger ML programs, serious problems of program organization and structure arise, and the type system needs to be augmented to help cope with these problems. As a practical matter, it is desirable that constructs for expressing large-scale program structure should also support type-secure separate compilation of program components and the resulting libraries of generic, pre-compiled units.

### Design principles

Given the stated design goals above, they list the following design principles:

> 1, Strict separation between the environments of *definition* and *use*, with an explicit specification of the complete environment of definition in terms of antecedent instances of specified signatures

I interpret this to mean that a module must be completely specified, and cannot rely on some type signatures being pulled in prior to the use site in the code. 

> 2, Modules are *parametric*. A module is a function producing environments of a particular signature when applied to argument instances of specified signatures.

This sounds like "generics" -- in a typed language, you want to be able to have a module be parametric in argument type, so that, for example, you can write a numeric function that takes an argument of type `T` and then instantiate this with whatever appropriate numeric type you want, without having to copy the function for each type.

> 3. Separate definition of *interfaces* (signatures) and their implementations (modules). Essential for parametric modules. 

Maybe this is evident in a language with a HM type system? C#'s generics doesn't necessarily have this restriction. Let's take it at face value.

> 4. The defined environment of a module must be *closed*, meaning (amongst other things) that no types may appear in its signature that are not defined directly or indirectly in the environment. This may mean that the defined environment may sometimes need to include or inherit from other modules. 

Fair enough, going back to the first point.

> 5. We introduce environments of *named signatures*, *modules* and *instances*. Such environments may be partially persistent, forming permanent systems or libraries.

> 6. We minimize the visibility of information by requiring explicit declaration of inheritance (information hiding).

Note that this is *type* inheritance, not inheritance in the OO sense. 

> 7. Shared antecedents requierd for coherent interaction between module parameters must be excplicitly specified.

This seems like something obvious stated in a complicated way. Everything must be fully specified. Maybe I'm reading it wrong?

## Basic proposal

Three constructs make up the module proposal:

1. Signatures
1. Modules
1. Instances. 

The *instance* is the core component. Note that *instance* here is not the *instance* concept commonly used in OO languages; it seems closer to that of a Dyalog namespace:

> ...an environment structure (called an *instance* here) consisting of a set of bindings of *types*, *values* and *exceptions*. An instance has two main functions: (1) it is a hybrid collection of entities incorporating related types, values, and exceptions; and (2) it provides names for its constituents. 

Instances also have their own kind of type, known as a *signature*. The instance signature specifies the types of each of the bindings it contains (like a C header file?). They give an example of an instance signature for a module implementing a stack:

```sml
signature STACK =
  sig
    type 'a stack
    val nilstack: 'a stack
    and push: 'a * 'a stack -> 'a stack
    and empty: 'a stack -> bool
    and pop: 'a stack -> 'a stack
    and top: 'a stack -> 'a
    exception pop: unit and top: unit
  end
```

with the corresponding module implementation:

```sml
module StackMod () : STACK
  type 'a stack = nilstack | push' of 'a * 'a stack
  exception pop: unit and top: unit
  val nilstack = nilstack'
  and push = push'
  and empty(nilstack) = true |
      empty _ = false
  and pop(push'(_, s)) = s |
      pop _ = escape pop
  and top(push'(x, _)) = x |
      top _ = escape top
end
```

Here, the body of `StackMod` has no free identifiers, so it takes an empty parameter list:

```sml
instance Stack = StackMod () { Stack: STACK }
```

The rest of the paper focuses on demonstrating some more in-depth interactions between modules, including implementation sharing and type inheritance.

## Summary

For us APLers, SML is quite distant: functionally pure, extremely strongly typed, and immutable. To me it seems that this avoids some of the problems with the issues we've been discussing in Dyalog of who 'owns' what when a module is imported in different places. With no mutable state, that sticking point goes away.












