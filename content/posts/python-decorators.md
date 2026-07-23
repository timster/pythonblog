+++
date = '2026-07-20T10:52:46-04:00'
draft = false
title = 'Python Decorators: From Syntax Sugar to Real-World Patterns'
description = 'A deep dive into Python decorators: how they work under the hood, common pitfalls, and patterns you can use today.'
tags = ['python', 'language-features']
+++

If you've used Flask, pytest, or Django, you've used decorators. `@app.route`, `@pytest.fixture`, `@login_required`: they're everywhere. But most developers treat them as magic syntax without understanding what's actually happening. Once you do understand them, you'll start reaching for decorators in your own code.

## What a decorator actually is

A decorator is just a function that takes a function and returns a function. That's it.

```python
def my_decorator(func):
    def wrapper():
        print("before")
        func()
        print("after")
    return wrapper

def say_hello():
    print("hello")

say_hello = my_decorator(say_hello)
say_hello()
# before
# hello
# after
```

The `@` syntax is shorthand for exactly that reassignment. This:

```python
@my_decorator
def say_hello():
    print("hello")
```

is 100% equivalent to writing `say_hello = my_decorator(say_hello)` after the function definition.

## Handling arguments with `*args` and `**kwargs`

The wrapper above only works for functions with no arguments. Real decorators need to pass arguments through:

```python
def my_decorator(func):
    def wrapper(*args, **kwargs):
        print("before")
        result = func(*args, **kwargs)
        print("after")
        return result
    return wrapper

@my_decorator
def add(a, b):
    return a + b

add(1, 2)  # works, returns 3
```

`*args` and `**kwargs` capture everything and forward it unchanged; this is the standard pattern for any decorator that doesn't need to inspect the arguments.

## The `functools.wraps` problem

Here's something that bites people in production:

```python
@my_decorator
def add(a, b):
    """Add two numbers."""
    return a + b

print(add.__name__)   # 'wrapper'  ← wrong
print(add.__doc__)    # None       ← wrong
```

Because `add` now points to `wrapper`, it has `wrapper`'s name and docstring. This breaks introspection, documentation tools, and anything that uses `__name__` (like some logging setups).

Fix it with `functools.wraps`:

```python
import functools

def my_decorator(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        result = func(*args, **kwargs)
        return result
    return wrapper

@my_decorator
def add(a, b):
    """Add two numbers."""
    return a + b

print(add.__name__)  # 'add'
print(add.__doc__)   # 'Add two numbers.'
```

Always use `@functools.wraps`. There's no reason not to.

## Decorators with arguments

What if you want to configure the decorator itself? `@app.route("/home")` passes an argument to the decorator: how does that work?

You need an extra layer of nesting: a function that *returns* a decorator.

```python
import functools

def repeat(n):
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            for _ in range(n):
                result = func(*args, **kwargs)
            return result
        return wrapper
    return decorator

@repeat(3)
def greet(name):
    print(f"Hello, {name}!")

greet("Alice")
# Hello, Alice!
# Hello, Alice!
# Hello, Alice!
```

`@repeat(3)` calls `repeat(3)` first, which returns `decorator`, which then wraps `greet`. Three levels deep, but each level has one job.

## Real-world pattern: timing

A timing decorator is one of the most useful tools for quick performance profiling without reaching for a full profiler:

```python
import functools
import time

def timed(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
        result = func(*args, **kwargs)
        elapsed = time.perf_counter() - start
        print(f"{func.__name__} took {elapsed:.4f}s")
        return result
    return wrapper

@timed
def slow_query(n):
    time.sleep(n)
    return n

slow_query(0.5)
# slow_query took 0.5001s
```

Slap `@timed` on any function and you have timing with zero boilerplate.

## Real-world pattern: retry

Network calls fail. Rather than writing retry logic in every function, wrap it once:

```python
import functools
import time

def retry(times=3, delay=1.0, exceptions=(Exception,)):
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(1, times + 1):
                try:
                    return func(*args, **kwargs)
                except exceptions as e:
                    if attempt == times:
                        raise
                    print(f"Attempt {attempt} failed: {e}. Retrying in {delay}s...")
                    time.sleep(delay)
        return wrapper
    return decorator

@retry(times=3, delay=0.5, exceptions=(ConnectionError,))
def fetch_data(url):
    # ... make HTTP request
    pass
```

Now `fetch_data` automatically retries on `ConnectionError` up to 3 times. The calling code never needs to know.

## Real-world pattern: simple caching

A basic memoization decorator for pure functions:

```python
import functools

def memoize(func):
    cache = {}

    @functools.wraps(func)
    def wrapper(*args):
        if args not in cache:
            cache[args] = func(*args)
        return cache[args]

    return wrapper

@memoize
def fibonacci(n):
    if n < 2:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)

fibonacci(35)  # fast
```

Note: Python's standard library has `@functools.lru_cache` and `@functools.cache` which do this better (thread-safe, bounded size, support for `maxsize`). Use those in real code, but understanding how to build one yourself is worthwhile.

## Class-based decorators

A decorator can also be a class that implements `__call__`. This is useful when you need to maintain more state:

```python
import functools

class CountCalls:
    def __init__(self, func):
        functools.update_wrapper(self, func)
        self.func = func
        self.count = 0

    def __call__(self, *args, **kwargs):
        self.count += 1
        print(f"Call #{self.count} to {self.func.__name__}")
        return self.func(*args, **kwargs)

@CountCalls
def greet(name):
    print(f"Hello, {name}!")

greet("Alice")  # Call #1 to greet
greet("Bob")    # Call #2 to greet
print(greet.count)  # 2
```

The instance replaces the function, so `greet.count` works naturally.

## Stacking decorators

You can apply multiple decorators to one function: they apply bottom-up.

```python
@timed
@retry(times=3)
def fetch():
    ...
```

This is equivalent to `fetch = timed(retry(times=3)(fetch))`. The `retry` decorator wraps `fetch` first, then `timed` wraps that. So the timer measures the total time including retries, not just one attempt.

Order matters: swap them and you'd time a single attempt, with the retry happening invisibly outside the timer.

## When to reach for a decorator

Decorators shine when you have behavior that:

- **Wraps** a function without changing what it does (logging, timing, auth)
- **Applies to many functions** across your codebase
- **Shouldn't pollute the function's body** with cross-cutting concerns

If you find yourself copying the same setup/teardown block into ten functions, that's a decorator waiting to be written.

Where they're less appropriate: when the logic is so specific to one function that abstracting it adds complexity without removing duplication, or when you need access to local variables in a way that breaks the wrapper pattern cleanly.

## Summary

| Concept | Key point |
|---|---|
| Basic decorator | A function that takes and returns a function |
| `*args, **kwargs` | Forward arguments through the wrapper |
| `@functools.wraps` | Preserve `__name__`, `__doc__`, and other metadata; always use it |
| Decorator with args | Add an outer function that returns the decorator |
| Stacking | Applies bottom-up; order matters |

The pattern is always the same: wrap a function, add behavior, return the wrapper. Once that clicks, the rest is just applying the pattern to different problems.
