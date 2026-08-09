# Core Concepts

Everything else in this documentation is easier once these ideas click. This page is the one worth reading slowly.

## One `App`

Your whole application is a single `App` instance:

```python
app = art.App(
    title="My App",
    theme="indigo",
    window_size=(400, 700),
)
```

You create exactly one, register your screens on it with `@app.page(...)`, and call `app.run()` at the end of your file. There's no separate router class, no "provider" tree to wire up, no context objects to thread through your widget hierarchy. If you need something from anywhere in your app, it's usually either a plain module-level variable, a `State`, or a method on `app` itself.

## Pages are just functions

```python
@app.page("/")
def home(page):
    return art.Text("hello")
```

A page is a function that:

- Takes `page` (Flet's page object) as its first argument — and optionally a second `params` argument if the route has `:name` segments (see [Navigation](05-navigation.md#route-parameters)).
- Returns **one control**, or a **list/tuple of controls** — either is fine. Usually you'll return a single `Column` or `Row` wrapping everything.

That's the entire contract. There's no lifecycle to learn (no `componentDidMount`, no `useEffect`, no `build()` override) — it's a plain function, called and its return value displayed.

## The re-render model — the single most important thing to understand

Here's the part that's different from vanilla Flet, and understanding *why* it works this way will save you real confusion:

**Every time a button is clicked (or a switch flipped, or a slider dragged — any "discrete" control), Artemis re-runs the current screen's page function from scratch and redraws the whole screen with the new result.**

You are not expected to call `page.update()` yourself. You're not expected to manually mutate an existing control's property and ask Flet to patch it in place. You just change some state, and the *next* time your page function runs — which Artemis triggers for you, automatically — it naturally reflects the new value, because it's a normal Python function computing its return value from current data.

```python
count = art.State(0)

@app.page("/")
def home(page):
    return art.Column([
        art.Text(str(count.value)),                              # reads current value
        art.Button("+", on_click=lambda e: count.set(count.value + 1)),
    ])
```

Click "+" and here's what actually happens, in order:

1. Your lambda runs: `count.set(count.value + 1)` — a plain Python mutation of a plain Python object. Nothing Flet-specific happens yet.
2. The `Button` widget's internal wrapper (every discrete Artemis widget has one) notices the click handler finished, and calls Artemis's internal re-render function.
3. That function looks up which screen is currently showing, calls `home(page)` again — a completely fresh call, building a brand new `Column`/`Text`/`Button` tree from scratch — and hands the new tree to Flet, which draws it.
4. `Text(str(count.value))` in that fresh call now reads the updated value, because `count.value` really did change in step 1.

This is deliberately closer to how frameworks like Streamlit or (conceptually) React work than to Flet's own default imperative style, where you're expected to mutate `some_control.value = x` and then call `page.update()`. Artemis trades a small amount of raw performance (rebuilding the tree instead of patching one property) for a much simpler mental model: **your page function is a pure description of "what does this screen look like right now," and you never have to manually keep the screen in sync with your data** — it's automatically always in sync, because it's recomputed from that data every time something happens.

### Why `Input` is the one exception

If Artemis redrew the whole screen on *every keystroke*, typing into a text field would be miserable — the cursor position and keyboard focus would reset with every character. So `Input()` deliberately does **not** trigger a redraw on `on_change`. Instead, you bind it to a `State` and Artemis quietly keeps that state updated as you type, with no redraw:

```python
name = art.State("")

art.Input(label="Your name", bind=name)
```

Read `name.value` later — in a button's `on_click`, for instance — and it'll have the latest typed value, even though no redraw happened while you were typing. This is covered in more depth in [Widget Reference → Input](03-widgets.md#input).

### Async handlers work exactly the same way

`on_click`/`on_change` can be a plain function or an `async def` — Flet natively awaits coroutine event handlers, and Artemis's auto-redraw wrapper handles both cases identically: the redraw happens right after your handler (awaited, if it's async) finishes.

```python
async def load(e):
    data = await art.fetch_json(url)
    results.value = data

art.Button("Load", on_click=load)
```

See [Data & Networking](07-data-and-networking.md) for the full story on async work, including the `AsyncData` helper that solves the "don't accidentally re-fetch on every unrelated re-render" problem this model would otherwise create.

## `State` — a box, not magic

```python
class State:
    def __init__(self, value): ...
    @property
    def value(self): ...
    @value.setter
    def value(self, new): ...
    def set(self, new): ...   # same as .value = new, usable inside a lambda
    def toggle(self): ...      # flips a boolean
```

`State` exists to solve one specific, boring Python problem: a plain local variable can't be reassigned from inside a nested function (a `lambda` or `def` inside your page function) without `nonlocal` declarations everywhere, which gets ugly fast. `State` sidesteps that by being a mutable *object* — you're not reassigning a variable, you're mutating an attribute of an object you already have a reference to, which Python closures handle fine without any extra ceremony.

There is no dependency graph, no computed/derived value system, no automatic subscription tracking. It's genuinely just a box. Given the re-render model above, that's actually all it needs to be — since your whole page function re-runs and recomputes everything from current state on every interaction, there's no need for `State` itself to know who's "listening" to it. It's not React's `useState`; it's closer to a mutable ref cell that happens to also update the screen for you when changed via a widget's own handler.

Two variants build on the same `State` API:

- **`PersistentState`** — identical interface, but every write is also saved to a small JSON file, so the value survives an app restart. Full details in [Data & Networking](07-data-and-networking.md#persistentstate).
- **`AsyncData`** — a different shape (`.loading`/`.error`/`.value` instead of a single `.value`), purpose-built for the "fetch data once per screen visit" pattern. Details in [Data & Networking](07-data-and-networking.md#asyncdata).

## Navigation is a real stack, not a page swap

`app.go("/details")` and `app.back()` push and pop a genuine navigation stack backed by Flet's `page.views` — not a simple "clear the screen and draw something else" swap. This matters because it means:

- A back arrow appears automatically in the AppBar once there's more than one screen on the stack.
- Android's hardware back button, and the browser back button (in web mode), both correctly pop the stack instead of doing nothing or closing the app outright.
- A bottom tab bar (`app.bottom_nav(...)`) or side drawer (`app.set_drawer(...)`) can correctly show or hide itself depending on whether you're on a root screen or several levels deep — the way a real native app behaves.

Full details in [Navigation](05-navigation.md).

## A broken screen doesn't crash the whole app

If a page function raises an exception — a bad route parameter, a typo in a dict key, anything — Artemis catches it at the point of rendering and shows a plain "something went wrong" screen with a "Go home" button, instead of the entire application dying. The real traceback still prints to your console (so you can actually debug it), but the *user* never sees a frozen or blank window. This happens automatically; there's nothing to configure.

## Everything is a plain Flet control

Worth repeating from the index page, because it's the thing that makes Artemis low-risk to adopt: `art.Column(...)` returns an actual `flet.Column`. `art.Button(...)` returns an actual `flet.FilledButton` (or whichever variant you asked for). There is no Artemis-specific control class sitting between you and Flet. This means:

- You can freely mix `art.*` widgets with raw `flet.*` controls in the same tree.
- Anything Artemis hasn't wrapped (an icon, a niche control, a property Artemis's wrapper doesn't expose) is one `import flet as ft` away — `art.flet` is that same import, re-exported so you don't need a second import line.
- If you already know Flet, none of this requires re-learning — it requires learning a smaller vocabulary that happens to produce nicer defaults.

## Next

**[Widget Reference →](03-widgets.md)** — every widget, every parameter, with examples.
