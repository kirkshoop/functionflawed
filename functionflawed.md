---
title: "Function Fundamentally Flawed - visit() and apply() should not exist"
subtitle: "Draft Proposal"
document: D0000R0
date: today
audience:
  - "LEWG Library Evolution"
author:
  - name: Doe
    email: <john@doe.name>
toc: true
---

Introduction
============

~~The common definition of function, across nearly all languages, for nearly all-time, is not composable due to asymmetry.~~

Yeah, that is my customary approach - say something that causes such a visceral reaction that everything that follows is immediately dismissed. Story of my life. This paper is the outcome of several years of trying to find a general pattern for asynchronous composition. I tried threads and mutexes. I tried atomics. I tried fibers. I tried callbacks. I wrote rxcpp v1. I wrote rxcpp v2, I wrote pushmi. I moved to Lewis Baker's implementation of Sender/Receiver, libunifex. I learned a lot from Lewis Baker on how to structure async lifetimes and how coroutines in C++ map from the existing function definition. I convinced Lewis Baker that the C++ coroutines were limited in ways that Sender/Receiver were not. Lewis Baker started defining coroutines v2. C++20 shipped coroutines v1.

So here we are.

## Asymmetry of Arity

Start with a simple function, `inc()`.

```cpp
int inc(int v) {return v+1;}
```

Compose `inc()`.

```cpp
int two = inc(inc(0));
```

Define `inc_each()`.

```cpp
@_?_@ inc_each(int v0, int v1) {return(inc(v0), inc(v1));}
```

Compose `inc_each()`.

```cpp
int four, two = inc_each(inc_each(2, 0));
```

"Thats not valid c++!", "If c++ had a language tuple, this would be easy!", "Other languages have a language tuple to solve this!", "Use library `std::tuple<>` to demonstrate composablity!"

```cpp
std::tuple<int, int> inc_each(int v0, int v1) {return std::make_tuple(inc(v0), inc(v1));}
```

Compose 'tuply' `inc_each()`.

```cpp
std::tuple<int, int> fortyTwo = inc_each(inc_each(2, 0));
```

"That isn't valid c++!", "Just use structured binding to decompose the tuple!"

> Q: Once we start using the word 'decompose', is this composition anymore? 

```cpp
std::tuple<int, int> fortyTwo = inc_each(auto [forty, two] = inc_each(2, 0));
```

"That isn't valid c++!", "Decompose the tuple correctly!"

```cpp
auto [three, one] = inc_each(2, 0);
std::tuple<int, int> fortyTwo = inc_each(three, one);
```

"Looks Great!", "That is not function composition!", "Use `std::apply()` to decompose and compose with a callback!"

```cpp
std::tuple<int, int> fortyTwo = std::apply(inc_each(2, 0), 
  [](int three, int one){
    return inc_each(three, one);
  });
```

"Perfect!", "That's ugly!", "That inverts the visual and actual composition order!", "If c++ had pattern matching, this would be easy!", "Use the pattern matching proposal to decompose and compose!"

```cpp
std::tuple<int, int> fortyTwo = inspect(inc_each(2, 0)) {
  (int three, int one): inc_each(three, one);
};
```

"Perfect!", "That's ugly!", "That inverts the visual and actual composition order!", "The definition of `inc_each()` is wrong, fix it!"

```cpp
std::tuple<int, int> inc_each(std::tuple<int, int> v) {
  return std::make_tuple(inc(std::get<0>(v)), inc(std::get<1>(v)));
}
```

```cpp
std::tuple<int, int> fortyTwo = inc_each(inc_each(std::make_tuple(2, 0)));
```

"Perfect!", "That's ugly!", "No! Not all `std::tuple<int, int>` are argument packs, implicitly converting a `tuple<int, int>` to an argument pack is a bug!"

> Q: If allowing only a single argument is the answer for function composablity, why do we allow more than one argument to a function?

"Perfect! Forcing a single argument would be symmetric! We can always compose the single argument out of sum and product types!"


> Q: Have you compared the code gen for tuple to function argument passing lately?

