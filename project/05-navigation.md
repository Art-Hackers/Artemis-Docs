# Navigation

As established in [Core Concepts](02-core-concepts.md#navigation-is-a-real-stack-not-a-page-swap), Artemis's navigation is a genuine stack backed by Flet's `page.views`, not a simple content swap. This page covers everything built on top of that stack.

## Registering screens

```python
@app.page(route="/", title=None, appbar=True, guard=None, redirect="/")
def home(page):
    return art.Text("hello")
```

- **`route`** — the path this function handles. `"/"` is the default and conventionally your home screen.
- **`title`** — shown in this screen's AppBar. Falls back to the app's own `title` (set in `App(title=...)`) if not given.
- **`appbar=False`** — renders this screen with no AppBar at all — useful for splash screens, full-bleed images, or any screen that shouldn't have the standard top bar chrome.
- **`guard`** / **`redirect`** — covered in [Route Guards](#route-guards) below.

## Pushing and popping screens

```python
app.go("/details")   # push a new screen onto the stack
app.back()            # pop the current screen, returning to whatever was under it
```

`app.go(...)` shows a back arrow in the AppBar automatically — you don't wire this up yourself. `app.back()` is a no-op if there's only one screen on the stack (nothing to go back to).

The same back-stack is also driven by:

- Tapping the AppBar's auto-generated back arrow.
- Android's hardware back button.
- The browser's back button, in web-mode builds.

All three correctly pop the Artemis-managed stack, not just Flet's underlying route state — this is one of the concrete things a raw `page.controls.clear()`-based approach (which some minimal Flet wrappers use) gets wrong, and Artemis deliberately doesn't.

## Route parameters

Routes can carry named segments with a `:name` prefix:

```python
@app.page("/user/:id")
def profile(page, params):
    return art.Text(f"User #{params['id']}")
```

```python
app.go("/user/42")   # matches the pattern above, calls profile(page, {"id": "42"})
```

Artemis only passes the second `params` argument if your function actually declares one — a plain `def home(page):` handler with no second parameter keeps working exactly as before, so adding params to one route never forces you to touch unrelated ones. Multiple named segments in one route work as you'd expect: `/user/:id/post/:post_id` produces `{"id": ..., "post_id": ...}`.

Route matching checks exact matches first (fast path), then falls back to checking registered `:param` routes in registration order. If nothing matches at all, Artemis falls back to whatever's registered at `"/"`.

## Route guards

```python
app.page("/admin", guard=lambda: current_user.is_admin, redirect="/login")
```

`guard` is a zero-argument function, checked right before that screen is about to render. If it returns a falsy value, Artemis renders `redirect` instead (default `"/"`) — the guarded screen's function never even gets called. This saves repeating the same auth check inside every protected page function.

```python
logged_in = art.State(False)

@app.page("/admin", guard=lambda: logged_in.value, redirect="/login")
def admin(page):
    return art.Text("Secret admin stuff")

@app.page("/login")
def login(page):
    return art.Text("Please log in")
```

Guards are checked once per render, using whatever `guard()` returns *at that moment* — there's no caching, so a guard reading live state (like the `logged_in.value` above) behaves correctly if that state changes between visits.

## Bottom tab bar

```python
app.bottom_nav([
    {"label": "Home", "icon": art.flet.Icons.HOME, "route": "/"},
    {"label": "Search", "icon": art.flet.Icons.SEARCH, "route": "/search"},
    {"label": "You", "icon": art.flet.Icons.PERSON, "route": "/profile"},
])
```

A persistent bottom tab bar across your top-level (root) screens. Tapping a tab **replaces** the whole stack with that tab's route — it's a tab switch, not a push, since there's conceptually nothing to "go back" to from a tab. The bar itself only appears on root-level screens: if you `app.go()` deeper into something from within a tab, the bar correctly hides (the way it would in a real native app), and reappears once you `app.back()` to the root again.

## Side navigation drawer

```python
app.set_drawer([
    {"label": "Home", "icon": art.flet.Icons.HOME, "route": "/"},
    {"label": "Settings", "icon": art.flet.Icons.SETTINGS, "route": "/settings"},
])
```

The same idea as `bottom_nav`, but as a side menu — shows a hamburger icon in the AppBar automatically (standard Flutter behavior once a screen has a drawer attached; Artemis doesn't hand-wire this icon in). A side drawer is generally the better fit for a wide desktop window, since it doesn't eat vertical space the way a bottom bar does on a tall, narrow phone screen. Like `bottom_nav`, it only shows on root-level screens.

You can use `bottom_nav` and `set_drawer` together if you genuinely want both (e.g., a drawer for less-common destinations, a bottom bar for the primary few) — they don't conflict, though most apps only need one.

## Page transitions

```python
app = art.App("My App", transitions="cupertino")   # or "fade", "zoom", "none"
```

Covered in more depth in [Theming & Branding](04-theming-and-branding.md#page-transitions), since it's set on `App(...)` alongside the other theme options — but it's worth knowing it lives here conceptually too, since it's specifically the *navigation* transition (what happens visually when you `app.go()`/`app.back()`), not a general animation system.

## Responsive layouts

Not navigation exactly, but closely related to building one app that adapts across window sizes — see [`art.responsive(...)`](02-core-concepts.md) — actually covered fully in the Widget Reference's layout section and demonstrated in `examples/dashboard.py`. The short version:

```python
@app.page("/")
def home(page):
    return art.responsive(
        page,
        mobile=art.Column([...]),
        desktop=art.Row([...]),
        desktop_at=700,
    )
```

Artemis re-renders the current screen on window resize, so this picks up live changes as a desktop window is resized — genuinely one codebase behaving differently on a phone versus a wide window, which is the whole point of a library aimed at both Android and desktop.

## Next

**[Forms & Validation →](06-forms-and-validation.md)**
