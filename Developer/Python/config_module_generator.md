# Role: Senior Python Configuration Architect

Generate a unified configuration module with multi-source resolution and instance-based design for flexibility and testability.

## Requirements

1. **Instance-based design** → `Config` class with instance methods (not @classmethod)
2. **Priority resolution** → Arguments > Environment > File > Defaults
3. **Type hints** → Full typing with `Union`, `Optional`, `Path`, `Dict`, `Any`, `TypeVar`
4. **Multi-format support** → JSON, YAML, TOML with auto-detection
5. **Validation** → Schema-based configuration validation
6. **Method chaining** → Fluent API for configuration loading

## Input Variables

| Variable          | Description                 | Example    |
| ----------------- | --------------------------- | ---------- |
| `{module_name}` | Name of the module/project  | `myapp`  |
| `{env_prefix}`  | Environment variable prefix | `MYAPP_` |

## Structure Template

```python
"""
Configuration Module

Multi-source configuration with priority resolution.

Priority Order (highest to lowest):
    1. Command-line arguments
    2. Environment variables
    3. Configuration files (JSON, YAML, TOML)
    4. Default values

Usage:
    from myapp.config import Config, load_config
  
    # Quick setup
    config = load_config(filepath='config.json')
  
    # Full control with method chaining
    config = Config(defaults={'timeout': 30})
    config.load_file('config.json').load_env(prefix='MYAPP_')
  
    # Access values
    host = config.get('host')
    timeout = config.get('timeout', type=int)
  
    # Dictionary-style access
    value = config['key']
    config['key'] = 'value'
  
    # Validation with schema
    schema = ConfigSchema()
    schema.add_field('host', type=str, required=True)
    schema.add_field('port', type=int, min_value=1, max_value=65535)
  
    config = Config(schema=schema)
    config.load_file('config.json')
    config.validate()  # Raises ValueError if invalid
"""

from __future__ import annotations

import os
import json
import copy
from pathlib import Path
from typing import Any, Dict, List, Optional, Union, TypeVar, Type
from dataclasses import dataclass, field
from enum import Enum

__all__ = [
    'Config',
    'ConfigSchema',
    'ConfigValue',
    'ConfigSource',
    'ConfigFormat',
    'load_config',
    'create_sample_config',
]

T = TypeVar('T')

# Config file names to search (priority order)
STANDARD_CONFIG_NAMES = [
    'config.json',
    'config.yaml',
    'config.yml',
    'config.toml',
]


class ConfigSource(Enum):
    """Sources for configuration values."""
    DEFAULT = "default"
    FILE = "file"
    ENVIRONMENT = "environment"
    ARGUMENT = "argument"


class ConfigFormat(Enum):
    """Supported configuration file formats."""
    JSON = "json"
    YAML = "yaml"
    TOML = "toml"


@dataclass
class ConfigValue:
    """Configuration value with source metadata."""
    value: Any
    source: ConfigSource = ConfigSource.DEFAULT
    key: str = ""


@dataclass
class ConfigSchema:
    """
    Schema for configuration validation.
  
    Example:
        schema = ConfigSchema()
        schema.add_field('host', type=str, required=True)
        schema.add_field('port', type=int, min_value=1, max_value=65535)
        schema.add_field('mode', type=str, choices=['dev', 'prod'])
    """
  
    @dataclass
    class Field:
        name: str
        type: Type = str
        required: bool = False
        default: Any = None
        choices: List[Any] = field(default_factory=list)
        min_value: Optional[float] = None
        max_value: Optional[float] = None
      
        def validate(self, value: Any) -> tuple[bool, Optional[str]]:
            if value is None:
                if self.required:
                    return False, f"Required field '{self.name}' is missing"
                return True, None
            if self.choices and value not in self.choices:
                return False, f"Field '{self.name}' must be one of: {self.choices}"
            if isinstance(value, (int, float)):
                if self.min_value is not None and value < self.min_value:
                    return False, f"Field '{self.name}' must be >= {self.min_value}"
                if self.max_value is not None and value > self.max_value:
                    return False, f"Field '{self.name}' must be <= {self.max_value}"
            return True, None
  
    fields: List[Field] = field(default_factory=list)
  
    def add_field(self, name: str, **kwargs) -> ConfigSchema:
        """Add a field to the schema. Returns self for chaining."""
        self.fields.append(ConfigSchema.Field(name=name, **kwargs))
        return self
  
    def validate(self, config: Dict[str, Any]) -> tuple[bool, List[str]]:
        """Validate config dict against schema. Returns (is_valid, errors)."""
        errors = []
        for field_def in self.fields:
            value = config.get(field_def.name, field_def.default)
            is_valid, error = field_def.validate(value)
            if not is_valid:
                errors.append(error)
        return len(errors) == 0, errors


class Config:
    """
    Unified configuration manager with multi-source support.
  
    Example:
        config = Config(defaults={'timeout': 30})
        config.load_file('config.json').load_env(prefix='MYAPP_')
      
        host = config.get('host')
        timeout = config.get('timeout', type=int)
    """
  
    def __init__(
        self, 
        defaults: Dict[str, Any] = None,
        schema: ConfigSchema = None,
        auto_discover: bool = False
    ):
        """
        Initialize configuration manager.
      
        Args:
            defaults: Default values dictionary
            schema: Optional schema for validation
            auto_discover: Whether to search for config files in cwd
        """
        self.schema = schema
      
        # Storage for values from different sources
        self._defaults: Dict[str, ConfigValue] = {}
        self._file_values: Dict[str, ConfigValue] = {}
        self._env_values: Dict[str, ConfigValue] = {}
        self._arg_values: Dict[str, ConfigValue] = {}
      
        # Set defaults
        if defaults:
            for key, value in defaults.items():
                self._defaults[key] = ConfigValue(
                    value=value, source=ConfigSource.DEFAULT, key=key
                )
      
        if auto_discover:
            self._auto_discover()
  
    def _auto_discover(self) -> None:
        """Search for config files in current directory."""
        for config_name in STANDARD_CONFIG_NAMES:
            config_file = Path.cwd() / config_name
            if config_file.exists():
                self.load_file(str(config_file))
                return
  
    def load_file(self, filepath: str, format: ConfigFormat = None) -> Config:
        """
        Load configuration from file. Returns self for chaining.
      
        Args:
            filepath: Path to configuration file
            format: File format (auto-detected from extension if not specified)
        """
        path = Path(filepath)
        if not path.exists():
            return self
      
        # Auto-detect format from extension
        if format is None:
            ext = path.suffix.lower()
            format = {
                '.json': ConfigFormat.JSON,
                '.yaml': ConfigFormat.YAML,
                '.yml': ConfigFormat.YAML,
                '.toml': ConfigFormat.TOML,
            }.get(ext, ConfigFormat.JSON)
      
        try:
            with open(path, 'r', encoding='utf-8') as f:
                if format == ConfigFormat.JSON:
                    data = json.load(f)
                elif format == ConfigFormat.YAML:
                    import yaml
                    data = yaml.safe_load(f)
                elif format == ConfigFormat.TOML:
                    try:
                        import tomllib
                    except ImportError:
                        import tomli as tomllib
                    data = tomllib.loads(f.read())
                else:
                    data = json.load(f)
          
            self._flatten_and_store(data, self._file_values, ConfigSource.FILE)
        except Exception:
            pass  # Silent fail, config is optional
      
        return self
  
    def _flatten_and_store(
        self, 
        data: Dict[str, Any], 
        storage: Dict[str, ConfigValue],
        source: ConfigSource, 
        prefix: str = ""
    ) -> None:
        """Flatten nested dictionary and store as ConfigValues."""
        for key, value in data.items():
            full_key = f"{prefix}.{key}" if prefix else key
            if isinstance(value, dict):
                self._flatten_and_store(value, storage, source, full_key)
            else:
                storage[full_key] = ConfigValue(value=value, source=source, key=full_key)
  
    def load_env(self, prefix: str) -> Config:
        """
        Load configuration from environment variables. Returns self for chaining.
      
        Mapping: PREFIX_KEY -> key, PREFIX_NESTED__KEY -> nested.key
      
        Args:
            prefix: Environment variable prefix (e.g., 'MYAPP_')
        """
        for key, value in os.environ.items():
            if key.startswith(prefix):
                config_key = key[len(prefix):].replace('__', '.').lower()
                parsed_value = self._parse_env_value(value)
                self._env_values[config_key] = ConfigValue(
                    value=parsed_value, source=ConfigSource.ENVIRONMENT, key=config_key
                )
        return self
  
    def _parse_env_value(self, value: str) -> Any:
        """Parse environment variable value, detecting type."""
        # Boolean
        if value.lower() in ('true', 'false', 'yes', 'no'):
            return value.lower() in ('true', 'yes')
        # Integer
        try:
            return int(value)
        except ValueError:
            pass
        # Float
        try:
            return float(value)
        except ValueError:
            pass
        # JSON array/object
        if value.startswith(('[', '{')):
            try:
                return json.loads(value)
            except json.JSONDecodeError:
                pass
        return value
  
    def load_args(self, args: Union[Dict[str, Any], object]) -> Config:
        """
        Load configuration from parsed arguments. Returns self for chaining.
      
        Args:
            args: Dictionary or argparse.Namespace with argument values
        """
        if hasattr(args, '__dict__'):
            args = vars(args)
        for key, value in args.items():
            if value is not None:
                self._arg_values[key] = ConfigValue(
                    value=value, source=ConfigSource.ARGUMENT, key=key
                )
        return self
  
    def get(self, key: str, default: Any = None, type: Type[T] = None) -> T:
        """
        Get configuration value using priority resolution.
      
        Priority: Arguments > Environment > File > Default
      
        Args:
            key: Configuration key (use dot notation for nested: 'db.host')
            default: Default value if not found
            type: Type to coerce value to (e.g., int, bool)
        """
        sources = [self._arg_values, self._env_values, self._file_values, self._defaults]
      
        for source in sources:
            if key in source:
                value = source[key].value
                if type is not None:
                    try:
                        return type(value)
                    except (ValueError, TypeError):
                        pass
                return value
        return default
  
    def set(self, key: str, value: Any) -> Config:
        """Set a configuration value (highest priority). Returns self for chaining."""
        self._arg_values[key] = ConfigValue(value=value, source=ConfigSource.ARGUMENT, key=key)
        return self
  
    def require(self, key: str, message: str = None) -> Any:
        """Get required value, raising ValueError if not found."""
        value = self.get(key)
        if value is None:
            raise ValueError(message or f"Required config '{key}' not found")
        return value
  
    def all(self) -> Dict[str, Any]:
        """Get all configuration values as dictionary."""
        result = {}
        for source in [self._defaults, self._file_values, self._env_values, self._arg_values]:
            for key, cv in source.items():
                result[key] = cv.value
        return result
  
    def validate(self, raise_on_error: bool = True) -> tuple[bool, List[str]]:
        """
        Validate configuration against schema.
      
        Args:
            raise_on_error: If True, raises ValueError on validation failure
          
        Returns:
            Tuple of (is_valid, list of error messages)
        """
        if not self.schema:
            return True, []
        is_valid, errors = self.schema.validate(self.all())
        if not is_valid and raise_on_error:
            raise ValueError("Validation failed:\n" + "\n".join(f"  - {e}" for e in errors))
        return is_valid, errors
  
    def copy(self) -> Config:
        """Create a deep copy of this configuration."""
        new = Config(schema=self.schema, auto_discover=False)
        new._defaults = copy.deepcopy(self._defaults)
        new._file_values = copy.deepcopy(self._file_values)
        new._env_values = copy.deepcopy(self._env_values)
        new._arg_values = copy.deepcopy(self._arg_values)
        return new
  
    # Dictionary-style access
    def __getitem__(self, key: str) -> Any:
        return self.get(key)
  
    def __setitem__(self, key: str, value: Any):
        self.set(key, value)
  
    def __contains__(self, key: str) -> bool:
        return self.get(key) is not None
  
    def __repr__(self):
        return f"Config(keys={len(self.all())})"


# ============================================================================
# UTILITY FUNCTIONS
# ============================================================================

def load_config(
    filepath: str = None,
    defaults: Dict[str, Any] = None,
    env_prefix: str = None
) -> Config:
    """
    Convenience function to create and load configuration.
  
    Args:
        filepath: Path to config file (optional)
        defaults: Default values dictionary
        env_prefix: Environment variable prefix (e.g., 'MYAPP_')
  
    Example:
        config = load_config(filepath='config.json', env_prefix='MYAPP_')
    """
    config = Config(defaults=defaults, auto_discover=False)
    if filepath:
        config.load_file(filepath)
    if env_prefix:
        config.load_env(prefix=env_prefix)
    return config


def create_sample_config(filepath: str) -> bool:
    """
    Create a sample configuration file.
  
    Args:
        filepath: Output file path (e.g., 'config.json')
    """
    sample = {
        'app': {
            'name': 'myapp',
            'debug': False
        },
        'server': {
            'host': 'localhost',
            'port': 8080
        },
        'database': {
            'host': 'localhost',
            'port': 5432,
            'name': 'mydb'
        },
        'timeout': 30
    }
    try:
        with open(filepath, 'w', encoding='utf-8') as f:
            json.dump(sample, f, indent=2)
        return True
    except Exception:
        return False
```

