+++
date = '2026-07-24T09:00:00-04:00'
draft = true
title = 'Python Pattern Matching: Beyond the Basic match Statement'
description = 'A deep dive into structural pattern matching in Python: match/case syntax, sequence, mapping, and class patterns, guard clauses, and real-world uses for dispatching, payload routing, and AST-style processing.'
tags = ['python', 'pattern-matching', 'match-case', 'control-flow']
+++

Python 3.10 added the `match` statement, and a lot of people took one look at it and filed it away as "Python finally got a switch statement." That's a disservice to what it actually does. A `switch` statement compares a value against other values. `match` compares the *shape* of data against a pattern, binds variables as it goes, and can reach inside nested structures in a single expression. It's closer to pattern matching in Rust or Haskell than to `switch` in C.

This article walks through the full pattern vocabulary, from literals to nested class patterns, and then shows where it earns its place in real code.

## Basic syntax

The `match` statement takes a subject expression and compares it against a series of `case` patterns, top to bottom, executing the first one that matches:

```python
def describe(value):
    match value:
        case 0:
            return "zero"
        case 1:
            return "one"
        case _:
            return "something else"
```

The `_` wildcard matches anything and binds nothing. It's conventionally the last case, playing the role `default` plays in a switch statement, but it's also legal to use it as a placeholder inside other patterns (more on that in the sequence and class sections).

Unlike `if`/`elif` chains, patterns are declarative: you describe what the data should look like, not the comparison logic needed to check it.

## Capture patterns and OR patterns

A bare name in a pattern isn't a comparison, it's a capture. It always matches, and it binds the subject to that name:

```python
def handle(command):
    match command:
        case "start":
            return "starting"
        case "stop":
            return "stopping"
        case other:
            return f"unknown command: {other}"
```

`other` here works like `_`, except it also binds the value so you can use it. This is a common source of bugs for people new to pattern matching: `case other` always matches, even before you've handled every literal you meant to. Order matters, and literal cases must come before an unconditional capture.

The `|` operator lets a single case match multiple patterns:

```python
def classify(status_code):
    match status_code:
        case 200 | 201 | 204:
            return "success"
        case 400 | 401 | 403 | 404:
            return "client error"
        case 500 | 502 | 503:
            return "server error"
        case _:
            return "unknown"
```

Each alternative in an OR pattern must bind the same set of names, since Python needs to know what's available regardless of which branch matched.

## Guard clauses

A case can carry an `if` condition, evaluated only if the pattern itself matches. This is where captured values become genuinely useful:

```python
def categorize(n):
    match n:
        case int(x) if x < 0:
            return "negative integer"
        case int(x) if x == 0:
            return "zero"
        case int(x):
            return "positive integer"
        case _:
            return "not an integer"
```

The guard runs after the structural match succeeds, so `int(x) if x < 0` first confirms `n` is an `int`, then binds it to `x`, then checks the condition. If the guard fails, matching continues to the next `case` rather than raising an error.

## Sequence patterns

Sequence patterns match lists, tuples, and other sequences by shape, and can unpack elements as they go:

```python
def handle_command(parts):
    match parts:
        case []:
            return "no command"
        case [command]:
            return f"running {command} with no args"
        case [command, arg]:
            return f"running {command} with arg {arg}"
        case [command, *args]:
            return f"running {command} with args {args}"
```

`[command, *args]` works like starred unpacking in an assignment: `command` captures the first element, `args` captures the rest as a list. You can also anchor `*rest` in the middle or use `*_` when you want to match "at least N items" without capturing the overflow:

```python
match parts:
    case [first, *_, last]:
        return f"first={first}, last={last}"
```

Sequence patterns match both lists and tuples (anything that isn't a `str`, `bytes`, or `bytearray`, which are deliberately excluded so `"ab"` doesn't accidentally match `[a, b]`).

## Mapping patterns

Mapping patterns match dicts (and other mapping types) by key, and unlike sequence patterns, they match on a *subset* of keys by default:

