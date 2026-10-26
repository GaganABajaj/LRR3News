# Life Republic R3 News: Architecture and Python Guide

This guide explains how the project is put together, how a typical piece of
content moves through it, and the Python syntax used to implement that flow. It
is intended for someone who has not worked on this codebase before and is still
learning Python.

The project is a small publishing tool, not a server-backed web application.
Python provides desktop publishing windows, stores content in SQLite, and
generates HTML files. A browser then displays those files using HTML, CSS, and a
small amount of JavaScript.

> **Source of truth:** The code is the authority when details in this guide and
> the running application differ. File names below are relative to the project
> root.

## Contents

1. [The application in one picture](#the-application-in-one-picture)
2. [Project layout](#project-layout)
3. [Typical content journeys](#typical-content-journeys)
4. [The Python language features used here](#the-python-language-features-used-here)
5. [How each Python module works](#how-each-python-module-works)
6. [How the database stores content](#how-the-database-stores-content)
7. [How a static page is generated](#how-a-static-page-is-generated)
8. [How the desktop windows work](#how-the-desktop-windows-work)
9. [Tests and safe ways to experiment](#tests-and-safe-ways-to-experiment)
10. [Configuration and operational notes](#configuration-and-operational-notes)
11. [A practical reading path](#a-practical-reading-path)

## The application in one picture

```text
Person using a desktop tool
          |
          v
root launcher script (for example update_homepage.py)
          |
          v
src/lrr3news/gui.py  <---->  src/lrr3news/groq_client.py
          |                         |
          |                         +---- Groq API, when AI generation is requested
          v
src/lrr3news/database.py
          |
          v
       news.db  (SQLite)
          |
          v
src/lrr3news/site.py
          |
          +----> index.html, archive.html, opinions.html,
          |      advertisements.html, articles/<slug>.html
          v
Browser reads the generated HTML + assets/site.css + local images
```

The important separation is:

- **`gui.py`** collects and validates a person's choices and calls other
  modules. It is the presentation layer.
- **`database.py`** owns SQLite tables, persistence, and database-level
  validation.
- **`groq_client.py`** handles the optional AI draft-generation request.
- **`site.py`** turns Python data into HTML and writes the static site files.
- **The browser** displays those files. It does not query `news.db`.

Because the web site is static, publishing a change means regenerating the
affected HTML files and deploying them. There is no Python web server that runs
for each visitor.

## Project layout

| Path | Purpose |
| --- | --- |
| `src/lrr3news/` | Main Python package. |
| `src/lrr3news/gui.py` | Tkinter editor, creator, and update windows. |
| `src/lrr3news/database.py` | SQLite schema, records, migrations, validation, and queries. |
| `src/lrr3news/site.py` | HTML rendering and writing of generated pages. |
| `src/lrr3news/groq_client.py` | Optional Groq chat-completion request and response validation. |
| `src/lrr3news/__init__.py` | Package entry point; its current `main()` is a small greeting scaffold. |
| `article_editor.py` | Root-level launcher for the article editor. |
| `create_advertisement.py` | Root-level launcher for the advertisement creation window. |
| `update_homepage.py` | Root-level launcher for the homepage updater. |
| `update_archive.py` | Root-level launcher for the archive updater. |
| `update_opinions.py` | Root-level launcher for the opinions updater. |
| `update_advertisements.py` | Root-level launcher for the advertisements updater. |
| `news.db` | Local SQLite database. It is ignored by Git and created/initialized by the application. |
| `index.html`, `archive.html`, `opinions.html`, `advertisements.html` | Generated top-level site pages. |
| `articles/` | Generated individual article pages. |
| `assets/site.css` | Styles shared by the generated pages. |
| `assets/images/` | Local image files referenced by the pages. |
| `tests/` | Automated tests for persistence, rendering, and API response handling. |
| `pyproject.toml` | Python package metadata and console-script declarations. |
| `.env.example` | Example environment-variable file; put real credentials in `.env`, not in source. |

The root launcher scripts are intentionally tiny. They put `src/` on Python's
module search path, import one function from `lrr3news.gui`, and call it only
when the script is run as a program. The main application logic lives in the
package, rather than being duplicated in each launcher.

## Typical content journeys

### Creating an article

1. Run `python article_editor.py`.
2. `gui.article_editor()` initializes the database and opens a Tkinter window.
3. Optionally select **Generate fields with Groq**. The GUI calls
   `groq_client.generate_article_fields()` with the source text and editable
   system prompt.
4. Review and edit the generated fields in the window.
5. Select **Save article**. The GUI builds a Python dictionary and calls
   `database.add_article()`.
6. The database validates the values, creates a unique slug, and inserts the
   article into SQLite.
7. The GUI calls `site.generate_article_page()` to write
   `articles/<slug>.html`.
8. Run `update_homepage.py` and `update_archive.py` to refresh the other pages
   that list articles.

The AI feature is optional; saving a manually written article does not call the
Groq API.

### Updating page order

The homepage, Opinions, and Advertisements updater windows show one content
name per line. A nested callback reads the edited list, maps each exact name
back to a database ID, rejects unknown names, then calls a page generator with
the selected IDs in the requested order. Removing a name hides it from that
generated page; it does not delete the database record.

The archive updater is different: it regenerates the archive from all saved
articles and does not offer a reorder list.

### Creating an advertisement

1. Run `python create_advertisement.py`.
2. The GUI gathers advertisement details, dates, optional image, and optional
   destination link.
3. `database.add_advertisement()` validates and saves the record.
4. Run `python update_advertisements.py` to choose display order and regenerate
   `advertisements.html`.

Advertisement links may be HTTP(S) URLs or relative paths such as
`Advertisement-Instructions.html`. The page generator renders each card as a
link only when its destination passes validation.

## The Python language features used here

The examples in this section are simplified patterns from the project. They
show the meaning of the syntax, not necessarily a full function.

### Imports and modules

An import makes code from another module available:

```python
import sqlite3
from pathlib import Path

from lrr3news.database import add_article
```

- `import sqlite3` makes the module available under its module name, such as
  `sqlite3.connect(...)`.
- `from pathlib import Path` imports `Path` directly, so code can write
  `Path(...)` instead of `pathlib.Path(...)`.
- `from lrr3news.database import add_article` imports one named function from
  this project's package.

Python loads each module once per process and caches it. Keeping database,
rendering, and interface code in separate modules makes those responsibilities
easier to find and test.

### Variables, strings, and f-strings

Names refer to values. Python uses indentation to mark blocks; it does not use
braces around function bodies.

```python
heading = "Community update"
message = f"Saved: {heading}"
```

An f-string starts with `f` and evaluates expressions inside `{}`. The project
uses f-strings both for messages and to build HTML. HTML values that come from
content are escaped with the `_e()` helper before they are inserted into
markup.

Python strings can use single or double quotes. Triple-quoted strings are
helpful for multiline SQL and HTML templates:

```python
sql = """
    SELECT id, heading
    FROM articles
    ORDER BY published_at DESC
"""
```

Adjacent string literals inside parentheses are automatically joined. This
allows long prompts or messages to be formatted across source lines.

### Indentation, conditionals, and loops

Indentation defines the body of an `if`, loop, function, and other compound
statement:

```python
if not heading.strip():
    raise ValueError("Heading is required.")

for tag in tags:
    print(tag)
```

`if` selects a path. `for` repeats a block for each value in an iterable such as
a list. The project uses these for required-field checks, input validation,
database rows, and rendering lists of cards.

### Functions, parameters, and return values

`def` declares a function. Parameters are inputs; `return` sends a value back
to the caller:

```python
def _slugify(value: str) -> str:
    return value.lower().replace(" ", "-")
```

The `: str` annotations say that `value` is expected to be a string and that the
function returns a string. They help readers and editor tools understand the
code; Python does not automatically enforce all annotations at runtime.

Some parameters have defaults:

```python
def get_articles(database_path: Path = DATABASE_PATH) -> list[dict[str, Any]]:
    ...
```

If a caller leaves out `database_path`, the default database file is used. Test
code passes a temporary path instead.

The `*` in a parameter list makes following parameters keyword-only:

```python
def initialize_database(
    database_path: Path = DATABASE_PATH, *, seed_demo: bool = True
) -> None:
    ...
```

Callers can write `initialize_database(path, seed_demo=False)`, but cannot
accidentally pass the flag as a second positional argument. `-> None` means the
function is intended to perform an action and not return a useful value.

### Common data structures

**Lists** keep an ordered sequence:

```python
tags = ["community", "notice"]
```

**Dictionaries** associate keys with values:

```python
article = {
    "heading": "Community update",
    "tags": ["community", "notice"],
}
```

The project passes article and advertisement records around as dictionaries.
The record keys correspond to the values needed by the database or renderer.
`dict[str, Any]` is a type annotation meaning a dictionary with string keys and
values of various types.

**Tuples** are fixed groups of values. The category list is a tuple because the
categories are defined as a stable set. A function can return more than one
value in a tuple:

```python
def example() -> tuple[str, int]:
    return ("October", 10)
```

The syntax `a, b = example()` unpacks those values into two names.

### Comprehensions and generators

A list comprehension builds a list in a compact form:

```python
tags = [tag.strip() for tag in raw_tags if tag.strip()]
```

It means: for each `tag` in `raw_tags`, trim it and include it only if the
trimmed value is non-empty. A set comprehension uses `{...}` and removes
duplicates; the archive uses one to collect unique tags.

`next(generator, None)` returns the first matching item, or `None` if there is
no match. The article page generator uses this pattern to look up one article
by ID.

### Exceptions and validation

An exception represents an error or invalid operation. `raise` creates one:

```python
if not source_text.strip():
    raise ValueError("Enter the source article before generating fields.")
```

Callers handle expected exceptions with `try` and `except`:

```python
try:
    article_id = add_article(record)
except (ValueError, OSError, sqlite3.Error) as error:
    show_error(str(error))
```

`as error` stores the caught exception in a variable so the GUI can show a
useful message. The GUI catches errors at the boundary where it can notify the
person; lower-level functions generally raise errors rather than displaying
dialogs themselves.

The project also uses exception chaining:

```python
try:
    parsed = date.fromisoformat(value)
except ValueError as error:
    raise ValueError("Date must use YYYY-MM-DD format.") from error
```

`from error` preserves the original cause for debugging while presenting a
more relevant message to the caller.

### Context managers and resource cleanup

The `with` statement uses a context manager to perform setup and cleanup
reliably:

```python
with closing(sqlite3.connect(database_path)) as connection:
    with connection:
        connection.execute("INSERT INTO ...", values)
```

`closing(...)` ensures that the database connection closes when the outer
block exits. A SQLite connection context manages a transaction: successful
completion commits; an exception rolls it back. This is safer than manually
remembering to close or commit on every branch.

`TemporaryDirectory()` in tests is also a context-managed resource; test
fixtures clean it up after a test.

### `Path` and filesystem paths

`pathlib.Path` represents filesystem paths:

```python
PROJECT_ROOT = Path(__file__).resolve().parents[2]
database_path = PROJECT_ROOT / "news.db"
```

`__file__` is the current Python file's path. `.resolve()` makes it absolute,
`.parents[2]` walks up to the project root from `src/lrr3news/`, and `/`
combines path parts using the operating system's path rules. This `/` is
Python's overloaded path-joining operator; it does not mean the path string is
hard-coded with Unix separators.

### Type annotations and `Any`

The modules use annotations such as `list[dict[str, Any]]`, `Path`, `str |
None`, and `tuple[str, float]`. These describe expected shapes and make code
easier to navigate in an editor. `Any` is used for values whose exact type is
not specified at that point, such as fields from flexible database records.

`from __future__ import annotations` appears in modules that use modern type
annotations. It postpones their evaluation, which helps with forward
references and annotation handling.

### Nested functions and callbacks

Tkinter buttons accept a function to run when clicked:

```python
def update() -> None:
    ...

ttk.Button(panel, text="Update", command=update)
```

Notice that the code passes `update`, not `update()`. Passing the function gives
Tkinter a callback to invoke later. Several GUI functions define `update`,
`save`, or `generate_fields` inside the outer window-building function. These
nested functions can read widgets and values from the surrounding function's
scope; this is called a closure.

### `if __name__ == "__main__"`

Root launcher scripts end with a guard like:

```python
if __name__ == "__main__":
    homepage_updater()
```

When Python runs a file directly, its `__name__` is `"__main__"`, so the
launcher opens the window. If another module imports that file, the guarded
action does not run as a side effect.

## How each Python module works

### `src/lrr3news/database.py`

This module owns persistence and database-oriented validation.

| Function or value | Responsibility |
| --- | --- |
| `PROJECT_ROOT`, `DATABASE_PATH` | Locate the project directory and default `news.db`. |
| `DEFAULT_IMAGE_PATH` | Default local article image. |
| `ADVERTISEMENT_CATEGORIES` | Supported advertisement category names. |
| `DEMO_ARTICLES`, `DEMO_ADVERTISEMENTS` | Illustrative records inserted into a new, empty database when demo seeding is enabled. |
| `_slugify(value)` | Turns a heading into a lowercase, hyphen-separated ID. |
| `_validated_urls(values)` | Checks that article images are real files under `assets/images/`; returns normalized paths. Despite its name, these are local paths, not web URLs. |
| `initialize_database(path, seed_demo=True)` | Creates missing tables, adds selected missing advertisement columns, validates/migrates stored image paths, and optionally seeds empty tables with sample content. |
| `insert_article(connection, article)` | Validates and inserts an article using an existing SQLite connection; adds a suffix if a heading-derived slug already exists. |
| `add_article(article, path)` | Initializes the database without demo seeding and inserts one article in a transaction. |
| `get_articles(path)` | Reads article rows in publication-date order and decodes JSON tags and image lists into Python lists. |
| `get_opinion_articles(path)` | Filters `get_articles()` to records with an `Editorial` or `Opinion` tag, ignoring case. |
| `get_advertisements(path)` | Reads advertisement rows and returns them as dictionaries. |
| `add_advertisement(ad, path)` | Validates required fields, date windows, destination links, and local images, then inserts an advertisement. |

The database uses parameter placeholders (`?`) for values in SQL queries.
For example, `WHERE id = ?` keeps data separate from the SQL command. Do not
build SQL by concatenating content values into a query.

The `articles` and `advertisements` tables are used by current workflows. The
initializer also creates an `opinions` table, but the current Opinions page
selects from `articles` by tags rather than reading from that table.

### `src/lrr3news/site.py`

This module reads database records, creates HTML strings, and writes generated
files. It has two broad kinds of functions:

1. `render_*()` functions return HTML text.
2. `generate_*()` functions get records, call renderers, and write files.

Important helpers:

| Function | Responsibility |
| --- | --- |
| `_e(value)` | HTML-escapes text and quotes before inserting content into markup. |
| `_date(value)` | Formats parseable publication date-times for display; leaves unparseable input as text. |
| `_excerpt(body, limit)` | Sanitizes/formats body text, strips markup for a preview, and shortens it. |
| `_BodyHTMLSanitizer` | HTML parser that permits a small set of article formatting tags, removes attributes except safe link `href`s, and suppresses script/style/embedded-object contents. |
| `_format_body(body)` | Wraps plain-text paragraphs in `<p>` tags or parses and sanitizes HTML-like article content. |
| `_tag_slug(tag)` | Converts a tag into a lowercase hyphen form for filtering. |
| `_image(article, ...)` | Renders the first validated local image or an empty placeholder. Article pages use a `../` prefix for images. |
| `_tags(article)` | Renders tag labels. |
| `_navigation(active, nested=False)` / `_masthead(...)` | Build the shared site navigation and header, adjusting relative paths for nested article pages. |
| `_document(title, description, content, nested=False)` | Wraps page content in a full HTML document, references the stylesheet and translation widget, and optionally adds Analytics when configured. |
| `_card(article, compact=False)` | Builds a linked article card used on the homepage, archive, and Opinions page. |
| `render_home(...)` | Renders a featured article and homepage story lists; can validate and apply an explicit article order. |
| `render_archive(articles)` | Renders article cards and embeds JavaScript for tag filtering. |
| `render_info_page(...)` | Builds a generic informational/placeholder page. |
| `_advertisement_date(...)`, `_advertisement_live_date(...)` | Format advertisement dates and validate dates used in the browser. |
| `_advertisement_link(value)` | Allows valid HTTP(S) destinations and safe relative paths; rejects unsafe or malformed values. |
| `_advertisement_image_path(value)` | Allows advertisement images under `assets/images/` and rejects parent traversal. |
| `_advertisement_category(value)` | Normalizes known category aliases to the standard category labels. |
| `render_advertisements(advertisements)` | Builds advertisement cards, category tabs, search/sort controls, and client-side filtering code. |
| `render_opinions(articles)` | Renders the selected Editorial/Opinion article cards. |
| `render_article(article)` | Renders one full article page, including its formatted body, tags, disclaimer, and nested navigation. |
| `generate_article_pages(path, output_dir)` | Generates an HTML page for every article. |
| `generate_article_page(article_id, path, output_dir)` | Generates one article page; raises `ValueError` when the article ID is missing. |
| `generate_home_page(...)` | Writes `index.html`. |
| `generate_archive_page(...)` | Writes `archive.html`. |
| `generate_info_pages(...)` | Currently delegates to advertisement page generation. |
| `generate_advertisements_page(...)` | Reads advertisements, optionally checks/reorders their IDs, and writes `advertisements.html`. |
| `generate_opinions_page(...)` | Reads Editorial/Opinion articles, optionally checks/reorders their IDs, and writes `opinions.html`. |

For content safety, ordinary text is escaped instead of interpreted as HTML.
Article body markup is handled separately by `_BodyHTMLSanitizer`, which only
allows specific tags. Image and destination paths are validated before they
become attributes. Keep these safeguards in place when adding new content
fields or HTML features.

### `src/lrr3news/gui.py`

This module builds the windows with Python's standard-library Tkinter toolkit.
Each public function creates a root window, lays out widgets, defines callbacks,
and starts `root.mainloop()`.

| Function | Window / role |
| --- | --- |
| `_show_error(title, error)` | Displays an exception message in a Tkinter error dialog. |
| `article_editor()` | Collects source text and article fields; optionally requests AI-generated fields; saves an article and generates its detail page. |
| `homepage_updater()` | Lets a person reorder or omit homepage article names, then regenerates the homepage. |
| `archive_updater()` | Shows article/tag counts and regenerates the complete archive. |
| `opinions_updater()` | Lets a person reorder or omit tagged Editorial/Opinion articles, then regenerates Opinions. |
| `advertisements_updater()` | Lets a person reorder or omit advertisement names, then regenerates the page without deleting database rows. |
| `advertisement_creator()` | Collects fields and saves a new advertisement. |

Tkinter widgets include `Label` (text), `Entry` (short input), `Text`
(multiline input), `Button` (an action), `Combobox` (a choice list), and
`Scrollbar`. The code uses `.pack()` and `.grid()` to lay out widgets. A window
usually uses one layout method consistently within a given parent frame.

The save and update callbacks catch expected input, filesystem, and SQLite
errors and show them to the person. The article workflow deliberately reports
separately when an article is saved but page generation fails; this makes it
clear that the database write has already happened.

### `src/lrr3news/groq_client.py`

This module is only used for AI-assisted article drafting.

| Function or value | Responsibility |
| --- | --- |
| `GROQ_ENDPOINT`, `DEFAULT_MODEL`, `DEFAULT_ARTICLE_PROMPT` | Configure the OpenAI-compatible Groq endpoint, default model, and instructions for the expected JSON response. |
| `_http_error_message(status, detail)` | Produces a bounded API error explanation, with special guidance for Cloudflare error 1010. |
| `_load_api_key()` | Reads `GROQ_API_KEY` from the process environment or project `.env`; rejects missing/example credentials. |
| `generate_article_fields(source_text, system_prompt=...)` | Validates input, requests a completion, parses JSON, validates required strings and string lists, and returns a dictionary. |

The response is treated as untrusted input. It must be valid JSON with article
text fields (`heading`, `author`, `published_at`, `category`, `body`,
`disclaimer`) and arrays of strings (`tags`, `image_urls`). AI output is a
draft; people should review it before saving or publishing.

### Root launcher scripts and `src/lrr3news/__init__.py`

Each root launcher follows the same small pattern:

```python
from pathlib import Path
import sys

sys.path.insert(0, str(Path(__file__).resolve().parent / "src"))

from lrr3news.gui import homepage_updater

if __name__ == "__main__":
    homepage_updater()
```

`sys.path` is the list of directories Python searches when importing modules.
The launcher adds this repository's `src` directory so `lrr3news` can be
imported when running the script from the project checkout. The
`lrr3news` console command in `pyproject.toml` points to `lrr3news:main`;
currently that function in `__init__.py` prints a greeting and is not one of
the publishing windows.

## How the database stores content

### Articles

Article records have a text slug primary key and fields for heading, author,
publication time, category, body, tags, disclaimer, image paths, and creation
time. Tags and image paths are Python lists in application code, serialized as
JSON strings for SQLite storage:

```python
tags_json = json.dumps(tags, ensure_ascii=False)
tags = json.loads(tags_json)
```

JSON is used here because SQLite has no dedicated list column type. When
`get_articles()` reads a row, it converts the row into a dictionary, removes
the `tags_json` and `image_urls_json` storage columns, and adds decoded `tags`
and `image_urls` lists.

The article ID is based on the heading, for example
`"Garden update"` becomes `"garden-update"`. If that ID already exists,
`insert_article()` tries `garden-update-2`, then `garden-update-3`, and so on.

### Advertisements

Advertisements use an integer ID and store their name, image path, details,
schedule, category, advertiser, badge, destination link, and `override_date`
flag. A SQLite integer is used for the override (`0` or `1`); the Python GUI
collects it as a Boolean.

Start and end dates are inclusive and use `YYYY-MM-DD`. A blank boundary means
the schedule is open on that side. The generated page embeds start/end dates
as data attributes, then browser JavaScript decides whether the ad is live
based on the visitor's local calendar date. `override_date` makes the card pass
that live-date check regardless of its date window.

### Initialization and migration

`initialize_database()` is safe to call repeatedly: it creates tables only if
they are missing and adds certain missing advertisement columns when upgrading
an older database. When asked to seed demo content, it seeds only if the
corresponding table is empty. Functions that add an actual new record disable
demo seeding so an empty database does not unexpectedly become filled with
samples before the new record is inserted.

Database writes use a transaction. If validation or an insert fails, the
transaction is rolled back when the connection context exits with an exception.

## How a static page is generated

Take the homepage as an example:

1. The GUI calls `generate_home_page(...)`.
2. The generator calls `get_articles(database_path)`.
3. `render_home(articles, ...)` selects an order and combines HTML fragments
   such as the masthead, featured story, and cards.
4. `_document(...)` wraps those fragments in the document shell.
5. `Path.write_text(..., encoding="utf-8")` writes the final HTML file.
6. A browser loads the page and then requests its linked CSS, images, and
   optional third-party translation/analytics scripts.

The page renderer generally inserts content using `_e()` so strings become
text, not executable HTML. The article body is a special case because it
supports a controlled set of HTML formatting; `_format_body()` sanitizes that
markup. The generated archive's tag filter and advertisements search/filter/
sort behavior are JavaScript embedded in the static HTML.

Generated paths can be supplied as arguments to generator functions. This
keeps the rendering code testable: tests can write pages into a temporary
directory instead of overwriting the real site.

### HTML, CSS, and JavaScript at a glance

- **HTML** describes page structure: headings, paragraphs, links, images,
  forms, buttons, and sections. Python produces that HTML as strings.
- **CSS** controls presentation: spacing, colors, typography, layout, and
  responsive behavior. The shared stylesheet is `assets/site.css`.
- **JavaScript** runs in the browser. The archive uses it to filter by tag;
  the advertisements page uses it to filter by date, category, and search text,
  and to sort cards.

JavaScript is not required to generate the files. It runs later, in a visitor's
browser, to provide interactive filtering and sorting.

## How the desktop windows work

Tkinter is event-driven. `root.mainloop()` starts the event loop, which waits
for actions such as button clicks and runs the associated callback. The
application does not usually run one continuous loop that polls the fields.

For example, an updater builds a text box from database names. When its Update
button is clicked, the callback:

1. Reads the text with `.get("1.0", "end")`.
2. Splits it into lines and trims whitespace.
3. Maps each exact name to one or more database IDs.
4. Displays an error and stops if a name is unknown.
5. Calls the generator with the requested ID order.
6. Updates a status label and displays a success dialog.

Duplicate names are handled by storing a list of IDs for each name and taking
the next matching ID each time. This is why the lookup maps values to lists,
not just single IDs.

## Tests and safe ways to experiment

The tests use Python's `unittest` framework. `StaticSiteTests.setUp()` creates a
temporary folder and an isolated SQLite database for each test; `tearDown()`
cleans it up. This prevents tests from modifying the real `news.db` or
generated site files.

The suite checks behaviors such as:

- Saving articles and connecting them to generated pages.
- Escaping unsafe text and sanitizing allowed article HTML.
- Rejecting images outside the local images folder.
- Generating/reordering pages and handling unknown IDs.
- Advertisement validation, dates, links, categories, and rendering.
- API prompt handling and error messages using mocks rather than live API
  calls.

From PowerShell at the project root, run:

```powershell
$env:PYTHONPATH = "src"
python -m unittest discover -s tests -v
```

`PYTHONPATH` tells Python to look in `src/` for the package when the test
runner imports `lrr3news`. Root launcher scripts add that path themselves.
Tests should not need a real Groq API key; the Groq client tests replace the
network client with a mock.

When adding behavior, a useful workflow is:

1. Find the existing function and a nearby test.
2. Add or update a test that describes the expected behavior.
3. Make a focused change in the module that owns that responsibility.
4. Run the focused test or full suite.
5. Inspect generated output or a test assertion if output format matters.

## Configuration and operational notes

- **Python version:** `pyproject.toml` declares Python 3.12 or newer.
- **Tkinter:** The desktop interfaces need Tkinter, which is usually included
  with standard Python installers.
- **Groq API key:** Article drafting reads `GROQ_API_KEY` from the environment
  first, then from `.env`. Keep real secrets out of Git and never paste them
  into source files.
- **Groq model:** `GROQ_MODEL` can override the default model selected in
  `groq_client.py`.
- **Google Analytics:** `GA_MEASUREMENT_ID` can be read from the environment or
  `.env`. The document generator checks its `G-...` format before adding the
  Analytics script.
- **Images:** Article images must exist under `assets/images/`. Advertisement
  images are likewise expected under that folder.
- **External services:** Translation scripts are included in generated pages.
  Analytics is included only when configured. Groq is called only when the AI
  generation action is requested.
- **Dependency declaration:** `groq_client.py` imports the third-party `openai`
  SDK, but the current `pyproject.toml` has an empty `dependencies` list. A
  fresh environment may therefore need that SDK installed separately for the
  Groq-enabled article editor; the project metadata does not currently install
  it automatically.
- **Package console command:** The `lrr3news` script currently calls the
  greeting-only `main()` in `__init__.py`. The dedicated editor/updater
  commands are separately declared in `pyproject.toml`.

## A practical reading path

If you want to understand or change a feature, follow its end-to-end path:

1. Start with its root launcher, for example `update_homepage.py`.
2. Find the matching public function in `src/lrr3news/gui.py`.
3. Follow the callback to the database read/write function in
   `src/lrr3news/database.py`.
4. Follow the page generator and renderer in `src/lrr3news/site.py`.
5. Read the matching test in `tests/test_site.py`.

For a new content field, trace all of its representations: GUI widget, Python
dictionary key, database column or JSON field, renderer output, and tests. For
an interaction or appearance change, also inspect `assets/site.css` and the
JavaScript embedded in the appropriate page renderer.

The project deliberately keeps these layers small and explicit. A good
first-time contribution usually changes one layer's behavior, updates the
directly related tests, and avoids mixing UI, SQL, and HTML-generation logic
inside one function.
