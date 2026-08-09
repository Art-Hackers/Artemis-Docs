# Dialogs & Feedback

Everything on this page is a method on `app` (except `toast`, which is also available as a standalone `art.toast(...)` function — see below) — no `page` argument to pass around, because Artemis already knows which page it's running on.

## Toasts

```python
app.toast(message, bg=None, seconds=3, action=None, on_action=None)
```

A one-line snackbar:

```python
app.toast("Saved!")
```

In raw Flet, this is `page.show_dialog(ft.SnackBar(...))` — plus first knowing that `SnackBar` is technically a dialog-like control at all, which trips people up regularly since "toast" and "dialog" don't sound related.

**With an action button:**

```python
app.toast("Task deleted", action="Undo", on_action=lambda e: restore(task))
```

`on_action` is called if the user taps the action button before the toast dismisses on its own — a genuinely common pattern (delete-with-undo) that's otherwise a few lines of manual `SnackBar` construction.

`bg` accepts a palette name or hex string, same as any other Artemis color parameter. `seconds` controls how long it stays visible before auto-dismissing.

`art.toast(...)` (imported directly, no `app.` prefix) does the same thing and is what `app.toast` calls internally — use whichever reads better in context; they're identical.

## Alerts

```python
app.alert(title, message, on_close=None)
```

A one-line "OK" dialog:

```python
app.alert("Heads up", "Something happened.")
```

`on_close`, if given, is called after the dialog is dismissed (whether by tapping "OK" or otherwise).

## Confirmations

```python
app.confirm(title, message, on_confirm, on_cancel=None, confirm_text="Yes", cancel_text="Cancel")
```

A yes/no dialog:

```python
app.confirm(
    "Delete task?",
    "This can't be undone.",
    on_confirm=lambda e: delete_task(),
)
```

Fires `on_confirm` or `on_cancel` depending on which button gets tapped, then closes itself either way — you don't need to manually set `dialog.open = False` or call `page.update()` yourself in either callback.

## Date & time pickers

```python
app.pick_date(on_result, first_date=None, last_date=None, help_text=None)
app.pick_time(on_result, help_text=None)
```

Both return an `on_click`-ready handler that opens the platform's native date/time picker:

```python
art.Button("Pick a date", on_click=app.pick_date(lambda d: print(d)))
```

`on_result` receives a `datetime.date` (for `pick_date`) or `datetime.time` (for `pick_time`), or `None` if the user cancelled without picking anything. `first_date`/`last_date` bound the selectable range for `pick_date`.

These go through Flet's `page.show_dialog(...)` mechanism internally — the same reliable path `alert`/`confirm` use — which is worth noting because it's a different (and considerably more reliable) mechanism than the one Flet's `FilePicker` uses. See [Troubleshooting](11-troubleshooting-and-faq.md#why-is-there-no-file-picker) for why that distinction matters and why Artemis doesn't currently wrap `FilePicker` at all.

## Keyboard shortcuts

```python
app.on_key(combo, handler)
```

Registers a global keyboard shortcut, primarily useful for desktop apps:

```python
app.on_key("ctrl+s", lambda e: save())
app.on_key("escape", lambda e: app.back())
```

`combo` is a plain string — modifiers (`ctrl`, `shift`, `alt`, `meta`) are optional, and their order doesn't matter (`"shift+ctrl+n"` and `"ctrl+shift+n"` both register the same shortcut). The final segment is the key itself.

Shortcuts are global to the app, not scoped to a particular screen — a shortcut registered once via `app.on_key(...)` fires no matter which screen is currently showing. If you need screen-specific shortcuts, check `app._nav_stack` (or, more cleanly, your own state tracking which screen is active) inside the handler and no-op if it's the wrong screen.

## Clipboard

```python
app.copy(text)
app.paste(on_result)
```

```python
art.Button("Copy link", on_click=lambda e: app.copy("https://example.com/shared"))
art.Button("Paste", on_click=app.paste(lambda text: print("got:", text)))
```

`app.copy(text)` is fire-and-forget — call it directly from a plain (non-async) `on_click`. `app.paste(...)` returns an `on_click`-ready handler, the same shape as `pick_date`/`pick_time`, because reading the clipboard is inherently asynchronous in Flet (you can't return a value synchronously from "what's on the system clipboard right now").

## Error screens (automatic, nothing to call)

Covered fully in [Core Concepts](02-core-concepts.md#a-broken-screen-doesnt-crash-the-whole-app) — worth a one-line mention here too since it's arguably the most important piece of "feedback" in the whole app: if a page function raises, Artemis shows a friendly error screen with a "Go home" button instead of crashing outright. Nothing to configure; it's on by default for every screen.

## Next

**[Testing →](09-testing.md)**
