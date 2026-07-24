+++
date = '2026-07-24T11:13:14-04:00'
draft = false
title = 'The Most Underrated Packages in the Python Standard Library'
description = "A tour of stdlib modules that quietly solve real problems - bisect, functools.singledispatch, contextlib.ExitStack, graphlib, itertools, and more - with production-ready patterns for each."
tags = ['python', 'packages', 'best-practices']
+++

Every Python developer knows `os`, `json`, and `collections`. Far fewer reach for `bisect` when they need a sorted insert, or `graphlib` when they need a dependency order, and end up pulling in a third-party package (or writing worse code by hand) for something the standard library already does well. This is a tour of the modules that don't get enough credit: what they do, and how they show up in real code.

## `bisect`: sorted containers without a library

`bisect` maintains a list in sorted order using binary search, so insertions and lookups are `O(log n)` instead of the `O(n)` you get from scanning a plain list.

```python
import bisect

scores = [10, 25, 40, 60]
bisect.insort(scores, 35)
print(scores)  # [10, 25, 35, 40, 60]
```

`bisect_left` and `bisect_right` find insertion points without actually inserting, which makes them handy for range queries and bucketing.

### Real-world pattern: grading buckets

A classic use is mapping a continuous value to a discrete tier without a chain of `if/elif`:

```python
import bisect

breakpoints = [60, 70, 80, 90]
grades = ["F", "D", "C", "B", "A"]

def grade(score):
    return grades[bisect.bisect(breakpoints, score)]

print(grade(85))  # "B"
print(grade(59))  # "F"
```

This same pattern works for rate limit tiers, log-level thresholds, or any "which bucket does this number fall into" problem, and it stays readable even as the number of buckets grows.

## `functools.singledispatch`: type-based branching without `isinstance` chains

`singledispatch` turns a function into a generic function that dispatches on the type of its first argument, replacing a ladder of `isinstance` checks with separate, independently testable implementations.

```python
from functools import singledispatch

@singledispatch
def to_json(value):
    raise TypeError(f"Unsupported type: {type(value)}")

@to_json.register
def _(value: dict):
    return {k: to_json(v) for k, v in value.items()}

@to_json.register
def _(value: list):
    return [to_json(v) for v in value]

@to_json.register
def _(value: str):
    return value
```

### Real-world pattern: serializers that grow without touching old code

The payoff shows up when new types arrive later. Instead of editing a central `if isinstance(x, Foo)` block (and risking a merge conflict with every other type that touches it), a new module just registers its own handler:

```python
from datetime import date

@to_json.register
def _(value: date):
    return value.isoformat()
```

This is the same shape you'll find in serialization libraries, visitor patterns for ASTs, and pretty-printers: each type owns its own conversion logic instead of one function knowing about every type in the system.

## `contextlib`: more than `@contextmanager`

Most people know `contextlib.contextmanager` for turning a generator into a context manager (see my [context managers article]({{< ref "python-context-managers" >}}) for a full walkthrough of that pattern). Two less-visited tools in the same module solve problems that otherwise turn into boilerplate.

`suppress` replaces a `try/except/pass` block that exists purely to ignore an expected exception:

```python
from contextlib import suppress

with suppress(FileNotFoundError):
    os.remove("cache.tmp")
```

### Real-world pattern: `ExitStack` for a dynamic number of resources

`ExitStack` manages a variable-length stack of context managers, which matters when you don't know how many resources you're opening until runtime:

```python
from contextlib import ExitStack

def merge_files(paths, output_path):
    with ExitStack() as stack, open(output_path, "w") as out:
        files = [stack.enter_context(open(p)) for p in paths]
        for file in files:
            out.writelines(file)
```

Every file opened via `stack.enter_context` is guaranteed to be closed, in reverse order, even if one of the middle files fails to open or an exception is raised partway through the loop. Writing that guarantee by hand with nested `with` statements or manual `try/finally` bookkeeping gets ugly fast once the count isn't fixed.

## `graphlib.TopologicalSorter`: dependency ordering, built in

Since Python 3.9, `graphlib` ships a topological sorter, so tasks that depend on other tasks completing first no longer need a hand-rolled Kahn's algorithm or a third-party DAG library.

```python
from graphlib import TopologicalSorter

graph = {
    "deploy": {"build", "test"},
    "test": {"build"},
    "build": {"install_deps"},
    "install_deps": set(),
}

ts = TopologicalSorter(graph)
print(list(ts.static_order()))
# ['install_deps', 'build', 'test', 'deploy']
```

### Real-world pattern: running independent tasks concurrently, respecting dependencies

`TopologicalSorter` also supports incremental use, which is exactly what a task runner needs: pull out everything that's currently ready, run it (possibly in parallel), and mark it done.

```python
from graphlib import TopologicalSorter

def run_pipeline(graph, run_task):
    ts = TopologicalSorter(graph)
    ts.prepare()
    while ts.is_active():
        ready = ts.get_ready()
        for task in ready:
            run_task(task)
            ts.done(task)
```

