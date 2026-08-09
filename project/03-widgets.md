# Widget Reference

Every function on this page returns a plain Flet control. Every one accepts `**kw`, which passes straight through to the underlying Flet control constructor — so if a Flet property isn't explicitly named in an Artemis wrapper's signature, you can usually still set it directly as a keyword argument.

Import convention used throughout: `import artemis as art`.

---

## Text

```python
art.Text(value="", size=16, bold=False, italic=False, color=None, center=False, muted=False, **kw)
```

The basic text control.

- `color` — a palette name (`"rose"`, `"forest"`, ...) or a raw hex string. See [Theming](04-theming-and-branding.md#named-palettes) for the full palette list.
- `muted=True` — a quick way to get a dimmer, secondary-text color (`ON_SURFACE_VARIANT`) without picking a specific shade yourself. Ignored if you also pass `color`.
- `center=True` — center-aligns the text horizontally within its container.

```python
art.Text("Welcome back", size=20, bold=True)
art.Text("Last updated 2 minutes ago", muted=True, size=12)
```

### Title

```python
art.Title(value, size=28, **kw)
```

Exactly `Text(value, size=size, bold=True, **kw)` — a heading shortcut so you're not typing `size=28, bold=True` on every screen title.

```python
art.Title("Settings")
art.Title("Settings", size=22, color="indigo")
```

---

## Button

```python
art.Button(text, on_click=None, variant="filled", icon=None, width=None, expand=False, loading=None, **kw)
```

`variant` selects which underlying Flet button class gets used:

| `variant` | Underlying control |
|---|---|
| `"filled"` (default) | `ft.FilledButton` |
| `"elevated"` | `ft.ElevatedButton` |
| `"outline"` / `"outlined"` | `ft.OutlinedButton` |
| `"text"` | `ft.TextButton` |

```python
art.Button("Save")
art.Button("Cancel", variant="text")
art.Button("Delete", variant="outline", icon=art.flet.Icons.DELETE)
```

### Automatic loading spinner

Pass a `State` via `loading=` to get an automatic spinner in place of the label while an async `on_click` is running, with the button disabled so it can't be double-tapped:

```python
saving = art.State(False)

async def save(e):
    await art.post_json(url, data={"name": name.value})

art.Button("Save", on_click=save, loading=saving)
```

Artemis flips `saving.value` to `True` right before calling your handler and back to `False` right after — whether the handler succeeded or raised. You can read `saving.value` elsewhere on the same screen too, if you want a loading indicator to appear in more than one place at once.

---

## Input

```python
art.Input(label="", value="", bind=None, field=None, password=False, multiline=False, on_change=None, width=None, **kw)
```

Wraps `ft.TextField`. As covered in [Core Concepts](02-core-concepts.md#why-input-is-the-one-exception), `Input` deliberately does **not** trigger Artemis's usual auto-redraw-after-change behavior, because redrawing on every keystroke would reset your cursor position and drop focus. Instead:

- **`bind=some_state`** — Artemis silently keeps `some_state.value` updated as the user types, with no redraw. Read it whenever you actually need it.

```python
name = art.State("")
art.Input(label="Your name", bind=name)
```

- **`field=some_field`** — same idea, but tied to a `Field` from the validation system (see [Forms & Validation](06-forms-and-validation.md)). The error message shows up under the input automatically once the field has been touched (blurred, or a form submit was attempted):

```python
email = art.Field("", art.validators.email())
art.Input(label="Email", field=email)
```

- **`on_change=...`** — if you'd rather handle changes yourself, pass a plain callback; it's passed straight through to Flet untouched, no wrapping.
- **`password=True`** — masks the input and shows a reveal toggle (`can_reveal_password` is set for you).

You can combine `bind`/`field` with your own `on_change` — both fire; Artemis's own value-syncing happens first, then your callback.

---

## Switch, Checkbox, Slider, Dropdown

```python
art.Switch(label="", value=False, on_change=None, **kw)
art.Checkbox(label="", value=False, on_change=None, **kw)
art.Slider(value=0, min=0, max=100, on_change=None, label=None, **kw)
art.Dropdown(options, value=None, on_change=None, label=None, **kw)
```

All four are thin wrappers that add Artemis's auto-redraw-after-change behavior on top of the equivalent Flet control — unlike `Input`, these are "discrete" interactions (a whole new state in one action, not a stream of keystrokes), so redrawing immediately is the right call and doesn't cause any UX problem.

```python
art.Switch(label="Dark mode", value=dark.value, on_change=lambda e: dark.set(e.control.value))
art.Checkbox(label="I agree to the terms", value=agreed.value, on_change=lambda e: agreed.set(e.control.value))
art.Slider(value=volume.value, min=0, max=100, label="{value}%", on_change=lambda e: volume.set(e.control.value))
```

`Dropdown`'s `options` accepts a plain list of strings (each becomes an option with that string as both value and label) — pass a list of `ft.dropdown.Option(...)` yourself instead if you need distinct display text and values:

```python
art.Dropdown(["Small", "Medium", "Large"], value="Medium", on_change=lambda e: size.set(e.control.value))
```

---

## Layout: Column, Row, Grid

```python
art.Column(controls=None, gap=10, center=False, scroll=False, expand=False, **kw)
art.Row(controls=None, gap=10, center=False, wrap=False, expand=False, **kw)
```

Standard vertical/horizontal layout containers. `gap` is spacing between children (maps to Flet's `spacing`); `center=True` centers children both along the main axis and cross axis; `scroll=True` on `Column` makes it scrollable when content overflows; `wrap=True` on `Row` lets children wrap to a new line instead of overflowing horizontally.

```python
art.Column([
    art.Title("Profile"),
    art.Text("your@email.com", muted=True),
], gap=8, center=True, expand=True)

art.Row([art.Button("Save"), art.Button("Cancel", variant="text")], gap=8)
```

### Grid

```python
art.Grid(items, columns=2, min_item_width=None, gap=10, expand=True, **kw)
```

A "gallery of things" layout — the pattern every product list, photo grid, or card dashboard needs — without manually nesting `Row`s of `Column`s to fake a grid.

```python
art.Grid([art.Card(art.Text(p)) for p in products], columns=3)
```

Pass `min_item_width=` instead of `columns` for a grid that picks its own column count based on available width — useful when the same screen needs to look sensible on both a phone and a wide desktop window:

```python
art.Grid(tiles, min_item_width=160)
```

(These two parameters are mutually exclusive in effect — `min_item_width`, if given, takes precedence, matching Flet's own `GridView(max_extent=...)` vs `GridView(runs_count=...)` behavior underneath.)

---

## Tabs

```python
art.Tabs(tabs, selected=0, on_change=None, expand=True, **kw)
```

**In-page** tabbed content — switching which panel is visible on the *current* screen. This is not navigation between separate screens; for that, see `bottom_nav`/`set_drawer` in [Navigation](05-navigation.md).

```python
art.Tabs([
    ("Overview", art.Text("...")),
    ("Settings", art.Column([...])),
])
```

Each entry is a `(label, content)` tuple. Raw Flet's `Tabs` requires you to construct a separate `TabBar` and `TabBarView` and keep a `length` parameter manually in sync with however many tabs you have — this wrapper does that bookkeeping for you.

---

## Expandable

```python
art.Expandable(title, content, subtitle=None, leading=None, expanded=False, **kw)
```

A collapsible section — wraps `ft.ExpansionTile`. Good for FAQs, grouped settings, "show more" details that shouldn't be visible by default.

```python
art.Expandable(
    "Shipping details",
    art.Text("Ships in 2-3 business days via standard courier."),
    subtitle="Tap to expand",
)
```

`title`/`subtitle` accept plain strings (auto-wrapped in `Text`) or a control directly if you want custom styling. `content` accepts a single control or a list of controls, all shown when expanded.

---

## Badge

```python
art.Badge(label=None, color=None, **kw)
```

**Not a wrapper control** — a value you attach to another control's `badge` property (most Flet controls have one). This mirrors exactly how Flet itself models badges, so it's worth calling out explicitly rather than pretending it's a container:

```python
art.flet.Icon(art.flet.Icons.NOTIFICATIONS, badge=art.Badge("3", color="rose"))
```

Leave `label=None` for a plain dot instead of a number.

---

## Box

```python
art.Box(content=None, pad=16, radius=12, bg=None, gradient=None, glass=False, on_click=None, **kw)
```

The general-purpose "put a rounded, styled box around this" helper — wraps `ft.Container`. Two things it does that raw Flet requires assembling by hand:

**Gradients** — give it a list of two or more colors, and Artemis builds the `LinearGradient` (top-left to bottom-right) for you:

```python
art.Box(art.Text("Premium", color="white"), gradient=["#6366F1", "#EC4899"])
```

**Glass panels** — `glass=True` gives you a frosted-glass effect (blurred backdrop, translucent fill, a faint border) in one flag, instead of manually combining `ft.Blur`, a semi-transparent `bgcolor`, and a border:

```python
art.Box(art.Text("Frosted panel", color="white"), glass=True, pad=20)
```

Looks best sitting on top of an image or a colorful/gradient background — a glass panel on a plain surface just looks like a slightly-transparent gray box, which is accurate to what it actually is.

Pass `on_click=` to make the whole box tappable (Artemis sets Flet's `ink=True` for you so the tap gets a visible ripple, and wraps the handler with Artemis's usual auto-redraw behavior).

---

## Card

```python
art.Card(content=None, pad=16, radius=16, elevation=1, **kw)
```

Wraps `ft.Card` with sensible default padding and radius baked in, so a card doesn't need manual `Container` nesting to get its content away from the edges.

```python
art.Card(art.Column([art.Text("Alice", bold=True), art.Text("alice@example.com", muted=True)]))
```

---

## Spacer, Divider

```python
art.Spacer(size=20)
art.Divider(**kw)
```

`Spacer` is an empty, invisible box of a fixed size — useful inside a `Row`/`Column` where you want deliberate blank space that `gap` alone doesn't cover (e.g., a bigger gap in one specific spot). `Divider` is a thin horizontal rule; pass any Flet `Divider` properties (`color`, `thickness`, ...) through `**kw`.

---

## Image

```python
art.Image(src, width=None, height=None, radius=0, fit="cover", **kw)
```

`fit` accepts `"cover"` (default — fills the box, cropping if needed), `"contain"` (fits entirely within the box, letterboxing if needed), or `"fill"` (stretches to fill exactly, ignoring aspect ratio).

```python
art.Image("https://example.com/photo.jpg", width=200, height=200, radius=12)
```

---

## BottomNav

```python
art.BottomNav(tabs, selected=0, on_change=None, **kw)
```

Builds a `ft.NavigationBar` from a list of `(label, icon)` pairs. In practice you'll rarely call this directly — `app.bottom_nav(...)` (covered in [Navigation](05-navigation.md#bottom-tab-bar)) uses this internally and keeps the bar fixed across screen changes, which is what you actually want for app-wide navigation.

---

## ListTile

```python
art.ListTile(title, subtitle=None, leading=None, trailing=None, on_click=None, **kw)
```

A row with an optional leading icon/avatar, a title, a subtitle, and something on the trailing edge — the bread-and-butter pattern for settings screens, contact lists, notification feeds, todo rows.

```python
art.ListTile(
    title="Alice Kim",
    subtitle="Last seen 2h ago",
    leading=art.Avatar(text="A"),
    trailing=art.flet.IconButton(art.flet.Icons.CHEVRON_RIGHT),
    on_click=lambda e: app.go("/contact/alice"),
)
```

`title`/`subtitle` accept plain strings (auto-wrapped) or controls.

---

## Avatar

```python
art.Avatar(text=None, src=None, size=40, bg=None, **kw)
```

A round avatar. Pass `src` for a photo (a URL or asset path), or just `text` (typically someone's initials) for a plain color circle with that text centered in it. `size` is the diameter in pixels.

```python
art.Avatar(text="AK")
art.Avatar(src="https://example.com/avatar.jpg", size=56)
```

---

## Loader, ProgressBar

```python
art.Loader(size=32, color=None, **kw)
art.ProgressBar(value=None, color=None, **kw)
```

`Loader` is a small spinning ring (`ft.ProgressRing`) — drop it in wherever something's loading. `ProgressBar` is a horizontal bar; leave `value=None` for an indeterminate "still working" animation, or pass a float `0.0`–`1.0` for a real percentage.

```python
art.Row([art.Loader(size=20), art.Text("Loading...")])
art.ProgressBar(value=0.65)
```

---

## Charts

```python
art.LineChart(values, labels=None, color=None, curved=True, height=200, **kw)
art.BarChart(values, labels=None, color=None, height=200, bar_width=20, **kw)
art.PieChart(data, height=200, **kw)
```

**Requires the optional `flet-charts` package** (`pip install flet-charts` or `pip install -e ".[charts]"`). If it isn't installed, calling any of these raises a `RuntimeError` with a clear message telling you to install it — not a confusing import error.

`LineChart`/`BarChart` take a plain list of numbers (Artemis builds the x-axis positions for you) or a list of `(x, y)` tuples if you need non-sequential x-values:

```python
art.LineChart([12, 19, 14, 24, 22, 30], labels=["Jan", "Feb", "Mar", "Apr", "May", "Jun"])
art.BarChart([12, 19, 8], labels=["Q1", "Q2", "Q3"])
```

`PieChart` takes a dict of `label -> value`, or a list of `(label, value)` / `(label, value, color)` tuples if you want to pick the slice colors yourself. Without explicit colors, it cycles through Artemis's own palette:

```python
art.PieChart({"Rent": 1200, "Food": 400, "Fun": 200})
art.PieChart([("A", 10, "#FF0000"), ("B", 20)])
```

Raw `flet-charts` requires manually constructing `LineChartDataPoint`/`BarChartGroup`/`PieChartSection` objects and wiring up axis labels by hand for every tick — these three functions do all of that from plain Python data.

---

## Everything else

Artemis's widget set focuses on the controls a typical small-to-medium app needs most. For anything not listed here — a `SearchBar`, a `Slider` variant, a specific icon, a niche layout control — reach for Flet directly:

```python
art.flet.Icon(art.flet.Icons.STAR, color=art.flet.Colors.AMBER)
```

It mixes into an Artemis control tree with zero friction, because (as covered in [Core Concepts](02-core-concepts.md#everything-is-a-plain-flet-control)) every Artemis widget already *is* a plain Flet control underneath.

## Next

**[Theming & Branding →](04-theming-and-branding.md)**