> [godbolt](https://godbolt.org/z/5cWEr7497)

## Asymmetry of Kind

Given:

```cpp
struct A {};
struct B {};
using aorb = std::variant<A, B>;
int count = 0;
```

Start with a simple function, `ab()`.

```cpp
@_?_@ ab(A a) {
  if (count++ & 1) {
    return a;
  }
  return B{};
}
@_?_@ ab(B b) {
  if (count++ & 1) {
    return A{};
  }
  return b;
}
```

Compose `ab()`.

```cpp
A a = ab(ab(A{}));
```

"Thats not valid c++!", "If c++ had a language variant, this would be easy!", "Other languages have a language variant to solve this!", "Use library `std::variant<>` to demonstrate composablity!"

```cpp
aorb ab(A a) {
  if (count++ & 1) {
    return a;
  }
  return aorb{B{}};
}
aorb ab(B b) {
  if (count++ & 1) {
    return aorb{A{}};
  }
  return b;
}
```

Compose `ab()`.

```cpp
A a = ab(ab(A{}));
```

"That isn't valid c++!", "Decompose the variant correctly!", "Use `std::visit()` to decompose and compose with a callback!"

```cpp
aorb a = std::visit([](auto v) -> aorb {return ab(v);}, ab(A{}));
```

"Perfect!", "That's ugly!", "That inverts the visual and actual composition order!", "If c++ had pattern matching, this would be easy!", "Use the pattern matching proposal to decompose and compose!"

```cpp
aorb a = inspect(ab(A{})) {
  (A a): ab(a);
  (B b): ab(b);
};
```

"Perfect!", "That's ugly!", "That inverts the visual and actual composition order!", "The definition of `ab()` is wrong, fix it!"

```cpp
aorb ab(aorb v) {
  return std::visit([]<class V>(V v){
    if constexpr (same_as<A, V>) {
      if (count++ & 1) {
        return v;
      }
      return aorb{B{}};
    }
    else if constexpr (same_as<B, V>) {
      if (count++ & 1) {
        return aorb{A{}};
      }
      return v;
    }, v);
}
```

```cpp{
aorb a = ab(ab(aorb{A{}}));
```

"Perfect!", "That's ugly!", "No! Not all `std::variant<A, B>` are argument packs, implicitly converting a `variant<A, B>` to an argument pack is a bug!"

> Q: Have you compared the code gen for variant to function argument passing lately?

[godbolt](https://godbolt.org/z/j6Mfv15Eo)

Description
===========

## Arity & Kind Asymmetries

- __Arity__ A result is a value, function arguments are a set of values.
- __Kind__ A result is one type, an overload set of functions, when used as a callback, represent a set of argument sets, each of which is a set of values.

The way these functions are made to compose is to create sum and product types and syntax to pack and unpack values. If a function result was always a set of values, rather than a single value, and a function definition had better syntax for creating and using overload sets, then composition would be natural and the compiler would be free to treat results the same way it treats arguments (no specified layout).

## `then()` in Sender/Receiver

[@P2300R4] includes the `then()` algorithm. `then()` is the simplest of algorithms. implement a function, apply the arguments to a given function, and then forward the result of the given function.

```cpp
template<class... An>
void forward_set_value(An&&...) {
  //...
}

auto given_fn(int v0, int v1) {
   @_return(v0+1, v1+1);_@
}

template<class... An>
void then_set_value(An&&... an) {
  forward_set_value(given_fn(an...));
}
```

The asymmetry prevents the given function from passing through all the arguments to the forward set_value function.

What about making this into the given functions problem?

```cpp
template<class... An>
void forward_set_value(An&&...) {
  //...
}

auto given_fn(auto Fn, int v0, int v1) {
   Fn(v0+1, v1+1);
}

template<class... An>
void thenplus_set_value(An&&... an) {
  given_fn(&forward_set_value, an...);
}
```

Well, besides the need to support forward_set_value having overloads, thenplus also has to ensure that the given function does call forward_set_value. not to do so is a bug.

```cpp
template<class... An>
void forward_set_value(An&&...) {
  //...
}

auto given_fn(auto Fn, int v0, int v1) {
   Fn(v0+1, v1+1);
}

template<class... An>
void thenplus_set_value(An&&... an) {
  bool called = false;
  auto fn = [&called](auto... an){
    called = true;
    forward_set_value(an...);
  };
  given_fn(fn, an...);
  if (!called) {std::terminate();}
}
```

Now, is this what we mean when we say function composition? 

I hope not.

## what asymmetries remain?

The ones that we want to be asymmetric. 

A function result has a representation for error and we want that to be asymmetric. We want an error result to be used to return from the function not have it forwarded to another nested function call.

Proposal
========



