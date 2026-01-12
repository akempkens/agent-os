## Python Coding Style Best Practices

### PEP 8 Compliance
- **Follow PEP 8**: Adhere to Python's official style guide (PEP 8) for code layout and formatting
- **Automated Formatting**: Use Black to automatically format code with consistent style (line length: 88-100 characters)
- **Linting**: Use Ruff for fast, comprehensive linting that catches style issues and code smells

### Naming Conventions
- **Variables & Functions**: Use `snake_case` for variables, functions, and methods
- **Classes**: Use `PascalCase` for class names
- **Constants**: Use `UPPER_CASE_WITH_UNDERSCORES` for module-level constants
- **Private Members**: Prefix with single underscore `_private_method` for internal use
- **Name Mangling**: Use double underscore `__private` only when necessary to avoid name conflicts in subclasses
- **Meaningful Names**: Choose descriptive names that reveal intent; avoid abbreviations except for well-known ones (e.g., `id`, `url`, `http`)

### Type Hints & Annotations
- **Always Use Type Hints**: Add type hints to all function signatures and class attributes
- **Modern Type Syntax**: Use modern type hint syntax (Python 3.10+): `list[str]`, `dict[str, int]` instead of `List[str]`, `Dict[str, int]`
- **Type Aliases**: Create type aliases for complex types to improve readability
- **Generic Types**: Use generics when writing reusable functions and classes
- **Return Types**: Always specify return types, including `None` when functions don't return a value
- **Use Protocol**: Prefer Protocol (structural subtyping) over abstract base classes for flexible interfaces

### Docstrings & Documentation
- **Module Docstrings**: Add docstrings to all modules explaining their purpose
- **Function Docstrings**: Use Google-style or NumPy-style docstrings for functions and methods
- **Class Docstrings**: Document classes with their purpose, attributes, and usage examples
- **Type Information**: Don't duplicate type information in docstrings when type hints are present
- **Examples**: Include usage examples in docstrings for complex functions

### Code Organization
- **Small, Focused Functions**: Keep functions small and focused on a single task (typically < 50 lines)
- **Single Responsibility**: Each class and function should have one clear purpose
- **Module Organization**: Group related functionality into modules; keep modules focused (< 500 lines)
- **Import Order**: Organize imports in three groups: standard library, third-party, local (use Ruff to auto-sort)
- **Avoid Star Imports**: Never use `from module import *`; be explicit about imports

### Consistent Indentation & Formatting
- **4 Spaces**: Use 4 spaces for indentation (never tabs)
- **Line Length**: Aim for 88 characters (Black default) or 100 max; break long lines logically
- **Blank Lines**: Use 2 blank lines between top-level functions/classes, 1 blank line between methods
- **Trailing Commas**: Use trailing commas in multi-line data structures for cleaner diffs

### Code Simplicity
- **Pythonic Code**: Write idiomatic Python; use list comprehensions, context managers, and generators appropriately
- **Remove Dead Code**: Delete unused imports, functions, and commented-out code immediately
- **DRY Principle**: Avoid duplication by extracting common logic into reusable functions or classes
- **YAGNI**: Don't add functionality until it's actually needed
- **Avoid Premature Optimization**: Write clear code first; optimize only when necessary and measured

### String Formatting
- **F-strings**: Prefer f-strings for string formatting (Python 3.6+): `f"Hello {name}"`
- **Avoid Concatenation**: Don't use `+` for string concatenation in loops or complex expressions
- **Multiline Strings**: Use triple quotes for multiline strings and docstrings

### Error Handling
- **Specific Exceptions**: Catch specific exception types, never use bare `except:`
- **Exception Chaining**: Use `raise ... from` to chain exceptions and preserve context
- **Early Returns**: Use early returns to avoid deep nesting in conditional logic

### Modern Python Features
- **Context Managers**: Use `with` statements for resource management
- **Dataclasses**: Use `@dataclass` for simple data containers instead of manual `__init__`
- **Enum**: Use `Enum` for fixed sets of constants
- **Pathlib**: Use `pathlib.Path` instead of `os.path` for file operations
- **Walrus Operator**: Use `:=` (assignment expressions) to reduce duplication when appropriate

### Backward Compatibility
- **No Backward Compatibility by Default**: Unless specifically instructed, assume you don't need to maintain backward compatibility
- **Clean Breaks**: Remove deprecated code cleanly rather than adding compatibility shims
- **Version Pinning**: Use Poetry to pin exact dependency versions in lock file
