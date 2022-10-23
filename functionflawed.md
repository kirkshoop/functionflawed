---
title: "Function Fundamentally Flawed - visit() and apply() should not exist"
subtitle: "Draft Proposal"
document: D0000R0
date: today
audience:
  - "LEWG Library Evolution"
author:
  - name: Kirk Shoop
    email: <kirk.shoop@gmail.com>
toc: true
---

Introduction
============

~~The common definition of function, across nearly all languages, for nearly all-time, is not composable due to asymmetry.~~

Yeah, that is my customary approach - say something that causes a strong visceral reaction, such that everything that follows is immediately dismissed. Stick around for the meat. 

This paper is the outcome of several years of trying to find a general pattern for asynchronous composition. I tried threads and mutexes. I tried atomics. I tried fibers. I tried raw-callback-functions. I wrote rxcpp v1. I wrote rxcpp v2, I wrote pushmi to introduce Sender/Receiver. I moved to Lewis Baker's implementation of Sender/Receiver, called libunifex. I learned a lot from Lewis Baker on how to structure async lifetimes and how coroutines in C++ map from the existing function definition. I convinced Lewis Baker that the C++ coroutines were limited in ways that Sender/Receiver were not. Lewis Baker started defining coroutines v2. C++20 shipped coroutines v1.

So here we are.

[exceptions, global state, global lock](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2022/p2544r0.html)

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

"Thats not valid c++!", "If c++ had a language tuple, this would be easy!", "Other languages have a language tuple to solve this!", "Use library `std::tuple<>` to demonstrate composability!"

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

"Perfect!", "That's ugly!", "No! Not all instances of `std::variant<A, B>` represent argument packs, implicitly converting a `variant<A, B>` to an argument pack is a bug!"

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

Well, besides the need to support forward_set_value having overloads, `thenplus` also has to ensure that the given function does call forward_set_value. not to do so is a bug.

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

> Q: Is `then()` the only algorithm affected by the asymmetry?

No, many other algorithms are impacted: `sync_wait()`, `when_all()`, `upon_error`, `upon_stopped`, `stopped_as_optional`, and `stopped_as_error` come to mind initially. 

## what asymmetries remain?

The ones that we want to be asymmetric. 

A function result has a representation for error and we want that to be asymmetric. We want to return an error result from the function and not have the error value forwarded to another nested function call. We want to skip any other nested function call when an error result occurs.

A function result has a representation for stop and we want that to be asymmetric. We want to return a stop result from the function and not have the stop result forwarded to another nested function call. We want to skip any other nested function call when a stop result occurs.

One suggestion made to me was to use `expected<>` as an input argument to every function to improve composition.

There are two options for how that might look shown below:

> Note: Think about how common it will be to pass the wrong expected along. 

> Note: Think about all the extra function calls and branches and copies of the error value on the error-path. 

1. Use the preceding expected exactly and transform it into the new `expected<>` when it contains an error.

```
template<class R, class Unrelated, class... Tn>
expected<R> make(expected<Unrelated> exp, Tn&&... ) {
  if (exp.has_error()) { return {move(exp)}; }
  /// ... produce R from Tn... or fail.
}

template<class Unrelated>
void function(expected<Unrelated> exp) {
  auto nam = make<myname>(exp, "name");
  auto addr = make<myaddress>(nam, "street", "state", "zip");
  auto eml = make<myemail>(addr, "email");
  auto client = make<myclient>(eml, nam, addr, eml);
  // ... etc..
}
```

> Note: Think about the `Unrelated` value passed into every function and what it means for reasoning about the inputs to the function.

This option does not improve composability. functions have multiple arguments and only one result.

2. Transform preceding expected into a new `expected<>` that has all the arguments.

```
template<class R, class Ignored, class... Tn>
expected<tuple<Tn...>> args_from(expected<Ignored> exp, Tn&&... tn) {
  if (exp.has_error()) { return {move(exp)}; }
  return tuple<Tn...>{(Tn&&)tn...};
}

template<class... Rn, class... Tn>
expected<tuple<Rn>> make(expected<tuple<Tn...>> tn) {
  if (tn.has_error()) { return {move(tn)}; }
  /// ... produce tuple<Rn...> from tuple<Tn...> or fail.
}

void function(expected<tuple<>> void_arg) {
  auto nam = make<myname>(args_from(void_arg, "name"));
  auto addr = make<myaddress>(args_from(nam, "street", "state", "zip"));
  auto eml = make<myemail>(args_from(addr, "email"));
  auto client = make<myclient>(args_from(eml, nam, addr, eml));
  // ... etc..
}
```

Now we have symmetry! It is a good thing that I did not type out all the tuple unpacking..

> Note: Now there are two branches per function call. `args_from()` adds the second.

> Note: The success path does a lot of round-trips into and out of tuples and also round-trip tuples into and out of expecteds.

## function composition?

Is this what we want when we compose functions? 

I hope not. I certainly do not want these limitations on function composition.


Proposal
========

There is a way out.

[functions that return multiple values, errors, and can stop](https://godbolt.org/z/5cnd43Ts6)

