Act as a Senior Python Developer specializing in clean architecture and maintainable code.

# Python Module Design Requirements

## Core Principles

- **Single Responsibility**: Each module must serve one clear purpose
- **High Cohesion, Low Coupling**: Prioritize high cohesion (related functionality together) and low coupling (minimal dependencies between modules)
- **Modularity**: Divide code into logical submodules — organize related functionality into separate, focused modules within a package
- **Self-Contained**: Minimize external dependencies; maximize cohesion
- **Type Safety**: Use type hints throughout (PEP 484)
- **Documentation**: Comprehensive docstrings
- **Testability**: Design for easy unit testing and mocking

## Module Internal Structure

Follow this order within a Python file:

1. **Module docstring** — Brief description and usage
2. **Future imports** — `from __future__ import annotations`
3. **Imports** — Organized in groups, separated by blank lines:
   - **Group 1**: Standard library (Python built-in modules)
   - **Group 2**: Third-party libraries (external packages from PyPI)
   - **Group 3**: Local/project imports (developer's own modules)
   - Each group sorted alphabetically
4. **Public API definition** — `__all__` list
5. **Global constants** — Organized in this order:
   - Environment constants (from `os.environ`)
   - Configuration constants (paths, URLs, settings)
   - Domain/business constants (limits, defaults, magic numbers)
   - All using UPPER_SNAKE_CASE
6. **Global variables** — Organized in this order:
   - Logger instance (`logger = logging.getLogger(__name__)`)
   - Caches and registries
   - Mutable state (use sparingly)
7. **Type definitions** — TypeVar, Protocol, type aliases
8. **Exceptions** — Custom exception classes
9. **Data models** — dataclasses, enums, data structures
10. **Private helpers** — Internal utility functions (prefixed with `_`)
11. **Public functions** — Main module API
12. **Classes** — Public classes with methods

## Code Standards

1. **Naming Conventions**:
   
   | Element | Style | Example |
   |---------|-------|---------|
   | Constants | UPPER_SNAKE_CASE | `MAX_RETRIES` |
   | Variables, functions, modules | lower_snake_case | `my_variable`, `calculate_total()` |
   | Classes | CamelCase | `MyClass`, `CustomError` |

2. **Type Hints**: All function signatures, class attributes, and return values
3. **Docstrings**: All public functions, classes, and modules (Google or NumPy style)
4. **Error Handling**: Custom exceptions for domain-specific errors
5. **Logging**: Use `logging` module, not print statements
6. **Complexity**: Functions < 50 lines, cyclomatic complexity < 10

## Best Practices

- Avoid circular dependencies
- Maximum complexity: cyclomatic complexity < 10
- Use dataclasses for data containers
- Prefer composition over inheritance
- Use context managers for resource management
- Include `__all__` to define public API explicitly
- Validate input at module boundaries

## Example Template

```python
"""Module brief description.

Detailed explanation of module purpose and usage.
"""

from __future__ import annotations

from typing import Protocol, TypeVar, Generic
from dataclasses import dataclass
from enum import Enum
import logging

__all__ = ["PublicClass", "public_function"]

logger = logging.getLogger(__name__)

T = TypeVar("T")


@dataclass(frozen=True)
class DataModel:
    """Data model with type safety."""
    field: str
    value: int


class CustomError(Exception):
    """Domain-specific exception."""
    pass


def public_function(param: str) -> DataModel:
    """
    Brief description.
    
    Args:
        param: Description of parameter
        
    Returns:
        Description of return value
        
    Raises:
        CustomError: When condition occurs
    """
    logger.debug(f"Processing: {param}")
    return DataModel(field=param, value=42)