```python
def handle_event(event):
    match event:
        case {"type": "click", "x": x, "y": y}:
            return f"click at ({x}, {y})"
        case {"type": "keypress", "key": key}:
            return f"key pressed: {key}"
        case {"type": event_type}:
            return f"unhandled event: {event_type}"
```

A dict with extra keys beyond `type`, `x`, and `y` still matches the first case; mapping patterns only check that the listed keys are present with matching values, not that the dict contains nothing else. If you want to capture whatever's left over, use `**rest`:

```python
match event:
    case {"type": "click", "x": x, "y": y, **rest}:
        return x, y, rest
```

Note that `**rest` cannot be combined with `**_`, and unlike sequence patterns there's no way to assert "exactly these keys and no others" without manually checking `len()` in a guard.

## Class patterns

Class patterns match on `isinstance` plus attribute values, which makes them useful with both built-in types and your own classes:

```python
from dataclasses import dataclass

@dataclass
class Point:
    x: int
    y: int

def describe_point(point):
    match point:
        case Point(x=0, y=0):
            return "origin"
        case Point(x=0, y=y):
            return f"on the y-axis at {y}"
        case Point(x=x, y=0):
            return f"on the x-axis at {x}"
        case Point(x=x, y=y):
            return f"point at ({x}, {y})"
        case _:
            return "not a point"
```

`dataclass` fields work as positional patterns too, because dataclasses automatically define `__match_args__`:

```python
match point:
    case Point(0, 0):
        return "origin"
    case Point(x, y):
        return f"point at ({x}, {y})"
```

For a plain (non-dataclass) class, you'd define `__match_args__ = ("x", "y")` yourself to enable positional matching. Keyword-style matching (`Point(x=0, y=0)`) always works for any class, dataclass or not, as long as the attributes exist.

## Nested patterns

The real payoff of structural pattern matching is combining sequence, mapping, and class patterns to describe deeply nested shapes in one readable expression:

```python
def process(payload):
    match payload:
        case {"event": "order_created", "order": {"id": order_id, "items": [*items]}}:
            return f"order {order_id} created with {len(items)} items"
        case {"event": "order_created", "order": {"id": order_id}}:
            return f"order {order_id} created, no items listed"
        case {"event": "user_updated", "user": {"id": user_id, "changes": {**changes}}}:
            return f"user {user_id} updated: {list(changes)}"
        case _:
            return "unrecognized payload"
```

Compare that to the equivalent with `if`/`elif` and manual `.get()` calls and nested `isinstance` checks. It's not just fewer lines, the structure of the pattern mirrors the structure of the data, so the code reads like documentation of the shapes it accepts.

## Real-world pattern: command dispatcher

Anywhere you'd otherwise write a long `if command == "x": ... elif command == "y": ...` chain, a `match` on parsed input is a direct, more readable replacement:

```python
def run_command(raw_input):
    match raw_input.split():
        case ["add", *items] if items:
            return f"adding: {', '.join(items)}"
        case ["remove", item]:
            return f"removing: {item}"
        case ["list"]:
            return "listing all items"
        case ["help"] | []:
            return "usage: add|remove|list|help"
        case [unknown, *_]:
            return f"unknown command: {unknown}"

print(run_command("add milk eggs bread"))
print(run_command("remove milk"))
print(run_command(""))
```

The guard on `["add", *items] if items` rejects a bare `"add"` with no arguments, falling through to the `unknown command` case instead of silently succeeding with an empty list. This kind of small-CLI dispatcher is one of the most common places `match` shows up in real code.

## Real-world pattern: API/webhook payload routing

Webhook handlers (Stripe, GitHub, Slack, and similar) typically deliver a JSON payload whose shape depends on an event type field. Pattern matching lets you route on that shape directly instead of writing a dispatch dict plus a pile of manual key lookups:

