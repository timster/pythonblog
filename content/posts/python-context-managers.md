+++
date = '2026-07-21T10:25:03-04:00'
draft = false
title = "Context Managers: The `with` Statement Demystified"
description = 'How the with statement actually works under the hood, how to build your own context managers, and the real-world patterns that make them worth learning.'
tags = ['python', 'intermediate', 'language-features']
+++

You've written `with open("file.txt") as f:` a thousand times. But what does `with` actually do? Why does the file close even if an exception is raised inside the block? Once you understand the protocol behind it, you can apply the same pattern to database connections, locks, timers, temporary state changes, or anything with a "setup, then guaranteed teardown" shape.

## The problem `with` solves

Before context managers, resource cleanup looked like this:

```python
f = open("file.txt")
try:
    data = f.read()
finally:
    f.close()
```

The `try`/`finally` is required because if `f.read()` raises, you still need to close the file. Forget the `finally` and you leak a file handle every time something goes wrong. This pattern is common enough (locks, connections, files, temp state) that Python built a dedicated syntax for it:

```python
with open("file.txt") as f:
    data = f.read()
```

Same guarantee, less ceremony. `f.close()` runs no matter how the block exits. Normally, via `return`, or via an exception.

## The protocol: `__enter__` and `__exit__`

`with` isn't magic; it's calling two methods that any object can implement:

```python
class ManagedFile:
    def __init__(self, filename):
        self.filename = filename

    def __enter__(self):
        self.file = open(self.filename)
        return self.file

    def __exit__(self, exc_type, exc_value, traceback):
        self.file.close()

with ManagedFile("file.txt") as f:
    data = f.read()
```

Here's the sequence:

1. `ManagedFile("file.txt")` creates the object.
2. `__enter__` runs. Its return value is bound to `f`.
3. The block body runs.
4. `__exit__` runs — *always*, even if the block raised an exception.

This is exactly equivalent to:

```python
mgr = ManagedFile("file.txt")
f = mgr.__enter__()
try:
    data = f.read()
finally:
    mgr.__exit__(None, None, None)  # simplified
```

## Handling exceptions in `__exit__`

`__exit__` receives three arguments describing any exception that occurred inside the block: `exc_type`, `exc_value`, and `traceback`. If no exception occurred, all three are `None`.

```python
class SuppressError:
    def __enter__(self):
        return self

    def __exit__(self, exc_type, exc_value, traceback):
        if exc_type is ValueError:
            print(f"Suppressed: {exc_value}")
            return True  # tells Python the exception is handled
        return False  # let other exceptions propagate

with SuppressError():
    raise ValueError("this gets swallowed")

print("execution continues here")
```

Returning `True` from `__exit__` tells Python "I've handled this, don't propagate it." Returning `False` (or `None`, the default) lets the exception continue up the call stack. This is a sharp edge — accidentally returning a truthy value from `__exit__` silently swallows real bugs.

## The easier way: `contextlib.contextmanager`

Writing a class with `__enter__`/`__exit__` is verbose for simple cases. `contextlib.contextmanager` lets you write a context manager as a generator function instead:

```python
from contextlib import contextmanager

@contextmanager
def managed_file(filename):
    f = open(filename)
    try:
        yield f
    finally:
        f.close()

with managed_file("file.txt") as f:
    data = f.read()
```

Everything before `yield` is `__enter__`. The yielded value is what gets bound by `as`. Everything after `yield` — wrapped in a `try`/`finally` — is `__exit__`. If the block raises, the exception surfaces at the `yield` line, so wrapping it in `try`/`finally` (or `try`/`except`) lets you clean up or suppress it exactly like a real `__exit__` would.

```python
@contextmanager
def suppress_value_error():
    try:
        yield
    except ValueError as e:
        print(f"Suppressed: {e}")
```

This one function replaces the entire `SuppressError` class from above.

## Real-world pattern: database transactions

Wrapping commit/rollback logic in a context manager means callers can never forget to roll back on failure:

