# Getting Started

## Requirements

- Python 3.9 or newer. (One caveat worth knowing up front: **Python 3.14** currently has a real, upstream compatibility gap with some of Flet's control-registration internals — see [Troubleshooting](11-troubleshooting-and-faq.md#python-314) if you're on it and something behaves strangely. Python 3.12 or 3.13 are the best-tested targets right now.)
- `flet >= 0.86.0` (installed automatically as a dependency).
- On Linux desktop specifically, a few system packages Flet itself needs for its native window — nothing Artemis adds on top.

## Installing

From the project root (the folder containing `pyproject.toml`):

```bash
pip install artemis-ui
```

That installs Artemis in "editable" mode, meaning changes to the `artemis/` source take effect immediately without reinstalling — useful if you're modifying the library itself, and harmless if you're not.

Two optional extras exist, install either or both as needed:

```bash
pip install artemis-ui ".[icons]"    # lets Artemis auto-convert a custom logo.png into a Windows .ico
pip install artemis-ui ".[charts]"   # needed for art.LineChart / BarChart / PieChart
```

Neither is required for a basic app. Artemis only complains (with a clear message, not a stack trace) if you actually try to use a chart without `flet-charts` installed.

### A note if you extracted this from a zip

If you're reading this because you downloaded and extracted an Artemis project archive: **when a new version changes or removes files, extracting a new zip on top of an old folder does not delete anything.** Only additions and overwrites happen — stale files silently stick around and can cause confusing `AttributeError`s referencing methods that no longer exist. When in doubt, delete the whole folder and extract fresh into an empty location.

## Your first app

Create `main.py`:

```python
import artemis as art

app = art.App("My First App", theme="ocean")

@app.page("/")
def home(page):
    return art.Column(
        [
            art.Title("Hello, Artemis"),
            art.Text("Edit main.py and re-run to see changes."),
            art.Button("Click me", on_click=lambda e: app.toast("Hi!")),
        ],
        center=True,
        expand=True,
    )

if __name__ == "__main__":
    app.run()
```

Run it:

```bash
python main.py
```

A native window opens (on desktop) with your content, a Material 3 color scheme derived from `"ocean"`, and a working button. That's genuinely the whole app — there's no separate template file, no XML, no build step required just to see it.

## What just happened, in order

1. `art.App(...)` created an application object. Nothing renders yet — this just configures things (title, theme, window size if you set one).
2. `@app.page("/")` registered a function as the handler for the `"/"` route. A "page" in Artemis is just a Python function that takes `page` (Flet's page object — you'll rarely touch it directly) and returns a control tree.
3. `app.run()` handed control to Flet, which opens a window, calls your `"/"` handler once to get the initial content, and starts listening for events.
4. Clicking the button called your `on_click` lambda, which called `app.toast(...)`. Behind the scenes, Artemis then automatically re-ran your `home()` function and redrew the screen — you didn't have to call anything like `page.update()` yourself. This automatic-redraw behavior is central enough to how Artemis works that it gets its own explanation in [Core Concepts](02-core-concepts.md) — read that next if anything above felt like it was glossing over how the "magic" works, because it isn't magic, and understanding it will save you real confusion later.

## Project layout as it grows

A minimal Artemis project needs only a script and an installed `artemis` package. As you build a real app, a few folders show up automatically — you don't create these by hand:

- **`assets/`** — created the first time you call `app.run()`. Artemis drops a default logo (`logo.png`, `icon.png`, and on Windows a `logo.ico`) in here so your app doesn't show a generic icon. Drop your own `logo.png` in this folder before running, and Artemis uses yours instead — it never overwrites a file that's already there. Full details in [Theming & Branding](04-theming-and-branding.md#branding-your-app-icon).
- **`.artemis_data/`** — created the first time you use `art.PersistentState`. Plain JSON files, one per key, that survive an app restart. Details in [Data & Networking](07-data-and-networking.md#persistentstate).

Both are safe to delete — Artemis just recreates them (with fresh defaults) the next time it needs them.

## Running the example apps

If you have the full Artemis source (not just installed via pip), there's an `examples/` folder with real, working apps demonstrating specific features — a counter, a todo list, a themed dashboard with charts, a login form, and more. Each one is a complete, runnable file:

```bash
python examples/counter.py
python examples/dashboard.py
```

Reading these is a genuinely good way to learn the library — they're referenced throughout this documentation wherever a feature has a matching example.

## Building a real, installable app

Everything above runs your app in a Python-driven dev preview. To produce something you can actually install on a phone or hand to someone else, use Flet's own build tooling directly — Artemis doesn't reinvent this part, because Flet already does it well:

```bash
flet build apk        # Android
flet build ipa         # iOS
flet build macos        # macOS
flet build windows        # Windows
flet build linux        # Linux
```

The `assets/icon.png` that Artemis auto-creates (see [Theming & Branding](04-theming-and-branding.md)) is exactly the file `flet build` looks for to generate the real, installed-app icon on every platform — so a fresh Artemis project is already correctly branded for a real build, not just the dev preview.

## Next

**[Core Concepts →](02-core-concepts.md)** — the mental model that makes everything else make sense.
