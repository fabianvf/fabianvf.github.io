---
layout: post
title:  "Function Composition with Types in Python"
date:   2015-11-18 19:00:00
categories: blog
---

This is a followup to my last blog post [Function Composition in Python](http://fabianvf/2015/09/09/function-composition.html).
In that post, I talked about how to implement function composition with various operators in Python. Today, I'm going to make
use of one of the features that is new in Python 3.5, and show you how to take it one step further to typed function composition.

Function composition is actually pretty interesting, in terms of its type signature. It is a function that takes two functions,
and returns a new function with the input type of the second function and the output type of the first. In Haskell, the type
signature would look something like this:

`
compose :: (b -> c) -> (a -> b) -> (a -> c)
`
Compose is a function that takes two arguments, the first, a function that takes a value of type b and returns a value of type c,
the second, a function that takes a value of type a and returns a value of type b. Compose returns a new function, which takes a value
of type a, and returns a value of type c (it's basically cutting out the middleman).

For those who took a logic class, this might seem a little familiar:

    b -> c
    a -> b
    ------
    a -> c


Now in haskell, this is easy to specify and type check. What I am interested in exploring is whether or not Python's new
"type-hints" system is powerful enough to express this same meaning, and capture errors when the output type of the second
function doesn't match the input type of the first. Let's start with a simple definition of function composition, stolen from the last
article.

    def compose(f1, f2):
        def inner(argument):
            return f1(f2(argument))
        return inner

Now, we'll annotate that with types, from Python 3.5's new type hints system. This might get a little messy.

    from typing import Callable, TypeVar

    T1 = TypeVar('T1')
    T2 = TypeVar('T2')
    T3 = TypeVar('T3')

    def compose(f1: Callable[[T2], T3],
                f2: Callable[[T1], T2]) -> Callable[[T1], T3]:
        def inner(arg: T1) -> T3:
            return f1(f2(arg))
    return inner

Python 3.5 introduced the `typing` module, which has a variety of functions classes for defining types. One of the more powerful features is `TypeVar`. `TypeVar` allows you to define a variable that can be many types, but within a declaration, all instances of a specific `TypeVar` must be the _same_ type. For something like function composition, this is a necessity, because the `compose` function does not know or care what the input/output types of the input functions are, so long as the input of one matches the output of the other.

For function composition, we need three `TypeVar`s - the first, to represent the input of second function (in this case, `T1`), the second, to represent the output of the second function (`T2`), which is also the input of the first function, and the third, to represent the output of the first function (`T3`). Now, let's dig into the actual signature of compose.

Let's start with our two arguments, `f1` and `f2`. They both have the structure `Callable[[InType], OutType]`, where `InType` is the input type of the function, and `OutType` is the return type of the function. `InType` is encapsulated by a list because functions can have multiple arguments in Python. The `Callable` after the `->` indicates the return type, in this case, our composed function, with an input type of `T1` and an output type of `T3`. This declaration of compose is a little wordy, but I think it is actually pretty elegant, once you get past the syntax. Now, the real question is, does it work? Let's say we have the following toy functions:

    def plus2(x):
        return x + 2

    def minus2(x):
        return float(x - 2)

Uh-oh - minus2 silently converts everything to a float. Let's see what these would look like with types:

    def plus2(x: int) -> int:
        return x + 2

    def minus2(x: int) -> float:
        return float(x - 2)

So let's say we wanted to make a completely useless function, `add0`. We can do this with compose in two ways

    add0_plus_first = compose(minus2, plus2)
    add0_minus_first = compose(plus2, minus2)

Now, let's figure out what the types of these functions are. `plus2` takes an `int` and returns an `int`, and `minus2` takes an `int` and returns a `float`. This means that `add0_plus_first` would have the type:

    plus2: (T1: int) -> (T2: int)
    minus2: (T2: int) -> (T3: float)
    --------------------
    add0_plus_first: (T1: int) -> (T3: float)

This is perfectly fine, type-wise. The input type `T2` of `minus2` matches the output type `T2` of `plus2`. `T2` is easily resolvable to `int`. The problem is with add0_minus_first. `minus2` returns a float, but `plus2` expects an `int`.

    minus2: (T1: int) -> (T2: float)
    plus2: (T2: int) -> (T3: int)
    --------------------
    N/A

This means that in the compose function, `T2` would have to be of type `int` and of type `float`. Obviously, it cannot be both, so any good type system would reject it.

Python 3.5 technically only included the ability to add type-hints to your code, but does not actually provide any type-checking functionality. For that, I used a static type-checker for python called [mypy](http://mypy-lang.org/). The result of running `mypy typed_compose.py` (you can download [typed_compose.py here](https://gist.github.com/fabianvf/a671bc0a0d9ff14f5efb)) is:

    compose.py:24: error: Cannot infer type argument 1 of "compose"

Success! This failed on line 24, the line on which we create `add0_minus_first`. Unfortunately, the error is kind of vague (what is type argument 1?), but I think it's fantastic that mypy was able to catch this error at all -- improving the error messages is the easier part. So there we have it, a type-checkable compose function in Python. Thanks for sticking with me through that, and I hope it wasn't too hard to follow.
