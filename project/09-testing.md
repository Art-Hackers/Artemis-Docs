# Testing

`artemis.testing` lets you test an Artemis app without opening a real window — no Flet client, no display, nothing platform-specific. It's genuinely useful in CI, and just as useful while you're actively building a feature, so you're not re-clicking through the same UI by hand after every code change.

This isn't a thin afterthought bolted on — it's a formalized version of the exact fake-page pattern used to verify every single feature described in the rest of this documentation before it was written down.

## How it works

`TestApp` gives your `App` instance a `FakePage` — an object that implements just enough of Flet's real `Page` (views, overlay, dialogs, clipboard, `update()`) for your actual page functions and event handlers to run for real. It does **not** spin up Flet's real client or rendering engine at all. This is about testing *your app's logic* — does clicking this button produce that text, does this form reject invalid input, does this route redirect correctly — not about testing Flet's own rendering, which is Flet's job to get right, not yours or Artemis's.

## Basic usage

```python
from artemis.testing import TestApp

def test_counter():
    t = TestApp(app).build()
    assert t.has_text("0")

    t.click(t.find_button("+"))
    assert t.has_text("1")
```

`TestApp(app)` wraps your `App` instance. `.build()` starts it against the fake page (equivalent to what happens right when a real app window opens) and returns `self`, so you can chain it.

## API reference

### `TestApp(app)`

Wraps an `artemis.App`. Creates a fresh `FakePage` internally.

### `.build(route="/")`

Starts the app — calls the same internal setup a real `app.run()` would trigger, against the fake page instead of a real one. Must be called before anything else. Returns `self`.

### `.go(route)` / `.back()`

Thin wrappers around `app.go(...)`/`app.back()`, for readability in tests. Return `self`.

### `.current_route()`

The route of the currently-visible screen (the top of the navigation stack).

### `.all_controls()`

A generator yielding every control currently on screen, across all views on the stack, flattened recursively — including nested children (a control's `.controls`, `.content`, and for list-tile-shaped controls, `.title`/`.subtitle`/`.leading`/`.trailing`). Rarely used directly; the `find_*`/`has_*` helpers below are usually more convenient.

### `.find_text(text)` / `.has_text(text)`

```python
t.has_text("Welcome back")
control = t.find_text("Welcome back")   # returns the actual control, or None
```

Looks for a control whose displayed `.value` exactly equals `text`. `has_text` is the boolean convenience form for assertions; `find_text` gives you the actual control if you need to inspect it further.

### `.find_button(label)`

```python
button = t.find_button("Save")
```

Finds a button by its visible label. Works because Artemis's `Button(text, ...)` stores that text as the button's `content` — this searches for a control whose `.content` matches `label` and which has an `on_click` handler.

### `.click(control)`

```python
t.click(t.find_button("+"))
```

Fires the control's `on_click` handler, the same way a real tap would — including awaiting it automatically if it's an `async def` handler, so testing async button behavior (network calls, `Button(loading=...)`) needs no special handling on your end. Raises a clear `ValueError` if you pass `None` (usually meaning `find_button`/`find_text` didn't find anything — a much more useful error than a cryptic `AttributeError` on `None`) or a control with no `on_click` at all.

### `.type_into(control, text)`

```python
email_input = next(c for c in t.all_controls() if getattr(c, "label", None) == "Email")
t.type_into(email_input, "test@example.com")
```

Simulates typing into a text field: sets `.value` and fires `on_change`, the way real keystrokes would (batched into one call — this doesn't simulate individual keypresses, just the end result of typing something and Flet's `on_change` firing).

### `.last_dialog()`

```python
t.click(delete_button)
dialog = t.last_dialog()
assert dialog.content.value == "Task deleted"
```

The most recently shown dialog-like control (`SnackBar`, `AlertDialog`, `DatePicker`, anything shown via `page.show_dialog(...)`) — useful for asserting on toasts, alerts, and confirmations. Returns `None` if nothing's been shown yet.

## A complete example

This is drawn directly from `tests/test_examples.py`, which is a real, currently-passing test suite for two of the bundled example apps:

```python
import sys
from pathlib import Path

sys.path.insert(0, str(Path(__file__).resolve().parent.parent / "examples"))

from artemis.testing import TestApp
import counter
import todo


def test_counter_increments():
    counter.count.value = 0  # reset shared module-level state between tests
    t = TestApp(counter.app).build()
    t.click(t.find_button("+"))
    assert t.has_text("1")


def test_todo_add_task():
    todo.tasks.value = []
    t = TestApp(todo.app).build()

    task_input = next(c for c in t.all_controls() if getattr(c, "label", None) == "New task")
    t.type_into(task_input, "Buy milk")
    t.click(t.find_button("Add"))

    assert t.has_text("Buy milk")
    assert not t.has_text("Nothing here yet - add something below.")
```

Run it with `pytest`:

```bash
pip install pytest
pytest tests/
```

## A note on shared state between tests

Because Artemis app modules typically hold their `State`/`AsyncData` objects at module level (as in `counter.py`'s `count = art.State(0)`), that state persists across test functions within the same test run — which is why the examples above explicitly reset it (`counter.count.value = 0`) at the start of each test. This isn't a quirk of `TestApp`; it's the same behavior a real running app has (state lives for the process's lifetime), surfaced by running multiple tests in one process. If this bothers you for a specific app, structure your app's state inside a factory function instead of at module level, and construct a fresh instance per test.

## Testing route guards, dialogs, and async flows

Everything covered elsewhere in this documentation is testable the same way:

```python
# route guards (see Navigation)
t.go("/admin")
assert t.current_route() == "/login"   # redirected, guard failed

# confirm dialogs (see Dialogs & Feedback)
t.click(delete_button)
dialog = t.last_dialog()
dialog.actions[1].on_click(None)   # tap the confirm action

# AsyncData / async handlers (see Data & Networking)
t.click(load_button)   # awaited automatically since the handler is async
assert not products.loading
```

## Why `TestApp`, not `Test...` naming conventions tripping up pytest

If you've used pytest before, you may know it auto-collects any class whose name starts with `Test` as a test class by default. `TestApp` deliberately sets `__test__ = False` internally specifically to opt out of that — so `from artemis.testing import TestApp` is safe to use directly in a pytest file without pytest trying (and failing) to collect `TestApp` itself as a test.

## Next

**[The `artemis` CLI →](10-cli.md)**
