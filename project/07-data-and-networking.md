# Data & Networking

Artemis gives you three related-but-distinct tools for holding data, and one small set of helpers for getting data from the network. This page explains when to reach for each.

## The three state tools, compared

| | Lives where | Survives restart? | Shape | Use for |
|---|---|---|---|---|
| `State` | Memory | No | single `.value` | everyday UI state — a counter, a toggle, a typed-but-not-submitted form value |
| `PersistentState` | A JSON file next to your script | Yes | single `.value`, same API as `State` | small preferences — theme choice, last tab, a short local list |
| `AsyncData` | Memory | No | `.value` / `.loading` / `.error` | data fetched from somewhere (an API, a slow computation) |

`State` is covered in depth in [Core Concepts](02-core-concepts.md#state--a-box-not-magic) — this page picks up from there.

## `PersistentState`

```python
theme_pref = art.PersistentState("theme", default="indigo")
theme_pref.value = "forest"   # written to disk immediately, no extra step
```

Identical interface to `State` — it's a drop-in replacement — but every write is also saved to a small JSON file. On creation, it reads whatever was saved last time; if nothing was ever saved, it falls back to `default` (and writes nothing until you actually change it).

**Where the file lives:** a `.artemis_data/` folder created next to your script, the same convention `assets/` uses for branding — nothing tucked away in an OS-specific app-data folder you'd have to go hunting for. The key you pass becomes the filename (sanitized to alphanumerics/`-`/`_`), so `PersistentState("theme", ...)` writes to `.artemis_data/theme.json`.

**What you can store:** anything JSON-serializable — numbers, strings, lists, dicts, booleans, `None`. Artemis deliberately does not try to pickle arbitrary Python objects here; that approach gets fragile fast the moment your app's code changes shape between runs (a pickled instance of a class you've since modified can fail to load at all). If you need to persist something more complex than JSON-shaped data, encode/decode it yourself around a `PersistentState` holding the serialized form.

**Failure handling:** if the file can't be written (disk full, read-only filesystem, permissions), the write fails silently rather than crashing your app — you keep the in-memory value, you just don't get persistence for that particular write.

```python
# a theme preference that survives a restart, from examples/dashboard.py
theme_pref = art.PersistentState("dashboard_theme", default="ocean")
app.theme_name = theme_pref.value

def switch_theme(name):
    theme_pref.value = name
    app.set_theme(name)
```

## `AsyncData`

Every screen that loads data from somewhere ends up needing the same three things — a loading flag, an error slot, a value slot — and because Artemis re-runs your page function on *every* click anywhere in the app (see [Core Concepts](02-core-concepts.md#the-re-render-model)), you also have to make sure you don't accidentally re-fetch on every single one of those re-renders, not just the ones that should actually trigger a fetch. `AsyncData` solves all of it:

```python
products = art.AsyncData(lambda: art.fetch_json(URL))

@app.page("/")
def home(page):
    products.render(page)  # fetches once; a no-op on every re-render after that

    if products.loading:
        return art.Loader()
    if products.error:
        return art.Text(f"Couldn't load that: {products.error}")
    return art.Column([art.Text(p) for p in products.value])
```

### How `render(page)` avoids re-fetching

`render(page)` is what makes this safe to call from a function that re-runs constantly: internally, it checks a flag before doing anything. The *first* time it's called for a given `AsyncData` instance, it flips that flag, sets `.loading = True`, and kicks off the fetch as a background task via `page.run_task(...)`. Every call after that — which is most of them, since your page function re-runs on every unrelated click too — sees the flag already set and does nothing.

When the fetch completes (success or failure), `AsyncData` updates `.value`/`.error`/`.loading` and triggers a redraw itself, so the screen picks up the result without you wiring anything extra.

### Forcing a re-fetch

```python
def refresh(e):
    products.reset()
    app.refresh()

art.Button("Refresh", on_click=refresh)
```

`.reset()` clears the "already started" flag along with `.value`/`.error`/`.loading`, so the *next* `render(page)` call — which happens the moment your page function runs again — starts a fresh fetch. `app.refresh()` (or any state change that triggers a redraw) is what makes that next call happen.

### `loader` can be any zero-argument async function

```python
art.AsyncData(lambda: art.fetch_json(url))                    # a network call
art.AsyncData(lambda: some_async_database_query())              # your own async function
art.AsyncData(some_async_function_with_no_args)                 # doesn't have to be a lambda
```

There's nothing network-specific baked into `AsyncData` itself — it just awaits whatever zero-arg coroutine function you give it and captures the result or exception.

## Network calls

```python
await art.fetch_json(url, timeout=10, **kwargs)
await art.fetch_text(url, timeout=10, **kwargs)
await art.post_json(url, data=None, timeout=10, **kwargs)
```

Thin async wrappers around [`httpx`](https://www.python-httpx.org/) — which is already a dependency of Flet itself, so using these adds nothing new to your install. Each one:

- Opens a fresh `httpx.AsyncClient` for the call (no connection pooling across calls — fine for typical app usage; if you're making many rapid calls to the same host, consider managing your own `httpx.AsyncClient` instead).
- Applies a default 10-second timeout (override with `timeout=`).
- Calls `raise_for_status()` — a non-2xx response raises `httpx.HTTPStatusError` rather than silently returning an error body as if it were data.
- Passes any extra `**kwargs` straight through to the underlying `httpx` call (headers, params, auth, etc.).

```python
data = await art.fetch_json("https://api.example.com/items", headers={"Authorization": f"Bearer {token}"})
```

`post_json` sends `data` as a JSON request body (`httpx`'s `json=` parameter) and returns the parsed JSON response.

### Using these directly, without `AsyncData`

`AsyncData` is the right tool when a screen needs to *show* loading/error/success state visually. For a one-off action that doesn't need that (submitting a form, firing off an update, logging an event), just call the fetch functions directly inside an async handler — this is the same async-handler pattern covered in [Core Concepts](02-core-concepts.md#async-handlers-work-exactly-the-same-way):

```python
async def submit(e):
    await art.post_json(API_URL, data={"name": name.value})
    app.toast("Saved!")

art.Button("Save", on_click=submit)
```

Combine this with `Button(loading=...)` (see [Widget Reference](03-widgets.md#button)) for an automatic spinner while the request is in flight:

```python
saving = art.State(False)
art.Button("Save", on_click=submit, loading=saving)
```

## See it running

`examples/network_and_guards.py` and `examples/async_data_demo.py` both use everything on this page in a complete, runnable app — including one test in `tests/test_examples.py` that exercises `AsyncData` against a real live network call, not a mock.

## Next

**[Dialogs & Feedback →](08-dialogs-and-feedback.md)**