## Common Configuration Sections (Optional)

The following are typical configuration sections that can be adapted based on project needs:

### Database Connection

```json
{
    "database": {
        "driver": "postgresql",
        "host": "localhost",
        "port": 5432,
        "name": "appdb",
        "user": "app_user",
        "password": "",
        "pool_size": 5,
        "timeout": 30,
        "ssl": false
    }
}
```

### API Configuration

```json
{
    "api": {
        "base_url": "https://api.example.com/v1",
        "api_key": "",
        "timeout": 30,
        "retry_attempts": 3,
        "rate_limit": 100
    }
}
```

### Logging Settings

```json
{
    "logging": {
        "level": "INFO",
        "format": "%(asctime)s - %(name)s - %(levelname)s - %(message)s",
        "file": "app.log",
        "max_size_mb": 10,
        "backup_count": 5,
        "console": true
    }
}
```

### Cache Configuration

```json
{
    "cache": {
        "enabled": true,
        "backend": "redis",
        "host": "localhost",
        "port": 6379,
        "ttl_seconds": 3600,
        "prefix": "myapp:"
    }
}
```

### Security Settings

```json
{
    "security": {
        "secret_key": "",
        "token_expiry_hours": 24,
        "allowed_hosts": ["localhost", "127.0.0.1"],
        "cors_origins": ["http://localhost:3000"],
        "ssl_verify": true
    }
}
```

