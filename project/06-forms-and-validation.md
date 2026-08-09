# Forms & Validation

Validating a login or signup form in raw Flet means manually tracking an error string per field, wiring up `on_blur`, and remembering not to show a "this field is required" error before the user has typed anything at all. Artemis's `Field`/`Form` pair does all of that once.

## The pieces

```python
from artemis import Field, Form, validators
```

(`Field`, `Form`, and `validators` are all available directly off the top-level `art` import too — `art.Field`, `art.Form`, `art.validators`.)

### `Field`

```python
Field(value="", *validators)
```

Wraps a `State` (accessible as `.state`, though you'll rarely need it directly) plus zero or more validator functions.

```python
email = art.Field("", art.validators.required(), art.validators.email())
```

Properties:

- **`.value`** — get or set the current value, same as `State.value`.
- **`.error`** — the first validator's failure message, or `None` if the field is valid *or hasn't been touched yet*. Touch-gating is deliberate — see below.
- **`.touched`** — whether the field has been blurred or a submit was attempted on it.
- **`.touch()`** — manually mark it touched (Artemis's `Input(field=...)` calls this automatically on blur).
- **`.is_valid()`** — runs every validator regardless of `.touched`, returning `True`/`False`. Use this when you need a definite answer, not the touch-gated `.error`.

### Why errors are touch-gated

`.error` returns `None` until the field has been touched, even if the current value is genuinely invalid. This is intentional: a fresh, empty form shouldn't greet the user with a wall of red "required" errors before they've had a chance to type anything. The error only appears once they've interacted with that field (typically by clicking into it and then away — a blur), or once a form submit was attempted with it still invalid.

### `Form`

```python
Form(**fields)
```

Ties named fields together for submit-time validation:

```python
form = art.Form(email=email, password=password)
```

- **`.is_valid()`** — touches every field (making any errors visible) and returns whether all of them pass.
- **`.submit(on_valid)`** — returns an `on_click`-ready handler. Touches every field; if all pass, calls `on_valid(values)` where `values` is a plain `{name: value}` dict built from the field names you passed to `Form(...)`. If any field fails, `on_valid` is simply never called — no exception, no special-casing needed on your end.

## Putting it together

```python
import artemis as art

email = art.Field("", art.validators.required(), art.validators.email())
password = art.Field("", art.validators.required(), art.validators.min_length(6))
form = art.Form(email=email, password=password)

def handle_login(values):
    app.toast(f"Welcome, {values['email']}!")

@app.page("/")
def home(page):
    return art.Column([
        art.Input(label="Email", field=email),
        art.Input(label="Password", field=password, password=True),
        art.Button("Sign in", on_click=form.submit(handle_login)),
    ])
```

`art.Input(field=email)` is what wires the field's error message up to the visible UI — see [Widget Reference → Input](03-widgets.md#input) for the full details of how `field=` differs from the plainer `bind=`.

## Built-in validators

Every validator is a *factory function* — call it to get the actual checker, which takes a value and returns an error message string (or `None` if the value's fine):

```python
art.validators.required(message="This field is required")
art.validators.email(message="Enter a valid email address")
art.validators.min_length(n, message=None)
art.validators.max_length(n, message=None)
art.validators.matches(other_field, message="Fields don't match")
art.validators.number(message="Must be a number")
```

- **`required()`** — fails on empty or whitespace-only values.
- **`email()`** — a pragmatic (not RFC-exhaustive) `something@something.something` pattern check.
- **`min_length(n)`** / **`max_length(n)`** — string length bounds. Default messages fill in `n` for you if you don't supply your own.
- **`matches(other_field)`** — for "confirm password" style fields; pass the *other* `Field` object, and this validator checks the current value against `other_field.value` at validation time (so it stays correct even if the other field changes later).
- **`number()`** — passes if the value can be parsed as a float.

```python
password = art.Field("", art.validators.required(), art.validators.min_length(8))
confirm = art.Field("", art.validators.matches(password, message="Passwords don't match"))
```

Fields can have any number of validators — they run in order, and `.error` shows the *first* one that fails.

## Writing your own validator

A validator is just a function that takes a value and returns a string (the error) or `None` (valid). The built-in ones are all factories that return such a function, following the same shape:

```python
def even_number(message="Must be an even number"):
    def check(value):
        try:
            return None if int(value) % 2 == 0 else message
        except (TypeError, ValueError):
            return message
    return check

quantity = art.Field("0", art.validators.number(), even_number())
```

There's no registration step, no base class to inherit from — if it matches the `value -> str | None` shape, `Field` will call it correctly.

## See it running

`examples/login.py` is a complete, minimal working form using everything on this page. `tests/test_examples.py` (see [Testing](09-testing.md)) doesn't currently cover it directly, but the same `TestApp` harness works identically for form flows — type into an input, click submit, assert on the resulting error or success state.

## Next

**[Data & Networking →](07-data-and-networking.md)**
