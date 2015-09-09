---
layout: post
title:  "Function Composition"
date:   2015-09-09 17:00:00
categories: blog
---

I'm not sure how many of you are familiar with the idea of function composition, so I'll just give a
quick background on that to make sure this post has some meaning to everyone.

Function composition is a concept most of you have probably been introduced to, in a high-school
algebra or pre-calculus course. Basically, it looks like this:

`
f(g(x)) = f ∘ g(x)
`

Where f and g are functions. This means that the return value of `g(x)` is passed into `f` as a parameter.
Let's say that `f` is a function that takes an integer and adds 2 to it, and `g` is a function that
takes an integer and subtracts 3 from it.
In a language like Haskell, you would accomplish the above like so* (this can be executed in ghci, if you like to follow along):

    let f x = x + 2
    let g x = x - 3

    (f . g) 3 == 2 -- True
    f (g 3) == 2 -- True

Behind the scenes, this is how one might implement function composition in Haskell (`\` is like Python's `lambda`):

    let f x = x + 2
    let g x = x - 3

    let compose f g = \x -> f (g x)

    (compose f g) 3 == 2 -- True
    f(g 3) == 2 -- True


Unfortunately, python does not have an operator for function composition. However, it is easy to implement a
function like we did for Haskell (in fact, here's a complete translation of the above script):

    def f(x):
        return x + 2

    def g(x):
        return x - 3

    def compose(f, g):
        return lambda x: f(g(x))

    compose(f, g) (3) == 2  # True
    f(g(3)) == 2  # True

This is pretty verbose though, especially once you start working with more than two functions,
and it would be really nice to have a good syntax for composing functions (like the dot operator in Haskell).

It turns out that it is actually a little bit possible to do this in Python. Python allows you to override a lot of operators
with dunder (or double under, like `__init__`) methods. When you do `object.key` in Python, what is actually happening is a function called `__getattr__`
is being called with the string "key", and it returns the value that it finds (usually stored in its object dictionary). My first, naive thought
was to just write a decorator class that looked like this:

    class Composable(object):

        def __init__(self, fn):
            self.fn = fn

        def __getattr__(self, other):
            return compose(self, other)

        def __call__(self, *args, **kwargs):
            return self.fn(*args, **kwargs)

This would work great, if the `other` passed into `__getattr__` were a function object. Unfortunately, it is actually a string,
storing the name of the function that is on the right hand side of the dot. This means that we need to be able to get access to the function
body from just the name, if we want dot composable functions. Luckily, we can do that with class attributes. Since only functions
decorated with the `Composable` class will be composable, we can just store the name of each decorated function in a dictionary,
with the value being the actual function object.

    class Composable(object):
        fns = {}

        def __init__(self, fn):
            self.fn = fn
            Composable.fns[fn.__name__] = fn

        def __getattr__(self, other):
            return compose(self, self.fns[other])

        def __call__(self, *args, **kwargs):
            return self.fn(*args, **kwargs)

This is basically it, actually! With this decorator class, we can take two decorated Composable functions, and with
Haskell style dot notation, compose them.

    @Composable
    def f(x):
        return x + 2

    @Composable
    def g(x):
        return x - 3

    f . g (10) == 9  # True
    Composable(lambda x: x * 2) . f . g . g (17) == 26  # True

However, this implementation has some serious limitations not shared by the Haskell version.
For example, anonymous functions (like the above lambda), can only be on the left hand side of the `.`.
In fact, the only functions that can be on the right hand side are functions that are
declared with `def` and decorated with `Composable` since any other function will not be present
in the `fns` dictionary. I could probably hack around that by playing with scope and `globals()`/`locals()`,
but that is just a little too much abuse for one day, I think.

There is an alternative implementation that actually could work, which uses one of the other operators.
I chose the `&` operator, mostly because the `|` operator would just be confusing. Here it is:

    def compose(f, g):
        return lambda x: f(g(x))

    class Composable(object):

        def __init__(self, fn):
            self.fn = fn

        def __call__(self, *args, **kwargs):
            ''' simply calls the function '''
            return self.fn(*args, **kwargs)

        def __and__(self, other):
            return Composable(compose(self.fn, other))

    (f & g) (10) == 9  # True
    (Composable(lambda x: x * 2) & f & g & g) (17) == 26  # True

This allows function composition without all the grossness of having a global dictionary of functions to lookup, and all that jazz.
It also allows non-decorated functions to be on the right hand side of the composition. On the other hand, it isn't a dot, and I
don't know of any language that has ever used the `&` operator for function composition, so it doesn't really have the intuitive benefits
that dot notation would.

Anyway, hope you enjoyed, and I'm sorry Guido!

\* I realize there are nicer ways to do this in Haskell, but since this is about Python I'll style my Haskell code to be readable
for Python developers