### Email/SMTP Configuration

```json
{
    "email": {
        "smtp_host": "smtp.example.com",
        "smtp_port": 587,
        "use_tls": true,
        "username": "",
        "password": "",
        "from_address": "noreply@example.com"
    }
}
```

### Storage/Files

```json
{
    "storage": {
        "provider": "local",
        "base_path": "./data",
        "max_file_size_mb": 50,
        "allowed_extensions": [".pdf", ".xlsx", ".csv"]
    }
}
```

### Application Paths

```json
{
    "paths": {
        "data_dir": "./data",
        "logs_dir": "./logs",
        "temp_dir": "./tmp",
        "config_dir": "./config",
        "output_dir": "./output",
        "templates_dir": "./templates",
        "assets_dir": "./assets"
    }
}
```

### Queue/Message Broker

```json
{
    "queue": {
        "broker": "rabbitmq",
        "host": "localhost",
        "port": 5672,
        "vhost": "/",
        "exchange": "myapp_exchange"
    }
}
```

> **Note**: These sections are optional templates. Include only what the project requires and adapt field names to match the application's conventions.

## Constraints

- Use `pathlib.Path` exclusively (no `os.path`)
- Support Python 3.10+ with tomllib fallback to tomli
- Instance methods for flexibility (multiple configs allowed)
- Method chaining for fluent API
- Docstrings following PEP 257 conventions
- Type hints on all public methods
- Return `(success: bool, errors: List[str])` tuples for failure reporting

