# Crystal Architecture

English | [日本語](./README.ja.md)

**Crystal Architecture** is a module design guideline for keeping Python projects extensible, readable, and reusable as they grow.

It organizes an entire project into a crystal-like, consistent structure through local placement rules and one-way dependencies.

The goal is to keep responsibilities readable from the file tree, module names, types, and public APIs even as the codebase grows, reducing cognitive load as much as possible.

## Concept

Crystal Architecture values the following ideas:

* The overall functionality of a project should be understandable from the file tree.
* The responsibility of a class, function, type, or constant should be understandable from its file path.
* The boundary between public APIs and internal implementation should be clear.
* Dependencies between modules should remain one-way.
* As much meaning as possible should be expressed through names, types, and file structure.
* Comments and docstrings should be limited to important information that cannot be expressed by code alone.

Crystal Architecture is not a framework.

It does not introduce a specific runtime library. It is an opinionated architecture guideline for designing, organizing, and maintaining Python packages.

## Core Principles

### 1. One File, One Public Element

A single file should define only one public class, function, type, or constant.

```txt
user/
  _models/
    user.py          # User
  _types/
    user_id.py       # UserId
  _helpers/
    create_user.py   # create_user
```

Private helper functions and internal implementation details may live in the same file.

This rule makes the relationship between file names and public elements clear, making responsibilities easier to read from the file tree.

### 2. Implement Features as Subpackages

Do not implement features directly in the top-level package. Split them into feature-based subpackages.

```txt
src/
  example/
    user/
    project/
    storage/
```

The top-level package is treated as the namespace for the entire project.

Actual functionality is placed in subpackages such as `example.user` or `example.storage`.

### 3. Make Internal Implementation Directories Explicit

Internal implementation directories inside a subpackage should generally use an `_` prefix.

```txt
user/
  __init__.py
  _helpers/
  _models/
  _schemas/
  _services/
  _types/
  _utils/
```

This makes the boundary between public APIs and internal implementation easy to understand.

Elements used externally are explicitly exported from the subpackage's `__init__.py`.

### 4. Collect Public APIs at the Top Level of Each Subpackage

Users of a subpackage should import public APIs from the subpackage top level instead of depending directly on internal directories.

```python
from example.user import User, UserId, create_user
```

Internal implementation paths should remain changeable when needed.

```python
# Avoid
from example.user._models.user import User
from example.user._helpers.create_user import create_user
```

When referencing another subpackage's public API, use an **absolute import, not a relative one**. Reserve relative imports for elements within the same subpackage. This way the form of the import statement alone tells you whether you're reaching into your own internals or into another package's public API.

```python
# Another package's public surface → absolute import
from example.storage import Storage

# Same package's internals → relative import
from ._models.user import User
```

```python
# Avoid: walking into another package's public surface via a relative import
from ...storage import Storage
```

### 5. Keep Dependencies One-Way

Modules inside a subpackage may depend only on lower layers.

```txt
_helpers
  ↓
_instances
  ↓
_models / _services / _operations / _exceptions
  ↓
_settings / _constants
  ↓
_schemas / _views
  ↓
_enums / _types / _utils
```

Avoid dependencies in the opposite direction.

When dependencies are needed within the same layer, keep them one-way as well so they do not become circular.

However, as an exception, referencing `_schemas` or `_models` from `_types` is permitted when defining a Union type of multiple defined schemas or models.

### 6. Express Meaning Through Names, Types, and Paths

Avoid relying too much on comments or docstrings. Express as much meaning as possible through names, types, and file paths.

```python
from typing import TypeAlias

AgentName: TypeAlias = str
```

```python
agent_name: AgentName
```

Comments and docstrings should be limited to important information that cannot be expressed by code, types, or names alone.

**When to introduce a type alias:**

A type alias like `AgentName` is not something to add unconditionally. Introduce one only when **a value loses its meaning at the point it is assigned or used**. Two signals guide the decision:

* **The raw type doesn't convey the meaning.** A bare `str` or `tuple[str, str]` doesn't tell you *what* string it is, and you'd otherwise need a comment to explain it.
* **It isn't confined to a short local scope.** Fields, dict keys, and index types (e.g. `dict[tuple[AgentName, AgentKey], Agent]`) appear all over a file, far from their declaration, so their origin gets diluted. Those are exactly the values worth carrying meaning through a type. A local variable whose meaning is obvious from the line above it does not need one.