```python
from contextlib import contextmanager

@contextmanager
def transaction(connection):
    cursor = connection.cursor()
    try:
        yield cursor
        connection.commit()
    except Exception:
        connection.rollback()
        raise
    finally:
        cursor.close()

with transaction(db_connection) as cursor:
    cursor.execute("UPDATE accounts SET balance = balance - 100 WHERE id = %s", (1,))
    cursor.execute("UPDATE accounts SET balance = balance + 100 WHERE id = %s", (2,))
```

If either `execute` call raises, the transaction rolls back automatically. If both succeed, it commits. The caller never writes `try`/`except`/`finally` themselves — the context manager owns that responsibility.

## Real-world pattern: temporarily changing state

A very common use case: change something, run code, then guarantee it's restored — regardless of what happens in between.

```python
import os
from contextlib import contextmanager

@contextmanager
def working_directory(path):
    original = os.getcwd()
    os.chdir(path)
    try:
        yield
    finally:
        os.chdir(original)

with working_directory("/tmp"):
    # any code here runs with /tmp as the cwd
    build_artifacts()
# original directory is restored, even if build_artifacts() raised
```

The same pattern applies to swapping environment variables, monkeypatching for tests, toggling feature flags, or temporarily elevating logging verbosity.

## Real-world pattern: timing and profiling

```python
import time
from contextlib import contextmanager

@contextmanager
def timer(label):
    start = time.perf_counter()
    try:
        yield
    finally:
        elapsed = time.perf_counter() - start
        print(f"{label}: {elapsed:.4f}s")

with timer("data processing"):
    process_large_dataset()
# data processing: 2.3145s
```

Cleaner than a decorator when you only want to time part of a function rather than the whole call.

## Real-world pattern: locks and concurrency

`threading.Lock`, `asyncio.Lock`, and friends are context managers for exactly this reason — you never want to acquire a lock without a guaranteed release:

```python
import threading

counter = 0
counter_lock = threading.Lock()

def increment():
    global counter
    with counter_lock:
        counter += 1
```

If the code inside the `with` block raises, the lock still releases. Without the context manager, a raised exception between `acquire()` and `release()` would deadlock every other thread waiting on that lock.

## Combining multiple context managers

You can open several context managers in one `with` statement, either with commas or (Python 3.10+) parentheses for readability:

```python
with open("input.txt") as infile, open("output.txt", "w") as outfile:
    outfile.write(infile.read().upper())

# Python 3.10+
with (
    open("input.txt") as infile,
    open("output.txt", "w") as outfile,
):
    outfile.write(infile.read().upper())
```

They nest in order — `outfile` is guaranteed to close before `infile` does, and both close even if the write fails.

## `contextlib.ExitStack` for a dynamic number of resources

Sometimes you don't know how many context managers you need until runtime — e.g., opening a variable list of files:

```python
from contextlib import ExitStack

def concatenate_files(paths, output_path):
    with ExitStack() as stack:
        files = [stack.enter_context(open(p)) for p in paths]
        with open(output_path, "w") as out:
            for f in files:
                out.write(f.read())
```

`ExitStack` tracks every context manager pushed onto it with `enter_context` and closes them all, in reverse order, when the `with` block exits — even if the list of paths is empty or built dynamically.

## Summary

| Concept | Key point |
|---|---|
| `with` statement | Guarantees cleanup runs even if an exception occurs |
| `__enter__` | Runs on entry; its return value is bound by `as` |
| `__exit__(exc_type, exc_value, tb)` | Runs on exit; return `True` to suppress the exception, `False`/`None` to propagate it |
| `@contextlib.contextmanager` | Turn a generator into a context manager — code before `yield` is `__enter__`, code after (in `try`/`finally`) is `__exit__` |
| Multiple managers | `with a, b:` nests them; both clean up even if the block fails |
| `contextlib.ExitStack` | Manage a dynamic, runtime-determined number of context managers |

The pattern to remember: anything with a "must clean up no matter what" shape — files, locks, connections, transactions, temporary state — is a candidate for a context manager.
