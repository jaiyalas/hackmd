---
title: Readible protocol with optic 
status: Draft 
author: YunYan Chi (yunyan@marigold.dev)
date: 15/03/2021
---

## Summary

To apply `optics`, the composable abstract interface to the protocol layer for increasing the maintainability and readibility of codebase.

## Motivation

The current design uses `Alpha_context` as a barrier and provides an abstract interface of entire context. Every computation details are hidden underneath. For manipulating context, we only need to simply invoke modules and functions defined in `Alpha_context`. 

This layered structures provide a simple way to view context differently depending on what we want to compute. But with the expending of codebase, more and more details, conditions and invariants are hidden deeply inside this architecture. In the end, we might lose sight of the big picture of protocol and context, or even worse, misundersatnding the behavior or semantics of certain module and function.

Additionally lots of computation are wrapped inside monad: one of both of `tzresult` and `lwt`. This makes codes much harder to understand and hack.

We need some other way to write code in a more *composable* and *combinatorial* fashion. Plus, it should also be able to integrated with monadic binding easily.

### Rationale

Ther are two perspectives of why we need more *composability* and *combinatorial*.

As the very first reason, having one single well-defined function for one specific computation is usually enough for the most use cases. However, since it's just one single function, from the point of view of the caller, the information can be very limited. Say, we use `Gas.consume ctxt g` to update the gas state in context. It's quite simple and straightforward, yet, we don't actually know what kind of precondition or invariant this computation would satisfy. Therefore, to show more information lexically, there are some other way to do. Say, having a very long function name: 

~~~ocaml
let updateLimitedGasFromContext g = ...
~~~

or, having delicate modules: 

~~~ocaml
let ctx = Alpha_context.Gas.Limited.consume ctx g
~~~

Unfortunately neither of them can solve the problem once for all. One still need to change code here and there.

The second reason is, to accomplish a context-based stateful computation which has no predefined single corresponding function (in `Alpha_cntext` for example), one need to either *locally get-then-compute-then-update that contextual state*; or, *add more functions*. They both seems fine. But with more developers getting involved in this system, larger and more complex codebase we might have. 

## Specification

### lens

The Foundamental concept is simple. Given two data structures, say `a` and `b`, `lens` is nothing  but a wrapper of two functions between `a` and `b`; and, one of them is `from : a -> b`, the other one is `to : a -> a`. Depends on the discussing context, sometimes we could also call them `getter` and `setter`. This is also known as the **bidirectional** programming. 

The rigid definition of `lens`, there is an assumption (also a restriction) regarding those two types invloved. First, one of them must be more complicated than the other. And, this complexity is defined in terms of co-product type. Namely, this bidirectional mapping is between a co-product structure and a simple one. In other words, building a `lens` is basically telling how to get a field value out of a record type; and, how to putting or updating a field within a record type with a new value.

By just introducing `lens`, we immediately improve our expressivity of record type manipulation. But it's usually insufficient due to that not only do we use the co-product types like record type, but also the product types like variant types. If we only apply `lens` into a real system, it's basically no different than simply providing a group of getting and updating functions.

### lens-like

There are another two lens-like strictures we are interested in here - the `prism` and `iso`.

A `prism` binds a product type into a  simple structure capturing the concept of _one-or-none_. The possible and most common choices are `option` and `either`. A `prism` is defined by a pair of functions called `preview` and `review` which are acutally as the `from` and `to` functions.

It's very handy or even necessary to introduce `prism` because it gives us the capability of describing a mapping relation between any structure and a _one-or-none_ structure that can be viewed as a _predicate_. For example, let's we have a type 

```ocaml
type gas = Unlimited | {current : int} Limited 
```

With `prism` we can create a mapping between this `gas` and, say, `int option` named: 

`isLimited : gas (int option) prism`

This function allows us to obtain a value of current gas when it's in _limited_ mode.

An `Iso` is another lens-like structure which defines a relation between two structurally identical types. So it's not hard to imagine that an `Iso` is just a type transformer. A simple yet straightforward example is the correspondence between `Gas_limit_repr` and `Saturation_repr`.

### optics

Collectively all the lens-like structures mentioned above are called `optics`. The best thing is, they are not just some mutually independent structures. In fact, they can be tranformed from one to another depending on the use cases. 

Assuming we define `(|$)` as an optical composition and `(|>)` as flipped function application, a sequences of context updating could be written as: 

```ocaml 
ctx 
|> state1_in_context |$ predicate1 |$ update_with v1
|> state2_in_context |$ predicate2 |$ update_with v2
|> state3_in_context |$ predicate3 |$ update_with v3
```

That shows a clear representation of what we'd like to do with states in context. Taking gas consumption as an example, comparing with just writing 

```ocaml
let ctx = Gas.consume ctxt gas in 
```

the optics version can be written as follows

```ocaml
ctx 
|> Gas.fromContext |$ isLimited |$ Gas.consume gas
```

which could reveal more (pre)conditions and even some invariant of computation. And if there are something else needs to be revealed explicitly, it's also very simple to expend - we just adding more optical combinators.

### TODOs

- optics for general structures, say, pair, list, option, either, int, string, etc..
- optical composition
- intergrate with monads (aka monadic composition)
- optics for raw_context 
- optics for domain-specific structures, like Gas, Saturation, all *_repr etc..
- rewrite functions in, for example interpreter and apply, with the new optical fashion

## Backwards Compatibility

The optics changes nothing but the way of representation on sytax level. And the behavior and semantics will be intact. So there is no backwards Compatibility issue.