```python
def handle_webhook(payload):
    match payload:
        case {"type": "payment_intent.succeeded", "data": {"object": {"id": pid, "amount": amount}}}:
            record_payment(pid, amount)
        case {"type": "payment_intent.payment_failed", "data": {"object": {"id": pid, "last_payment_error": {"message": message}}}}:
            log_payment_failure(pid, message)
        case {"type": "customer.subscription.deleted", "data": {"object": {"id": sub_id}}}:
            cancel_subscription(sub_id)
        case {"type": event_type}:
            log_unhandled_event(event_type)
        case _:
            raise ValueError("malformed webhook payload")
```

Each case both validates and destructures the payload in one step. If `data.object.id` isn't present, the pattern simply doesn't match and falls through, rather than raising a `KeyError` partway through handling.

## Real-world pattern: AST/token processing

Class patterns are a natural fit for walking small tree structures, whether that's Python's own `ast` module or a hand-rolled node hierarchy for a parser or interpreter:

```python
from dataclasses import dataclass
from typing import Union

@dataclass
class Num:
    value: float

@dataclass
class Add:
    left: "Expr"
    right: "Expr"

@dataclass
class Mul:
    left: "Expr"
    right: "Expr"

Expr = Union[Num, Add, Mul]

def evaluate(expr: Expr) -> float:
    match expr:
        case Num(value):
            return value
        case Add(left, right):
            return evaluate(left) + evaluate(right)
        case Mul(left, right):
            return evaluate(left) * evaluate(right)
        case _:
            raise TypeError(f"unknown expression: {expr!r}")

tree = Add(Num(2), Mul(Num(3), Num(4)))
print(evaluate(tree))  # 14
```

This is essentially how a small tree-walking interpreter is structured in languages built around pattern matching, and it translates cleanly to Python. Each node type gets its own `case`, recursion handles the nesting, and adding a new node type (say, `Sub`) is a matter of adding one more `case` rather than editing a chain of `isinstance` checks.

## Gotchas

A few things trip people up the first time they use `match`:

- **Dotted names are values, bare names are captures.** `case Color.RED:` compares against the enum member `Color.RED`. But `case RED:` (no dot) is a capture pattern; it matches anything and binds it to `RED`, which is almost never what you want. If you need to match a variable holding a literal (not a dotted attribute), you have to use a guard: `case x if x == RED`.
- **Exhaustiveness isn't checked.** Unlike Rust's `match`, Python won't warn you if your `case`s don't cover every possibility. Without a final `case _:`, a subject that matches nothing simply falls through the whole statement with no error and no effect. If you want a guaranteed error for unhandled cases, add an explicit `case _: raise ValueError(...)`.
- **Case order matters, and unconditional patterns can shadow later ones.** A capture pattern or bare `_` before a more specific `case` will always win, since matching stops at the first success. Put literal and structural cases before catch-alls.
- **Sequence patterns don't match strings by "characters."** `case [a, b]` won't match `"ab"`, because `str` is explicitly excluded from sequence pattern matching. This is intentional, to avoid the common footgun of a string being silently treated as a sequence of one-character strings.

## Summary

| Pattern type | Syntax example | Use case |
|---|---|---|
| Literal | `case 200:` | Exact value comparison |
| Capture | `case x:` | Bind subject to a name, always matches |
| Wildcard | `case _:` | Match anything, bind nothing (default case) |
| OR pattern | `case 400 \| 404:` | Match any of several alternatives |
| Guard | `case x if x > 0:` | Add a runtime condition after structural match |
| Sequence | `case [first, *rest]:` | Match and unpack lists/tuples by shape |
| Mapping | `case {"key": value}:` | Match dicts by key, ignoring extra keys |
| Class | `case Point(x=0, y=0):` | Match by type and attribute values |
| Nested | `case {"data": {"id": id}}:` | Combine pattern types for structured data |

Pattern matching doesn't replace `if`/`elif` everywhere, but for code whose job is "figure out the shape of this data and act accordingly," a command parser, an event router, a tree-walking interpreter, it reads more like a specification of the accepted shapes than a sequence of manual checks.