This is the core loop behind build systems and CI pipeline runners: `install_deps` and any unrelated task with no shared dependency can run at the same time, while `deploy` waits until both `build` and `test` report done.

## `itertools`: the toolbox for iterator plumbing

`itertools` rarely gets reached for until someone hits the exact wall it solves. Three functions pull their weight constantly:

- `groupby` groups consecutive items by a key (it requires the input to already be sorted by that key).
- `chain` flattens multiple iterables into one without building an intermediate list.
- `islice` slices a lazy iterator without materializing it.

### Real-world pattern: grouping log lines by request ID

```python
from itertools import groupby

logs = sorted(read_logs(), key=lambda entry: entry.request_id)

for request_id, entries in groupby(logs, key=lambda entry: entry.request_id):
    entries = list(entries)
    print(f"{request_id}: {len(entries)} log lines")
```

### Real-world pattern: paginating an API without loading everything into memory

```python
from itertools import islice

def paginate(iterable, page_size):
    iterator = iter(iterable)
    while page := list(islice(iterator, page_size)):
        yield page

for page in paginate(fetch_all_records(), page_size=100):
    send_batch(page)
```

`fetch_all_records()` can be a generator pulling from a database cursor or an API with pagination tokens, and `islice` only pulls as many items as the current page needs, so memory use stays flat regardless of the total record count.

## `secrets`: the module `random` should never be used for

`random` is a Mersenne Twister: fast, deterministic given a seed, and completely unsuitable for anything security-sensitive because its output is predictable if an attacker observes enough of it. `secrets` wraps the OS's cryptographically secure random source with a small, purpose-built API.

```python
import secrets

token = secrets.token_urlsafe(32)   # session tokens, API keys
otp = secrets.randbelow(1_000_000)  # numeric one-time codes
```

### Real-world pattern: generating password reset tokens

```python
import secrets
from datetime import datetime, timedelta

def create_reset_token(user_id, store):
    token = secrets.token_urlsafe(32)
    store.save(token, user_id, expires=datetime.utcnow() + timedelta(hours=1))
    return token
```

`secrets.compare_digest` is the other half of this: comparing a submitted token against the stored one with `==` leaks timing information an attacker can use to guess the value byte by byte. `secrets.compare_digest` runs in constant time regardless of where the strings first differ.

```python
import secrets

def verify_token(submitted, expected):
    return secrets.compare_digest(submitted, expected)
```

## `difflib`: readable diffs and fuzzy matching without a dependency

`difflib` computes differences between sequences and does approximate string matching, both of which usually get outsourced to a package before anyone checks whether the standard library already covers it.

```python
import difflib

before = ["def foo():", "    return 1"]
after = ["def foo():", "    return 2"]

diff = difflib.unified_diff(before, after, lineterm="")
print("\n".join(diff))
```

### Real-world pattern: "did you mean" suggestions for CLI tools

`get_close_matches` does fuzzy matching against a list of valid options, which is exactly what powers the "did you mean" suggestion in tools like `pip` and `git` when a command is mistyped.

```python
import difflib

def suggest_command(user_input, valid_commands):
    matches = difflib.get_close_matches(user_input, valid_commands, n=1, cutoff=0.6)
    return matches[0] if matches else None

commands = ["status", "commit", "checkout", "branch"]
print(suggest_command("chekout", commands))  # "checkout"
```

This turns "unknown command" into "did you mean `checkout`?" with two lines of code and zero dependencies.

## When to reach for a third-party package instead

None of this is an argument to avoid PyPI. A few signals that the stdlib module is the wrong tool:

- **`itertools`/`bisect` for genuinely large, mutating sorted data.** If you're doing thousands of sorted inserts on large sequences, a proper sorted-container library (like `sortedcontainers`) or a database index will outperform a Python-level `bisect.insort`, which is still `O(n)` for the insert itself even though the search is `O(log n)`.
- **`difflib` for large-scale or binary diffing.** It's fine for readable text diffs and fuzzy string matches; for anything approaching the scale or format complexity of `git diff`, use a dedicated diffing tool.
- **`secrets` for anything beyond tokens and OTPs.** It's not a general cryptography library; for hashing, signing, or encryption, reach for `hashlib`/`hmac` or a library like `cryptography`.

For everyday problems, sorted inserts, type-based dispatch, variable-length resource cleanup, dependency ordering, iterator plumbing, secure tokens, and fuzzy matching, the standard library modules above are usually a better first move than an import from PyPI.

None of these modules are hidden. They're one `import` away, fully documented, and battle-tested by every Python installation on the planet. The only thing standing between you and them is the habit of checking the stdlib before reaching for `pip install`. Next time you're about to write a binary search by hand or add a dependency for a topological sort, take a second look at what's already sitting in your Python installation.