The job of a type alias is to **carry meaning across scope** — think of it as promoting a comment into the type. Conversely, don't over-apply it to values that read fine inline (`priority: int` and the like).

**Write docstrings at the definition:**

When you do attach a docstring to a type or a field, write it right after the definition (on the line following the assignment). It then shows up as documentation for that symbol itself in IDE autocomplete and hover.

```python
AgentName: TypeAlias = str
"""The name that uniquely identifies an agent."""
```

If a field's type alias already carries the explanation, don't repeat a docstring on the field — it would duplicate the type's.

### 7. Keep Comments and Docstrings Minimal

Do not write comments or docstrings for things you can learn **by reading the code, its types, its names, or the rest of the file**. Reserve them for what you can only know by looking at another file or module — design intent, threading or ordering constraints, an external protocol, or the contract with callers.

```python
# Avoid: restating what the body already says
def resolve_content_type(filename: str, content_type: str | None) -> str:
    """Use the explicit one if given, else guess from the extension, else the default."""
    return content_type or guess_type(filename)[0] or DEFAULT_CONTENT_TYPE
```

```python
# Good: keep only the caller constraint that the body cannot reveal
def sweep_expired(self) -> list[str]:
    """Call on the worker thread — it frees framework memory, so it must not
    be called from the event loop."""
    ...
```

And even for what remains, **prefer expressing it through types and names** to cut prose. If a type alias (Principle 6) or a meaningful parameter or function name can carry the meaning, reach for that before writing a docstring. A docstring is the last resort.

## Single Responsibility and Signs for Splitting

"One file, one responsibility" and "One subpackage, one responsibility."
These might seem contradictory at first glance, but **it is important to adhere to both, simply at different levels of granularity**.

A single file should have only one responsibility, and similarly, a single subpackage should encapsulate only one feature (domain boundary).

When this "single responsibility" is broken, the project starts showing the following symptoms:

* It becomes difficult to come up with a clear, fitting file or package name.
* The file tree within a subpackage becomes bloated and visually messy.
* Dependencies within or between subpackages become complex and tangled.

These symptoms are evidence that the file is taking on too many responsibilities, or the subpackage is hoarding too many features.
Treat these as **clear signs that a file or subpackage needs to be split**, and perform refactoring accordingly.

## Standard File Structure

A subpackage generally follows this structure:

```txt
example/user/
  __init__.py
  _i18n.py
  _settings.py
  _constants/
    default_role.py
  _enums/
    user_status.py
  _exceptions/
    user_not_found_error.py
  _helpers/
    create_user.py
  _instances/
    default_user_service.py
  _models/
    user.py
  _operations/
    normalize_user_name.py
  _schemas/
    user_config.py
  _services/
    user_service.py
  _types/
    user_id.py
  _utils/
    generate_user_id.py
  _views/
    user_view.py
```

Each directory has the following role:

| Directory     | Role                                                |
| ------------- | --------------------------------------------------- |
| `_helpers`    | Functions that expose subpackage functionality externally |
| `_models`     | Data-centric classes that contain business logic    |
| `_services`   | Business logic organized as classes or modules      |
| `_operations` | Function-based internal logic                       |
| `_schemas`    | Data-centric classes without business logic         |
| `_views`      | Data-centric classes representing inputs/outputs for functions and operations |
| `_settings`   | Settings such as Pydantic Settings                  |
| `_constants`  | Constants                                           |
| `_exceptions` | Exception classes                                   |
| `_instances`  | Default or shared instances created from classes    |
| `_enums`      | Enum definitions                                    |
| `_types`      | Type definitions such as TypeAlias, TypeVar, and TypedDict |
| `_utils`      | Pure, stateless, reusable utilities                 |
| `_i18n.py`    | Internationalized string definitions                |

You do not need to create every directory all the time.

Add only the responsibilities that are needed.

## Role of `__init__.py`

`__init__.py` defines the public API of a subpackage.

