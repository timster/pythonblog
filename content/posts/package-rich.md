+++
date = '2026-07-23T09:20:30-04:00'
draft = false
title = 'Rich: Beautiful Terminal Output in Python'
description = 'A tour of the Rich library: what it does, how to install it, its killer feature, and real-world patterns for logging, progress bars, and tables in production CLIs.'
tags = ['python', 'packages', 'cli']
+++

Most Python programs still talk to their users through `print()`. That's fine until you need a progress bar, a table, syntax-highlighted tracebacks, or just readable output that doesn't run together into a wall of text. `rich` replaces `print()` with something that understands color, layout, and structure, and it does it without asking you to learn a templating language first.

## What it is

[Rich](https://github.com/Textualize/rich) is a Python library for rendering formatted text, tables, progress bars, syntax-highlighted code, markdown, and tracebacks to the terminal. It detects what your terminal supports (truecolor, 256-color, or plain) and degrades gracefully, so the same code produces reasonable output whether it's running in a modern terminal, a CI log, or a dumb pipe.

It's maintained by the same author as Textual (the terminal UI framework), and it has become a de facto standard: `pip`, `pytest` plugins, and countless CLI tools either use Rich directly or use libraries (like Typer) that use it under the hood.

## Install

```bash
pip install rich
```

No extra dependencies to worry about, and it works cross-platform, including Windows terminals, without extra configuration.

## Killer feature: the `Console` object and rich tracebacks

The single highest-leverage thing Rich does is replace `print` with a `Console` that understands markup, and replace Python's default traceback with one that's actually readable:

```python
from rich.console import Console

console = Console()
console.print("Hello", "World", style="bold magenta")
console.print("[bold red]Error:[/bold red] connection refused")
```

The bracket syntax (`[bold red]...[/bold red]`) is Rich's markup language: inline styling without manually managing ANSI escape codes. It supports color names, hex codes, styles like `bold`/`italic`/`underline`, and combinations like `[bold white on red]`.

Even bigger for day-to-day debugging: swap in Rich's traceback handler once, globally, and every unhandled exception in your program gets syntax highlighting, local variable inspection, and clearer frame separation:

```python
from rich.traceback import install
install(show_locals=True)

def divide(a, b):
    return a / b

divide(1, 0)
```

Instead of Python's default plain-text traceback, you get color-coded source context and the actual values of `a` and `b` at the point of failure. For anything you run interactively or debug from CI logs, this alone is worth the install.

## Real-world pattern: structured logging with `RichHandler`

Rich plugs directly into the standard library's `logging` module, replacing plain log lines with colorized, aligned output that highlights log levels, timestamps, and tracebacks:

```python
import logging
from rich.logging import RichHandler

logging.basicConfig(
    level="INFO",
    format="%(message)s",
    datefmt="[%X]",
    handlers=[RichHandler(rich_tracebacks=True)],
)

log = logging.getLogger("app")

log.info("Starting job %s", "nightly-sync")
log.warning("Retrying after timeout")
log.error("Job failed", exc_info=True)
```

Your existing `logging.getLogger()` calls don't change at all. You only swap the handler, so this is a drop-in upgrade for any codebase already using the standard logging module, no need to rewrite call sites.

## Real-world pattern: progress bars for long-running jobs

Progress bars are the thing most people reach for Rich first, and the API scales from "one bar" to "many concurrent bars with different columns" without much extra code:

```python
from rich.progress import Progress

files = ["report.csv", "users.csv", "orders.csv"]

with Progress() as progress:
    task = progress.add_task("[cyan]Processing files...", total=len(files))
    for filename in files:
        process_file(filename)
        progress.update(task, advance=1)
```

For a batch job with multiple stages (download, then parse, then upload) you can track each stage as its own task and they render as stacked bars:

```python
from rich.progress import Progress

with Progress() as progress:
    download_task = progress.add_task("[green]Downloading...", total=100)
    parse_task = progress.add_task("[yellow]Parsing...", total=100)

    for i in range(100):
        progress.update(download_task, advance=1)
        if i > 20:
            progress.update(parse_task, advance=1.25)
```

Because `Progress` is itself a context manager, cleanup (stopping the live display, leaving the terminal in a sane state) is guaranteed even if the job raises partway through.

## Real-world pattern: tables for CLI reports

Formatting tabular data with `print` and manual `str.ljust()` calls falls apart the moment column widths vary. Rich's `Table` handles alignment, wrapping, and styling for you:

```python
from rich.console import Console
from rich.table import Table

console = Console()
table = Table(title="Deployment Status")

table.add_column("Service", style="cyan")
table.add_column("Version", style="magenta")
table.add_column("Status", justify="right")

table.add_row("api", "2.4.1", "[green]healthy[/green]")
table.add_row("worker", "2.4.0", "[yellow]degraded[/yellow]")
table.add_row("scheduler", "2.3.9", "[red]down[/red]")

console.print(table)
```

This is a common pattern for internal ops tooling: a script that checks the state of several services and reports it as a clean table instead of a wall of print statements, directly readable by whoever is on call.

## Real-world pattern: pretty-printing data structures

`console.print()` understands Python objects out of the box, so debugging nested dicts and lists doesn't require `pprint` or manual formatting:

```python
from rich import print as rprint

data = {
    "user": "alice",
    "roles": ["admin", "editor"],
    "settings": {"theme": "dark", "notifications": True},
}

rprint(data)
```

Rich auto-detects the structure and prints it with syntax highlighting and indentation, similar to `pprint` but with color. For quick debugging, `rich.inspect()` goes further and dumps an object's attributes, methods, and docstrings:

```python
from rich import inspect

inspect(data, methods=True)
```

## When not to use it

Rich assumes it's writing to a terminal (or something emulating one). A few situations where it's the wrong tool:

- **Output consumed by other programs.** If a script's stdout is meant to be parsed (JSON, CSV, piped into `jq`), don't route that data through Rich; save Rich for the human-facing diagnostic output and keep machine-readable output plain.
- **Extremely resource-constrained environments.** Rich is lightweight compared to a full TUI framework, but if you're writing a script that needs to start in single-digit milliseconds (a shell completion hook, for example), the import cost may not be worth it.
- **Logging destined purely for log aggregation systems.** If your logs go straight into something like Datadog or Elasticsearch as structured JSON, colorized terminal formatting adds nothing since no human ever reads the raw line.

For anything else, a CLI tool, a build script, a data pipeline with progress reporting, an admin script your team runs by hand, Rich is close to a strict upgrade over `print()`.

## Summary

| Concept | Key point |
|---|---|
| `Console` | Drop-in replacement for `print()` with markup, color, and terminal-aware rendering |
| Markup syntax | `[bold red]...[/bold red]` for inline styling without manual ANSI codes |
| `rich.traceback.install()` | Global upgrade to readable, syntax-highlighted tracebacks with local variables |
| `RichHandler` | Plugs into standard library `logging` for colorized, structured log output |
| `Progress` | Single or multi-task progress bars for long-running jobs |
| `Table` | Aligned, styled tabular output for CLI reports |
| `rprint` / `inspect` | Pretty-print data structures and introspect objects during debugging |
| When to skip it | Machine-parsed stdout, ultra-lightweight scripts, or logs that only ever go to aggregation systems |