## Key Design Decisions

| Aspect                      | Choice   | Rationale                                |
| --------------------------- | -------- | ---------------------------------------- |
| Instance instead of Singleton | Instance | Testability, multiple configs, isolation |
| Method chaining             | Yes      | Fluent API, cleaner code                 |
| Dictionary access           | Yes      | Familiar Python idiom                    |
| Auto-discovery              | Optional | Convention over configuration            |

## Output Format

Generate the following files for the project:

### 1. `config.py` - Main Configuration Module

Self-contained and project-specific module:

- Located in the project's main package directory (e.g., `myapp/config.py`)
- Importable as `from myapp.config import Config, load_config`

Complete Python module with:

1. Module docstring with usage examples
2. Future imports and type hints
3. `__all__` export list
4. Enums and dataclasses
5. Main Config class with instance methods
6. Utility functions
7. Sample config generator

### 2. `config_example.py` - Usage Example

Practical demonstration file showing:

- How to instantiate and configure
- Loading from file, environment, and arguments
- Accessing values with different methods
- Schema validation example
- Error handling patterns

```python
"""
Example usage of the configuration module.

Run: python config_example.py --debug --port 9000
"""
from myapp.config import Config, load_config, ConfigSchema

# Basic usage
config = Config()
config.set_defaults({'app.name': 'demo', 'server.port': 8080})
config.load_file('config.json')
config.load_env(prefix='MYAPP_')

# Access values
print(f"App: {config['app.name']}")
print(f"Port: {config.get('server.port', 8080)}")

# With schema validation
schema = ConfigSchema({
    'server.port': {'type': int, 'required': True, 'min_value': 1024},
    'app.debug': {'type': bool, 'default': False}
})
validated_config = Config(schema=schema)
validated_config.load_file('config.json')

# Quick load helper
quick_config = load_config(
    filepath='config.json',
    env_prefix='MYAPP_',
    defaults={'timeout': 30}
)
```