```python
from ._helpers.create_user import create_user
from ._models.user import User
from ._types.user_id import UserId

__all__ = [
    "User",
    "UserId",
    "create_user",
]
```

When lazy imports are needed, public APIs can be resolved lazily with `__getattr__`.

```python
from importlib import import_module
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from ._helpers.create_user import create_user
    from ._models.user import User
    from ._types.user_id import UserId

__all__ = [
    "User",
    "UserId",
    "create_user",
]

def __getattr__(name: str) -> object:
    if name not in __all__:
        raise AttributeError(f"module {__name__} has no attribute {name}")

    module_map = {
        "User": "._models.user",
        "UserId": "._types.user_id",
        "create_user": "._helpers.create_user",
    }

    globals()[name] = getattr(import_module(module_map[name], __name__), name)
    return globals()[name]
```

## Plugin Pattern

When using a plugin pattern, separate the abstraction layer from the implementation layer.

```txt
example/storage/
  __init__.py
  _settings.py
  _helpers/
    create_storage.py
  _models/
    base_storage.py
  _types/
    storage.py
    storage_name.py

example/storage_impl/
  local/
    __init__.py
    _models/
      local_storage.py
  s3/
    __init__.py
    _settings.py
    _helpers/
      create_s3_storage.py
    _models/
      s3_storage.py
```

`example.storage` is the stable abstraction layer.

`example.storage_impl` is the extension area for adding concrete implementations.

Implementation classes are resolved dynamically by import path, and users depend on the abstraction layer.

## Supporting Both Sync and Async APIs

When providing both synchronous and asynchronous versions, separate the public APIs while collecting shared logic in `_core`.

```txt
example/client/
  __init__.py
  asyncio.py
  _settings.py
  _core/
    _helpers/
    _schemas/
  _sync/
    _helpers/
  _async/
    _helpers/
```

The synchronous version is used like this:

```python
from example.client import get_client
```

The asynchronous version is used like this:

```python
from example.client.asyncio import get_client
```

## When Using Pydantic

When defining Pydantic models as user-configurable values, actively utilize the model's own description and the title and description of Fields.

When using Pydantic models as entities, prefer field docstrings that are easy to inspect through IDE completion over `Field(description=...)`.

However, if the class name, field name, and type information are enough to make the meaning clear, omit the docstring itself.

In Crystal Architecture, it is assumed that maintainers will understand the codebase by reading the code itself rather than automatically generated documentation.

## Test Structure

Test code should mirror the file structure of the target subpackage.

```txt
src/
  example/
    user/
      _helpers/
        create_user.py

tests/
  example/
    user/
      _helpers/
        test_create_user.py
```

Tests should be written as functions.

As a rule, aim for 100% line coverage.

However, for branches that are only for type-checking workarounds or are practically unreachable in the runtime environment, add `# pragma: no cover` to the implementation instead of forcing tests for them.

## Projects Where This Fits Well

Crystal Architecture fits projects such as:

* Long-term maintained Python packages
* Libraries developed by multiple people
* SDKs
* CLI tools
* Core packages for business applications
* Projects with plugin extensions
* Projects that provide both synchronous and asynchronous APIs

## Cases Where This May Be Unnecessary

Crystal Architecture may be excessive for cases such as:

* One-off scripts
* Small PoCs
* Disposable experimental code
* Very small personal tools with few files

Crystal Architecture is not a rule for all Python code.

It is a design guideline for codebases that are expected to grow, be reused, and be maintained.

## Documentation

Detailed documentation will be added under `docs/`.

```txt
docs/
  en/
  ja/
```

Main documents:

* Philosophy
* Principles
* Module Architecture
* Dependency Rules
* Public API
* Plugin Pattern
* Pydantic
* Sync and Async
* Testing
* Comments and Docstrings
* Glossary

## Examples

Implementation examples are placed in `examples/`.

```txt
examples/
  basic-package/
  plugin-package/
  sync-async-package/
```

## Templates

Reusable templates are placed in `templates/`.

```txt
templates/
  python-package/
```

In the future, this can evolve into `copier`, `cookiecutter`, a CLI, import rule checkers, and similar tools.

## License

This project is intended to be published as OSS.

See the repository `LICENSE`.

## Philosophy

Programming is Elegant.
