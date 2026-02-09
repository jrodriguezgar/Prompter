# Role: Senior Python Configuration Architect

Generate a centralized configuration module following the Nono framework pattern with multi-source resolution.

## Requirements

1. **Class-based design** → Single `Config` class with `@classmethod` methods
2. **Priority resolution** → Environment variables > Programmatic > Config file > Defaults
3. **Type hints** → Full typing with `Union`, `Optional`, `Path`, `Dict`, `Any`
4. **Lazy loading** → Config file loaded only when first accessed
5. **Thread-safe** → Class-level state with proper initialization guards
6. **Convenience functions** → Module-level `get_*` and `set_*` wrappers

## Input Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `{module_name}` | Name of the module/project | `MyApp` |
| `{config_items}` | List of configurable paths/values | `templates_dir, prompts_dir, cache_dir` |
| `{env_prefix}` | Environment variable prefix | `MYAPP_` |
| `{config_format}` | Config file format | `toml`, `yaml`, `json` |

## Structure Template

```python
"""
{module_name} Configuration Module

Environment Variables:
    {env_prefix}{ITEM}_DIR: Path to {item} directory
    {env_prefix}CONFIG_FILE: Path to config file

Usage:
    from {module_name.lower()}.config import {module_name}Config, get_{item}_dir
    
    # Option 1: Environment variables
    os.environ["{env_prefix}{ITEM}_DIR"] = "/path/to/{item}"
    
    # Option 2: Programmatic
    {module_name}Config.set_{item}_dir("/path/to/{item}")
    
    # Option 3: Custom config file
    {module_name}Config.load_from_file("/path/to/config.{config_format}")
"""

import os
import logging
from pathlib import Path
from typing import Any, Dict, Optional, Union

logger = logging.getLogger("{module_name}.Config")

# Config file parser import (adapt for format)
try:
    import tomllib  # Python 3.11+
except ImportError:
    import tomli as tomllib

# Module-level defaults
_MODULE_DIR = Path(__file__).parent
_DEFAULT_{ITEM}_DIR = _MODULE_DIR / "{item}"
_DEFAULT_CONFIG_FILE = _MODULE_DIR / "config.{config_format}"


class {module_name}Config:
    """Central configuration with priority: env > programmatic > file > default."""
    
    # Class-level storage
    _{item}_dir: Optional[Path] = None
    _config_data: Dict[str, Any] = {}
    _config_loaded: bool = False
    
    @classmethod
    def set_{item}_dir(cls, path: Union[str, Path]) -> None:
        path = Path(path)
        if not path.is_absolute():
            path = Path.cwd() / path
        cls._{item}_dir = path
        logger.info(f"{item} directory set to: {{path}}")
    
    @classmethod
    def get_{item}_dir(cls) -> Path:
        # 1. Environment variable
        env_path = os.environ.get("{env_prefix}{ITEM}_DIR")
        if env_path:
            return Path(env_path) if Path(env_path).is_absolute() else Path.cwd() / env_path
        
        # 2. Programmatic
        if cls._{item}_dir is not None:
            return cls._{item}_dir
        
        # 3. Config file
        cls._ensure_config_loaded()
        config_path = cls._config_data.get("paths", {{}}).get("{item}_dir", "")
        if config_path:
            path = Path(config_path)
            return path if path.is_absolute() else _MODULE_DIR.parent / path
        
        # 4. Default
        return _DEFAULT_{ITEM}_DIR
    
    @classmethod
    def load_from_file(cls, config_path: Union[str, Path]) -> None:
        config_path = Path(config_path)
        if not config_path.exists():
            logger.warning(f"Config file not found: {{config_path}}")
            return
        with open(config_path, "rb") as f:
            cls._config_data = tomllib.load(f)
        cls._config_loaded = True
    
    @classmethod
    def _ensure_config_loaded(cls) -> None:
        if cls._config_loaded:
            return
        env_config = os.environ.get("{env_prefix}CONFIG_FILE")
        config_path = Path(env_config) if env_config else _DEFAULT_CONFIG_FILE
        if config_path.exists():
            try:
                with open(config_path, "rb") as f:
                    cls._config_data = tomllib.load(f)
            except Exception as e:
                logger.warning(f"Failed to load config: {{e}}")
        cls._config_loaded = True
    
    @classmethod
    def reset(cls) -> None:
        cls._{item}_dir = None
        cls._config_data = {{}}
        cls._config_loaded = False
    
    @classmethod
    def get_all_config(cls) -> Dict[str, Any]:
        cls._ensure_config_loaded()
        return {{
            "{item}_dir": str(cls.get_{item}_dir()),
            "config_file": cls._config_data,
            "env_{item}_dir": os.environ.get("{env_prefix}{ITEM}_DIR"),
        }}


# Convenience functions
def get_{item}_dir() -> Path:
    return {module_name}Config.get_{item}_dir()

def set_{item}_dir(path: Union[str, Path]) -> None:
    {module_name}Config.set_{item}_dir(path)
```

## Constraints

- Use `pathlib.Path` exclusively (no `os.path`)
- Logging via `logging` module (no print statements)
- Support Python 3.10+ with tomllib fallback to tomli
- Class methods only (no instance state)
- Docstrings in Google format

## Output Format

Complete Python module with:
1. Module docstring with usage examples
2. Imports and constants
3. Config class with all methods
4. Convenience wrapper functions
5. `__all__` export list

## Example Invocation

```
Generate a config module for:
- Module name: DataProcessor
- Config items: data_dir, output_dir, cache_dir
- Env prefix: DP_
- Config format: toml
```

Provide production-ready Python code following PEP 8 and type annotation best practices.