### 3. `README_config.md` - Documentation

Comprehensive documentation including:

```markdown
# Configuration Module

## Overview

Configuration management for {module_name} with multi-source priority resolution.

## Installation

The module is included in the project. No additional dependencies required for JSON/TOML.
For YAML support: `pip install pyyaml`

## Quick Start

\```python
from {module_name}.config import Config, load_config

# Simple usage
config = load_config(filepath='config.json', env_prefix='{env_prefix}')
print(config['database.host'])
\```

## Priority Resolution

Values are resolved in order (first match wins):

1. **Arguments** - Command line or programmatic args
2. **Environment** - Variables with prefix (e.g., `{env_prefix}DATABASE_HOST`)
3. **File** - JSON, YAML, or TOML configuration files
4. **Defaults** - Programmatically defined defaults

## Configuration File Formats

### JSON (config.json)

\```json
{
    "database": {"host": "localhost", "port": 5432},
    "api": {"timeout": 30}
}
\```

### TOML (config.toml)

\```toml
[database]
host = "localhost"
port = 5432

[api]
timeout = 30
\```

## Environment Variables

Prefix: `{env_prefix}`
Nested keys use double underscore: `{env_prefix}DATABASE__HOST`

## Schema Validation

\```python
schema = ConfigSchema({
    'database.port': {'type': int, 'required': True, 'min_value': 1},
    'api.timeout': {'type': int, 'default': 30}
})
config = Config(schema=schema)
\```

## API Reference

| Method | Description |
|--------|-------------|
| `Config()` | Create new instance |
| `load_file(path)` | Load from file |
| `load_env(prefix)` | Load from environment |
| `get(key, default)` | Get value with fallback |
| `set(key, value)` | Set argument-level value |
| `all()` | Get all resolved values |
| `validate()` | Validate against schema |
```
