# FallGuysDocs — Contributor Guide

Documentation site for the **Intro to RL Course (Fall Guys)**.

---

## What is Material for MkDocs?

[Material for MkDocs](https://squidfunk.github.io/mkdocs-material/) is a powerful, production-ready documentation theme built on top of [MkDocs](https://www.mkdocs.org/). It turns plain Markdown files into a beautiful, searchable documentation site, complete with syntax highlighting, tabbed content, callout boxes, dark/light mode, and much more.

In short: you write Markdown, Material for MkDocs makes it look great.

---

## Prerequisites

Make sure you have **[uv](https://docs.astral.sh/uv/)** installed. `uv` is a fast Python package manager used to manage this project's dependencies.

```bash
# Install uv (if you haven't already)
curl -LsSf https://astral.sh/uv/install.sh | sh
```

---

## Setting Up the Environment

Clone the repo and activate the virtual environment.

**macOS / Linux**
```bash
source .venv/bin/activate
```

**Windows**
```powershell
.venv\Scripts\activate
```

If the `.venv` folder doesn't exist yet, create it first:
```bash
uv sync
```

This will install all dependencies (including `mkdocs-material`) listed in `pyproject.toml`.

---

## Serving the Docs Locally

Once your environment is activated, run:

```bash
mkdocs serve
```

Then open your browser at **[http://127.0.0.1:8000](http://127.0.0.1:8000)**. The site hot-reloads whenever you save a file — no need to restart.

---

## Adding New Content

All documentation lives in the `docs/` folder as Markdown (`.md`) files.

### Adding a New Page

1. Create a new `.md` file inside `docs/` (or a subdirectory):
   ```
   docs/my-new-topic.md
   ```

2. Add it to the navigation in `mkdocs.yml` under the appropriate section:
   ```yaml
   nav:
     - Welcome: index.md
     - Introduction:
       - Reinforcement Learning Theory: introduction/rl-theory.md
       - ML Agent Theory: introduction/ml-agent-theory.md
     - Installation:
       - MacOS: installation/macos.md
       - Windows: installation/windows.md
       - Tensorboard: installation/tensorboard.md
     - Levels:
       - Level 0: levels/level-0.md
       # ...
   ```

3. Start writing! The filename becomes the URL slug.

---

### Adding Code Blocks

Code blocks use triple backticks with a language identifier for syntax highlighting.
We use [Pygments](https://pygments.org/docs/lexers/#lexers-for-javascript-and-related-languages) for language support — nearly every language is supported.

#### Basic code block with language highlighting

````markdown
```python
def reward(state, action):
    return 1 if action == optimal(state) else -1
```
````

#### With line numbers

Add `linenums="1"` to enable line numbers (starting from line 1):

````markdown
```python linenums="1"
def reward(state, action):
    return 1 if action == optimal(state) else -1
```
````

#### With highlighted lines

Use `hl_lines` to draw attention to specific lines:

````markdown
```python linenums="1" hl_lines="2 3"
def reward(state, action):
    if action == optimal(state):
        return 1
    return -1
```
````

You can also highlight a range: `hl_lines="2-4"`.

> **Common language identifiers:** `python`, `javascript`, `typescript`, `bash`, `yaml`, `json`, `html`, `css`, `sql`, `rust`, `go`
> See the full list at [pygments.org/docs/lexers](https://pygments.org/docs/lexers/#lexers-for-javascript-and-related-languages).

---

### Adding Tabs

Tabs are great for presenting the same concept at different depths. Use the `=== "Tab Name"` syntax from `pymdownx.superfences`.

**Recommended structure:** use three depth tiers for every concept so readers can engage at their own level.

````markdown
=== "Just the Basics"

    A simple, high-level explanation here. Keep it jargon-free.

=== "Good Understanding"

    Go a layer deeper. Introduce key terms and light formalism.

=== "Deep Knowledge"

    Full technical depth and implementation details.

````

> **Tip:** Every section of the docs should have all three tabs. The last tab ("Deep Knowledge") is where you can go as technical as you like — readers who just want the basics can safely ignore it.

---

### Adding Callouts (Admonitions)

Callouts highlight important information. Use the `!!! type "Title"` syntax.

**Supported types:** `note`, `abstract`, `info`, `tip`, `success`, `question`, `warning`, `failure`, `danger`, `bug`, `example`, `quote`
→ Full list at [squidfunk.github.io/mkdocs-material/reference/admonitions/#supported-types](https://squidfunk.github.io/mkdocs-material/reference/admonitions/#supported-types)

#### Examples

```markdown
!!! tip "Pro Tip"
    Use a small learning rate (e.g. `0.001`) when your reward signal is noisy.

!!! warning "Watch Out"
    Setting `gamma = 1.0` can cause divergence in non-episodic environments.

!!! note
    This section assumes you're familiar with Markov Decision Processes.

!!! info "Further Reading"
    Check out Sutton & Barto's *Reinforcement Learning: An Introduction* for the full theory.
```

> **Convention for this project:** Use `tip` and `hint`-style callouts to surface practical advice and gotchas. Avoid overusing `warning` or `danger` — save those for genuinely risky or confusing areas.

#### Collapsible callouts

Add a `?` instead of `!` to make a callout collapsible:

```markdown
??? tip "Click to reveal a hint"
    Try initialising Q-values optimistically (e.g. all zeros → all ones) to encourage exploration.
```

---

## Project Structure

```
FallGuysDocs/
├── docs/                        # All Markdown source files
│   ├── index.md                 # Welcome / home page
│   ├── introduction/
│   │   ├── rl-theory.md
│   │   └── ml-agent-theory.md
│   ├── installation/
│   │   ├── macos.md
│   │   ├── windows.md
│   │   └── tensorboard.md
│   ├── levels/
│   │   ├── level-0.md … level-5.md
│   └── assets/                  # Images, logos, custom CSS
│       ├── logo.png
│       ├── favicon.ico
│       └── custom.css
├── mkdocs.yml                   # Site configuration & navigation
├── pyproject.toml               # Python dependencies (mkdocs-material)
└── uv.lock                      # Locked dependency versions
```

---

## Quick Reference

| Task | Command |
|---|---|
| Install dependencies | `uv sync` |
| Activate environment (mac/linux) | `source .venv/bin/activate` |
| Activate environment (windows) | `.venv\Scripts\activate` |
| Serve docs locally | `mkdocs serve` |
| Build static site | `mkdocs build` |
