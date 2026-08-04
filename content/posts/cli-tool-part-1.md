+++
date = '2026-08-04T08:38:50-04:00'
draft = false
title = 'Building a CLI Tool in Python: Structure & Typer'
description = 'Part 1 of a series on building a real Python CLI tool: project layout, argument parsing with Typer, subcommands, and the structure that keeps a growing tool maintainable.'
tags = ['python', 'cli', 'tooling']
+++

Every Python developer eventually writes a script that starts as a single `if __name__ == "__main__":` block and grows into something that needs subcommands, flags, help text, and configuration. The difference between a script and a real CLI tool is structure: how you parse arguments, how you organize commands, and how you keep the whole thing testable as it grows.

This is part one of a three-part series where we build a real CLI tool from scratch. By the end of this series you'll have a tool that's structured cleanly, reads configuration files, and is published to PyPI so anyone can `pip install` it. This part covers project structure and argument parsing with [Typer](https://typer.tiangolo.com/).

## The tool we're building

Across this series we'll build `taskcli`, a small command-line task tracker. It's simple enough to build in three articles but realistic enough to demonstrate real patterns:

- `taskcli add "buy milk"` adds a task
- `taskcli list` shows open tasks
- `taskcli done 3` marks task 3 complete
- `taskcli remove 3` deletes a task

Part 1 (this article) covers structure and command parsing. Part 2 covers config files and packaging. Part 3 covers publishing to PyPI.

## Why not argparse?

The standard library's `argparse` works, and it's fine for a single-command script. But once you need subcommands (`taskcli add`, `taskcli list`, `taskcli done`), argparse's subparser API gets verbose fast, and you lose type safety: every argument arrives as a string or whatever type you manually specified, with no connection to your function signatures.

[Typer](https://typer.tiangolo.com/) (built by the same author as FastAPI) builds a CLI from your function signatures using type hints. It gives you:

- Subcommands as plain functions
- Automatic `--help` generation from docstrings and type hints
- Type validation and coercion for free (an `int` parameter rejects non-numeric input automatically)
- Built on [Click](https://click.palletsprojects.com/) under the hood, so it's production-tested

## Install

```bash
pip install typer
```

Typer's `rich` integration (for prettier help output and tracebacks) is included by default in modern versions. If you want the leaner install, `pip install "typer-slim"` skips those extras.

## Project structure

Resist the urge to put everything in one file. Even for a small tool, a package layout pays off the moment you add a second command:

```
taskcli/
├── pyproject.toml
├── src/
│   └── taskcli/
│       ├── __init__.py
│       ├── cli.py          # Typer app and command definitions
│       ├── storage.py       # task persistence
│       └── models.py        # data structures
└── tests/
    └── test_cli.py
```

The `src/` layout (rather than putting `taskcli/` directly at the repo root) prevents a common footgun: without it, `import taskcli` during testing can accidentally succeed by importing from your working directory instead of the installed package, hiding packaging bugs until you actually publish. We'll set up `pyproject.toml` properly in part 2; for now, focus on `cli.py`.

## A minimal Typer app

```python
# src/taskcli/cli.py
import typer

app = typer.Typer(help="A simple command-line task tracker.")


@app.command()
def add(text: str) -> None:
    """Add a new task."""
    typer.echo(f"Added task: {text}")


@app.command()
def list() -> None:
    """List all open tasks."""
    typer.echo("No tasks yet.")


if __name__ == "__main__":
    app()
```

Run it directly during development:

```bash
python -m taskcli.cli add "buy milk"
# Added task: buy milk

python -m taskcli.cli --help
```

Typer generates a full `--help` screen, including per-command help, from the docstrings and function signatures alone. No separate description strings to keep in sync.

## Arguments vs. options

Typer distinguishes two kinds of parameters, matching the underlying Click concepts:

- **Arguments** are positional and (usually) required: `taskcli done 3` where `3` is a `Argument`.
- **Options** are named flags like `--priority high` or boolean switches like `--verbose`.

By default, a plain parameter becomes a positional argument. Use `typer.Option` to make it a named flag:

```python
from typing import Optional
import typer

app = typer.Typer()


@app.command()
def add(
    text: str,
    priority: str = typer.Option("normal", "--priority", "-p", help="low, normal, or high"),
    tags: Optional[list[str]] = typer.Option(None, "--tag", help="Attach a tag (repeatable)"),
) -> None:
    """Add a new task."""
    typer.echo(f"Added '{text}' with priority={priority}, tags={tags or []}")
```

```bash
taskcli add "ship release" --priority high --tag work --tag urgent
# Added 'ship release' with priority=high, tags=['work', 'urgent']
```

Notice `tags` is a repeatable option (`list[str]`) and Typer handles collecting multiple `--tag` flags into a list automatically, no manual `action="append"` bookkeeping like you'd need with argparse.

## Enums for constrained choices

`priority` above accepts any string, which means `--priority banana` silently "succeeds." Constrain it with an `Enum`, and Typer will validate the value and show the valid choices in `--help`:

```python
from enum import Enum
import typer

app = typer.Typer()


class Priority(str, Enum):
    low = "low"
    normal = "normal"
    high = "high"


@app.command()
def add(
    text: str,
    priority: Priority = typer.Option(Priority.normal, "--priority", "-p"),
) -> None:
    """Add a new task."""
    typer.echo(f"Added '{text}' with priority={priority.value}")
```

```bash
taskcli add "ship release" --priority urgent
# Error: Invalid value for '--priority' / '-p': 'urgent' is not one of 'low', 'normal', 'high'.
```

This is the kind of validation that's tedious to hand-roll with argparse (`choices=[...]` gets you partway, but not typed access to the value in your function body) and falls out naturally from Typer's type-hint-driven design.

## Real-world pattern: exit codes and error handling

A CLI tool that prints an error but exits with status 0 breaks every script or CI pipeline that checks its exit code. Use `typer.Exit` to control the exit code explicitly, and catch domain errors at the command boundary rather than letting a raw traceback hit the user:

```python
# src/taskcli/cli.py
import typer

app = typer.Typer()


class TaskNotFoundError(Exception):
    pass


def _get_task(task_id: int) -> dict:
    tasks = {1: {"text": "buy milk"}}
    if task_id not in tasks:
        raise TaskNotFoundError(f"No task with id {task_id}")
    return tasks[task_id]


@app.command()
def done(task_id: int) -> None:
    """Mark a task as complete."""
    try:
        task = _get_task(task_id)
    except TaskNotFoundError as e:
        typer.echo(f"Error: {e}", err=True)
        raise typer.Exit(code=1)

    typer.echo(f"Completed: {task['text']}")
```

```bash
taskcli done 99
# Error: No task with id 99
echo $?
# 1
```

Two details matter here: errors go to stderr (`err=True`), so they don't pollute stdout that a script might be parsing, and the process exits non-zero so `taskcli done 99 && echo "ok"` correctly skips the `echo`.

## Real-world pattern: a testable command with CliRunner

Because Typer commands are plain functions wired into an `app`, you can test them without spawning a subprocess, using Typer's `CliRunner` (re-exported from Click):

```python
# tests/test_cli.py
from typer.testing import CliRunner
from taskcli.cli import app

runner = CliRunner()


def test_add_task():
    result = runner.invoke(app, ["add", "buy milk", "--priority", "high"])
    assert result.exit_code == 0
    assert "buy milk" in result.stdout


def test_done_missing_task():
    result = runner.invoke(app, ["done", "99"])
    assert result.exit_code == 1
    assert "No task with id 99" in result.stderr
```

`CliRunner.invoke` runs the command in-process, captures stdout/stderr separately, and gives you the exit code, so these tests run in milliseconds instead of the seconds a real subprocess launch would cost. This is worth setting up early: once a CLI has a handful of commands and flag combinations, manual testing by hand stops scaling.

## Grouping commands as the tool grows

`taskcli` only has a few commands so a single `app` is fine, but Typer supports sub-apps for grouping related commands under a namespace, which becomes useful once a tool has distinct areas of functionality:

```python
import typer

app = typer.Typer()
tag_app = typer.Typer(help="Manage tags on tasks.")
app.add_typer(tag_app, name="tag")


@tag_app.command("add")
def tag_add(task_id: int, tag: str) -> None:
    typer.echo(f"Added tag '{tag}' to task {task_id}")


@tag_app.command("remove")
def tag_remove(task_id: int, tag: str) -> None:
    typer.echo(f"Removed tag '{tag}' from task {task_id}")
```

This gives you `taskcli tag add 3 urgent` and `taskcli tag remove 3 urgent`, cleanly namespaced instead of top-level commands named `tag-add` and `tag-remove`.

## What's next

We have a working command structure, but the task data disappears the moment the process exits, and there's no way to install `taskcli` as a real command on your system yet. Part 2 covers persisting tasks with a config/data file and packaging the project properly with `pyproject.toml`.
