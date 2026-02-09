# Role: Senior Python CLI Developer

Generate a robust command-line interface module following Python best practices with argparse extensions, colored output, and progress indicators.

## Requirements

1. **Class-based design** → Single `CLIBase` class with argument groups and output utilities
2. **Type hints** → Full typing with `Optional`, `List`, `Dict`, `Any`, `Union`
3. **Colored output** → ANSI codes with Windows compatibility (ctypes/colorama fallback)
4. **Progress indicators** → Progress bars, spinners, and formatted tables
5. **Parameter files** → Support for `@file.txt` syntax to load arguments from files
6. **Dry-run mode** → Built-in simulation capability without making changes
7. **Confirmation prompts** → Interactive user confirmation for destructive operations
8. **Logging integration** → Verbosity levels mapped to Python logging

## Input Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `{module_name}` | Name of the CLI module | `my_tool_cli` |
| `{tool_name}` | Name of the tool for help text | `MyTool` |
| `{version}` | Version string | `1.0.0` |
| `{description}` | Brief tool description | `Export data to multiple formats` |
| `{connection_type}` | Type of connection args | `database`, `ldap`, `api`, `none` |
| `{operation_type}` | Primary operation type | `export`, `import`, `sync`, `audit` |
| `{output_formats}` | Supported output formats | `csv,excel,json,sqlite` |

## Module Structure

```python
"""
{module_name}.py - Command Line Interface for {tool_name}

{description}

Usage:
    python -m {module_name} --help
    python -m {module_name} @params.txt
    python -m {module_name} --source file.csv --format json --output result.json

Author: Your Name
Version: {version}
"""

from __future__ import annotations

import sys
import os
import argparse
import json
import logging
from typing import Optional, Dict, Any, List, Union
from dataclasses import dataclass
from enum import Enum
from datetime import datetime

__all__ = [
    "CLIBase",
    "OutputFormat",
    "Colors",
    "print_success",
    "print_error",
    "print_warning",
    "print_info",
    "print_table",
    "print_summary",
    "print_progress",
    "confirm_action",
    "create_cli",
]

logger = logging.getLogger(__name__)
logger.addHandler(logging.NullHandler())


# ============================================================================
# CONSTANTS AND ENUMS
# ============================================================================

class OutputFormat(Enum):
    """Supported output formats for CLI display."""
    TABLE = "table"
    JSON = "json"
    CSV = "csv"
    SUMMARY = "summary"
    QUIET = "quiet"


class LogLevel(Enum):
    """Logging verbosity levels."""
    DEBUG = "debug"
    INFO = "info"
    WARNING = "warning"
    ERROR = "error"
    QUIET = "quiet"


class Colors:
    """ANSI color codes with Windows compatibility."""
    RESET = "\033[0m"
    BOLD = "\033[1m"
    RED = "\033[91m"
    GREEN = "\033[92m"
    YELLOW = "\033[93m"
    BLUE = "\033[94m"
    MAGENTA = "\033[95m"
    CYAN = "\033[96m"
    WHITE = "\033[97m"
    GRAY = "\033[90m"
    
    SUCCESS = GREEN
    ERROR = RED
    WARNING = YELLOW
    INFO = CYAN
    
    @classmethod
    def disable(cls) -> None:
        """Disable colors for non-TTY or unsupported terminals."""
        for attr in dir(cls):
            if not attr.startswith('_') and isinstance(getattr(cls, attr), str):
                setattr(cls, attr, '')
    
    @classmethod
    def init(cls) -> None:
        """Initialize colors with Windows ANSI support."""
        if sys.platform == 'win32':
            try:
                import ctypes
                kernel32 = ctypes.windll.kernel32
                kernel32.SetConsoleMode(kernel32.GetStdHandle(-11), 7)
            except Exception:
                try:
                    import colorama
                    colorama.init()
                except ImportError:
                    cls.disable()
        if not sys.stdout.isatty():
            cls.disable()


Colors.init()


# ============================================================================
# OUTPUT UTILITIES
# ============================================================================

def cprint(message: str, color: str = "", bold: bool = False, file=sys.stdout) -> None:
    """Print colored message to terminal."""
    prefix = ""
    if bold:
        prefix += Colors.BOLD
    if color:
        prefix += color
    suffix = Colors.RESET if (bold or color) else ""
    print(f"{prefix}{message}{suffix}", file=file)


def print_success(message: str) -> None:
    """Print success message with checkmark."""
    cprint(f"✓ {message}", Colors.SUCCESS)


def print_error(message: str, file=sys.stderr) -> None:
    """Print error message with X mark."""
    cprint(f"✗ {message}", Colors.ERROR, file=file)


def print_warning(message: str) -> None:
    """Print warning message with warning sign."""
    cprint(f"⚠ {message}", Colors.WARNING)


def print_info(message: str) -> None:
    """Print info message with info symbol."""
    cprint(f"ℹ {message}", Colors.INFO)


def print_header(title: str, width: int = 70, char: str = "=") -> None:
    """Print formatted section header."""
    print()
    cprint(char * width, Colors.CYAN, bold=True)
    cprint(f" {title}", Colors.CYAN, bold=True)
    cprint(char * width, Colors.CYAN, bold=True)


def print_table(
    headers: List[str], 
    rows: List[List[Any]], 
    max_col_width: int = 40
) -> None:
    """Print formatted ASCII table."""
    if not headers or not rows:
        return
    
    col_widths = []
    for i, header in enumerate(headers):
        max_width = len(str(header))
        for row in rows:
            if i < len(row):
                max_width = max(max_width, len(str(row[i])))
        col_widths.append(min(max_width, max_col_width))
    
    def truncate(value, width):
        s = str(value)
        return s[:width-3] + "..." if len(s) > width else s
    
    header_row = " | ".join(truncate(h, w).ljust(w) for h, w in zip(headers, col_widths))
    separator = "-+-".join("-" * w for w in col_widths)
    
    cprint(header_row, Colors.CYAN, bold=True)
    print(separator)
    
    for row in rows:
        row_str = " | ".join(
            truncate(row[i] if i < len(row) else "", w).ljust(w) 
            for i, w in enumerate(col_widths)
        )
        print(row_str)


def print_summary(stats: Dict[str, Any], title: str = "SUMMARY", width: int = 70) -> None:
    """Print formatted summary statistics."""
    print()
    cprint("=" * width, Colors.CYAN, bold=True)
    cprint(f" {title}", Colors.CYAN, bold=True)
    cprint("=" * width, Colors.CYAN, bold=True)
    
    for key, value in stats.items():
        key_display = key.replace('_', ' ').title()
        
        if 'error' in key.lower() and value > 0:
            value_color = Colors.ERROR
        elif 'success' in key.lower() or 'created' in key.lower():
            value_color = Colors.SUCCESS
        elif 'warning' in key.lower() or 'skipped' in key.lower():
            value_color = Colors.WARNING
        else:
            value_color = Colors.WHITE
        
        print(f"  {key_display + ':':<20} ", end="")
        cprint(str(value), value_color)
    
    cprint("=" * width, Colors.CYAN, bold=True)


def print_progress(
    current: int, 
    total: int, 
    prefix: str = "", 
    suffix: str = "", 
    width: int = 40
) -> None:
    """Print progress bar."""
    if total == 0:
        percent, filled = 100, width
    else:
        percent = (current / total) * 100
        filled = int(width * current // total)
    
    bar = "█" * filled + "-" * (width - filled)
    print(f"\r{prefix} |{bar}| {percent:.1f}% {suffix}", end="", flush=True)
    
    if current >= total:
        print()


def confirm_action(message: str, default: bool = False) -> bool:
    """Prompt user for confirmation."""
    suffix = " [Y/n]" if default else " [y/N]"
    response = input(f"{message}{suffix}: ").strip().lower()
    
    if not response:
        return default
    return response in ('y', 'yes', 'si', 's')


# ============================================================================
# CLI CONFIGURATION
# ============================================================================

@dataclass
class CLIConfig:
    """Configuration for CLI behavior and appearance."""
    prog_name: str = "{module_name}"
    version: str = "{version}"
    description: str = "{description}"
    epilog: str = ""
    
    colors_enabled: bool = True
    default_output_format: OutputFormat = OutputFormat.SUMMARY
    default_log_level: LogLevel = LogLevel.INFO
    
    allow_parameter_files: bool = True
    require_confirmation: bool = False
    dry_run_by_default: bool = False
    
    default_timeout: int = 30
    default_page_size: int = 1000


# ============================================================================
# CLI BASE CLASS
# ============================================================================

class CLIBase:
    """Base class for CLI applications with consistent behavior.
    
    Usage:
        cli = CLIBase(prog="{module_name}", description="{description}", version="{version}")
        cli.add_output_group()
        args = cli.parse_args()
        cli.print_summary({{'processed': 100, 'errors': 0}})
    """
    
    def __init__(
        self, 
        prog: str = None, 
        description: str = "", 
        version: str = "1.0.0",
        epilog: str = None, 
        config: CLIConfig = None
    ):
        self.config = config or CLIConfig(
            prog_name=prog or os.path.basename(sys.argv[0]),
            version=version,
            description=description,
            epilog=epilog or ""
        )
        
        self.parser = argparse.ArgumentParser(
            prog=prog,
            description=description,
            epilog=epilog,
            formatter_class=argparse.RawTextHelpFormatter,
            fromfile_prefix_chars='@' if self.config.allow_parameter_files else None
        )
        
        self.parser.add_argument('--version', '-V', action='version', version=f'%(prog)s {version}')
        self._add_global_arguments()
        
        self._groups: Dict[str, argparse._ArgumentGroup] = {}
        self.args = None
        self.stats: Dict[str, int] = {}
        self.start_time: datetime = None
    
    def _add_global_arguments(self) -> None:
        """Add global arguments available to all CLI tools."""
        grp = self.parser.add_argument_group("Global Options")
        
        grp.add_argument('--verbose', '-v', action='count', default=0,
                         help="Increase verbosity (-v=INFO, -vv=DEBUG)")
        grp.add_argument('--quiet', '-q', action='store_true',
                         help="Suppress non-error output")
        grp.add_argument('--no-color', action='store_true',
                         help="Disable colored output")
        grp.add_argument('--dry-run', action='store_true',
                         default=self.config.dry_run_by_default,
                         help="Simulate without making changes")
        grp.add_argument('--output-format', choices=[f.value for f in OutputFormat],
                         default=self.config.default_output_format.value,
                         help="Output display format")
        grp.add_argument('--config-file', type=str, metavar="FILE",
                         help="Load configuration from JSON file")
        grp.add_argument('--log-file', type=str, metavar="FILE",
                         help="Write logs to file")
    
    def add_group(self, name: str, title: str = None, description: str = None) -> argparse._ArgumentGroup:
        """Add a custom argument group."""
        group = self.parser.add_argument_group(title or name.title(), description)
        self._groups[name] = group
        return group
    
    # -------------------------------------------------------------------
    # CONNECTION ARGUMENT GROUPS (customize based on {connection_type})
    # -------------------------------------------------------------------
    
    def add_database_connection_group(self) -> argparse._ArgumentGroup:
        """Add database connection arguments."""
        group = self.add_group("db_connection", "Database Connection")
        group.add_argument('--db-type', required=True, 
                           choices=['postgresql', 'mysql', 'sqlite', 'oracle', 'mssql'],
                           help="Database type")
        group.add_argument('--db-host', help="Database host")
        group.add_argument('--db-port', type=int, help="Database port")
        group.add_argument('--db-name', required=True, help="Database name")
        group.add_argument('--db-user', help="Database username")
        group.add_argument('--db-password', help="Database password")
        group.add_argument('--db-password-file', metavar="FILE",
                           help="Read password from file")
        return group
    
    def add_ldap_connection_group(self) -> argparse._ArgumentGroup:
        """Add LDAP/Active Directory connection arguments."""
        group = self.add_group("ldap_connection", "LDAP Connection")
        group.add_argument('--host', '-H', required=True, help="LDAP server hostname")
        group.add_argument('--user', '-U', required=True, help="Bind DN or user principal")
        group.add_argument('--password', '-P', required=True, help="Bind password")
        group.add_argument('--password-file', metavar="FILE", help="Read password from file")
        group.add_argument('--base-dn', '-b', required=True, help="Search base DN")
        group.add_argument('--no-ssl', action='store_true', help="Disable SSL/TLS")
        group.add_argument('--auth-method', choices=['SIMPLE', 'NTLM', 'KERBEROS'],
                           default='SIMPLE', help="Authentication method")
        return group
    
    def add_api_connection_group(self) -> argparse._ArgumentGroup:
        """Add REST API connection arguments."""
        group = self.add_group("api_connection", "API Connection")
        group.add_argument('--api-url', required=True, help="API base URL")
        group.add_argument('--api-key', help="API key")
        group.add_argument('--api-key-file', metavar="FILE", help="Read API key from file")
        group.add_argument('--client-id', help="OAuth client ID")
        group.add_argument('--client-secret', help="OAuth client secret")
        group.add_argument('--token', help="Bearer token")
        group.add_argument('--timeout', type=int, default=30, help="Request timeout (seconds)")
        return group
    
    # -------------------------------------------------------------------
    # OPERATION ARGUMENT GROUPS (customize based on {operation_type})
    # -------------------------------------------------------------------
    
    def add_export_group(self, formats: List[str] = None) -> argparse._ArgumentGroup:
        """Add export configuration arguments."""
        group = self.add_group("export", "Export Configuration")
        formats = formats or [{output_formats}]
        
        group.add_argument('--format', '-f', required=True, choices=formats, help="Output format")
        group.add_argument('--output', '-o', required=True, help="Output file/connection")
        group.add_argument('--filter', help="Filter expression")
        group.add_argument('--select-fields', help="Comma-separated fields to include")
        group.add_argument('--limit', type=int, help="Maximum records to export")
        return group
    
    def add_import_group(self, formats: List[str] = None) -> argparse._ArgumentGroup:
        """Add import configuration arguments."""
        group = self.add_group("import", "Import Configuration")
        formats = formats or ['csv', 'excel', 'json']
        
        group.add_argument('--source', '-s', required=True, help="Source file path")
        group.add_argument('--format', '-f', required=True, choices=formats, help="Source format")
        group.add_argument('--skip-validation', action='store_true', help="Skip data validation")
        group.add_argument('--update-existing', action='store_true', help="Update existing records")
        group.add_argument('--batch-size', type=int, default=100, help="Batch size for processing")
        return group
    
    def add_sync_group(self) -> argparse._ArgumentGroup:
        """Add synchronization configuration arguments."""
        group = self.add_group("sync", "Sync Configuration")
        group.add_argument('--source', '-s', required=True, help="Source connection/file")
        group.add_argument('--target', '-t', required=True, help="Target connection/file")
        group.add_argument('--mode', choices=['full', 'incremental', 'delta'],
                           default='incremental', help="Sync mode")
        group.add_argument('--conflict-resolution', choices=['source', 'target', 'newest'],
                           default='source', help="Conflict resolution strategy")
        return group
    
    # -------------------------------------------------------------------
    # ARGUMENT PARSING AND EXECUTION
    # -------------------------------------------------------------------
    
    def parse_args(self, args: List[str] = None) -> argparse.Namespace:
        """Parse command-line arguments."""
        if args is None and len(sys.argv) == 1:
            self._print_usage_hint()
            sys.exit(1)
        
        self.args = self.parser.parse_args(args)
        self._post_process_args()
        self._configure_logging()
        self.start_time = datetime.now()
        
        return self.args
    
    def _print_usage_hint(self) -> None:
        """Print brief usage hint when no arguments provided."""
        print(f"{self.config.prog_name} v{self.config.version}")
        print(f"\nUsage: {self.config.prog_name} [options]")
        print(f"Try '{self.config.prog_name} --help' for more information.")
    
    def _post_process_args(self) -> None:
        """Post-process parsed arguments."""
        if getattr(self.args, 'no_color', False):
            Colors.disable()
        
        # Load password from file if specified
        for file_arg, target_arg in [('password_file', 'password'), 
                                      ('db_password_file', 'db_password'),
                                      ('api_key_file', 'api_key')]:
            file_path = getattr(self.args, file_arg, None)
            if file_path and os.path.isfile(file_path):
                with open(file_path, 'r', encoding='utf-8') as f:
                    setattr(self.args, target_arg, f.readline().strip())
    
    def _configure_logging(self) -> None:
        """Configure logging based on verbosity."""
        if getattr(self.args, 'quiet', False):
            level = logging.ERROR
        elif getattr(self.args, 'verbose', 0) >= 2:
            level = logging.DEBUG
        elif getattr(self.args, 'verbose', 0) >= 1:
            level = logging.INFO
        else:
            level = logging.WARNING
        
        logging.basicConfig(
            level=level,
            format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
        )
        
        log_file = getattr(self.args, 'log_file', None)
        if log_file:
            handler = logging.FileHandler(log_file, encoding='utf-8')
            handler.setFormatter(logging.Formatter('%(asctime)s - %(levelname)s - %(message)s'))
            logging.getLogger().addHandler(handler)
    
    # -------------------------------------------------------------------
    # STATISTICS AND OUTPUT
    # -------------------------------------------------------------------
    
    def increment_stat(self, key: str, amount: int = 1) -> None:
        """Increment a statistic counter."""
        self.stats[key] = self.stats.get(key, 0) + amount
    
    def print_final_summary(self) -> None:
        """Print final execution summary with elapsed time."""
        elapsed = datetime.now() - self.start_time if self.start_time else None
        
        if elapsed:
            self.stats['elapsed_time'] = str(elapsed).split('.')[0]
        
        print_summary(self.stats, title=f"{self.config.prog_name.upper()} RESULTS")
    
    def exit_with_error(self, message: str, code: int = 1) -> None:
        """Print error and exit with code."""
        print_error(message)
        sys.exit(code)
    
    def exit_success(self, message: str = None) -> None:
        """Print success message and exit."""
        if message:
            print_success(message)
        sys.exit(0)


# ============================================================================
# FACTORY FUNCTION
# ============================================================================

def create_cli(
    prog: str,
    description: str,
    version: str = "1.0.0",
    connection_type: str = None,
    operation_type: str = None
) -> CLIBase:
    """Factory function to create configured CLI instance.
    
    Args:
        prog: Program name
        description: Program description
        version: Version string
        connection_type: 'database', 'ldap', 'api', or None
        operation_type: 'export', 'import', 'sync', or None
    
    Returns:
        Configured CLIBase instance
    """
    cli = CLIBase(prog=prog, description=description, version=version)
    
    # Add connection arguments based on type
    if connection_type == 'database':
        cli.add_database_connection_group()
    elif connection_type == 'ldap':
        cli.add_ldap_connection_group()
    elif connection_type == 'api':
        cli.add_api_connection_group()
    
    # Add operation arguments based on type
    if operation_type == 'export':
        cli.add_export_group()
    elif operation_type == 'import':
        cli.add_import_group()
    elif operation_type == 'sync':
        cli.add_sync_group()
    
    return cli


# ============================================================================
# EXAMPLE USAGE
# ============================================================================

if __name__ == "__main__":
    # Example: Create CLI for a data export tool
    cli = create_cli(
        prog="{module_name}",
        description="{description}",
        version="{version}",
        connection_type="{connection_type}",
        operation_type="{operation_type}"
    )
    
    args = cli.parse_args()
    
    # Your tool logic here...
    cli.increment_stat('processed', 100)
    cli.increment_stat('success', 95)
    cli.increment_stat('errors', 5)
    
    cli.print_final_summary()
```

## Usage Guidelines

### Quick Start

Replace template variables and customize argument groups:

```python
# my_exporter.py
from cli import CLIBase, print_success, print_error

cli = CLIBase(
    prog="my_exporter",
    description="Export data to multiple formats",
    version="1.0.0"
)
cli.add_database_connection_group()
cli.add_export_group(formats=['csv', 'json', 'parquet'])

try:
    args = cli.parse_args()
    
    if args.dry_run:
        print_info("DRY RUN - No changes will be made")
    
    # Process data...
    cli.increment_stat('exported', 500)
    cli.print_final_summary()
    
except Exception as e:
    cli.exit_with_error(str(e))
```

### Command Examples

```bash
# Basic usage
python my_exporter.py --db-type postgresql --db-name mydb -f csv -o output.csv

# With verbosity and dry-run
python my_exporter.py -vv --dry-run --db-type sqlite --db-name test.db -f json -o out.json

# Load arguments from file
python my_exporter.py @production_params.txt

# With logging to file
python my_exporter.py --log-file export.log --db-type mysql --db-name prod -f excel -o report.xlsx
```

### Parameter File Format (@params.txt)

```
--db-type
postgresql
--db-host
localhost
--db-name
production
--format
csv
--output
/data/exports/output.csv
--verbose
```

## Customization Points

| Component | How to Customize |
|-----------|------------------|
| **Colors** | Override `Colors` class attributes or call `Colors.disable()` |
| **Output formats** | Pass custom list to `add_export_group(formats=[...])` |
| **Connection types** | Add new `add_*_connection_group()` methods |
| **Validation** | Override `_post_process_args()` method |
| **Help formatting** | Change `formatter_class` in parser construction |

## Integration Pattern

```python
# Tool with graceful fallback if CLI module not available
try:
    from cli import CLIBase, print_success, print_error
    CLI_AVAILABLE = True
except ImportError:
    CLI_AVAILABLE = False

def main():
    if CLI_AVAILABLE:
        return _main_with_cli()
    else:
        return _main_legacy()

def _main_with_cli():
    cli = CLIBase(prog="mytool", description="My Tool", version="1.0.0")
    # ... CLI implementation

def _main_legacy():
    # Fallback using basic argparse
    parser = argparse.ArgumentParser(description="My Tool")
    # ... basic implementation

if __name__ == "__main__":
    main()
```

## Output Format

Generate the following files for the project:

### 1. `cli.py` - Main CLI Module

Self-contained and project-specific module:

- Located in the project's main package directory (e.g., `myapp/gears/cli.py`)
- Importable as `from myapp.gears.cli import CLIBase, create_cli`

Complete Python module with:

1. Module docstring with usage examples
2. Future imports and type hints
3. `__all__` export list
4. Enums and dataclasses (OutputFormat, LogLevel, Colors, CLIConfig)
5. Output utilities (cprint, print_success, print_error, print_table, etc.)
6. Main CLIBase class with instance methods
7. Connection group methods (add_*_connection_group)
8. Factory function `create_cli()`

### 2. `cli_example.py` - Usage Example

Practical demonstration file showing all CLI capabilities:

```python
"""
Example usage of the CLI module.

Run: python cli_example.py --help
Run: python cli_example.py --host localhost --user admin --password secret -b DC=example,DC=com -f csv -o output.csv
Run: python cli_example.py @params.txt
"""
from {module_name}.gears.cli import (
    CLIBase, 
    create_cli,
    print_success, 
    print_error, 
    print_warning,
    print_info,
    print_header,
    print_table,
    print_summary,
    print_progress,
    confirm_action,
    Colors
)


def main():
    # ==========================================================================
    # Example 1: Basic CLI with factory function
    # ==========================================================================
    cli = create_cli(
        prog="{module_name}",
        description="{description}",
        version="{version}",
        connection_type="{connection_type}",  # 'ad', 'azure', 'db', 'ftp', 'email', 'sharepoint', 'api'
        operation_type="{operation_type}"     # 'export', 'import'
    )
    
    # Add usage examples for --help
    cli.add_examples([
        "%(prog)s --host dc.example.com -U admin -P pass -b DC=example,DC=com -f csv -o users.csv",
        "%(prog)s @production_params.txt",
        "%(prog)s --dry-run --verbose --host dc.local -U user -P pass -b DC=local -f json -o test.json"
    ])
    
    args = cli.parse_args()
    
    # ==========================================================================
    # Example 2: Manual CLI construction with custom groups
    # ==========================================================================
    # cli = CLIBase(
    #     prog="custom_tool",
    #     description="Custom tool with specific needs",
    #     version="2.0.0"
    # )
    # cli.add_ad_connection_group()      # Active Directory
    # cli.add_db_connection_group()      # Database
    # cli.add_export_group(formats=['csv', 'json', 'parquet'])
    # cli.add_examples([...])
    # args = cli.parse_args()
    
    # ==========================================================================
    # Example 3: Output utilities demo
    # ==========================================================================
    print_header("CLI DEMO - Output Utilities")
    
    # Colored messages
    print_success("Operation completed successfully")
    print_error("An error occurred")
    print_warning("This is a warning message")
    print_info("Informational message")
    
    # Progress bar
    print()
    print_info("Processing items...")
    for i in range(101):
        print_progress(i, 100, prefix="Progress", suffix="Complete")
    
    # Table output
    print()
    headers = ["Name", "Status", "Count"]
    rows = [
        ["Users", "Active", 150],
        ["Groups", "Synced", 25],
        ["Computers", "Pending", 78]
    ]
    print_table(headers, rows)
    
    # Summary statistics
    stats = {
        'total_processed': 253,
        'success': 240,
        'warnings': 8,
        'errors': 5
    }
    print_summary(stats, title="EXECUTION SUMMARY")
    
    # ==========================================================================
    # Example 4: Confirmation prompts
    # ==========================================================================
    # if not args.dry_run:
    #     if confirm_action("Are you sure you want to proceed?", default=False):
    #         print_info("Proceeding with operation...")
    #     else:
    #         print_warning("Operation cancelled by user")
    #         return
    
    # ==========================================================================
    # Example 5: Stat tracking
    # ==========================================================================
    cli.increment_stat('processed', 100)
    cli.increment_stat('success', 95)
    cli.increment_stat('errors', 5)
    
    cli.print_final_summary()
    print()
    print_info(f"Elapsed time: {cli.get_elapsed_time()}")


if __name__ == "__main__":
    main()
```

### 3. `README_cli.md` - Documentation

Comprehensive documentation for CLI module usage:

```markdown
# CLI Module

## Overview

Command-line interface module for {module_name} with colored output, progress indicators, 
and unified argument parsing.

## Installation

The module is included in the project. No additional dependencies required.
For Windows color support without native ANSI: `pip install colorama`

## Quick Start

\```python
from {module_name}.gears.cli import create_cli, print_success

# Create CLI with factory function
cli = create_cli(
    prog="my_tool",
    description="My awesome tool",
    connection_type="db",   # Adds database connection args
    operation_type="export" # Adds export config args
)

# Add usage examples
cli.add_examples([
    "%(prog)s --db-type postgresql --db-name mydb -f csv -o output.csv",
    "%(prog)s @params.txt"
])

# Parse arguments
args = cli.parse_args()

# Your logic here...
print_success("Done!")
\```

## Command Line Usage

### Basic Execution

\```bash
# Show help with all options and examples
python my_tool.py --help

# Show version
python my_tool.py --version

# Basic execution
python my_tool.py --host server.com --user admin --password secret

# With verbosity
python my_tool.py -v --host server.com ...   # INFO level
python my_tool.py -vv --host server.com ...  # DEBUG level

# Dry run mode
python my_tool.py --dry-run --host server.com ...

# Load parameters from file
python my_tool.py @production_params.txt
\```

### Parameter File Format (@params.txt)

\```
--host
dc.example.com
--user
admin@domain.com
--password
secretpassword
--base-dn
DC=example,DC=com
--format
csv
--output
/data/exports/users.csv
--verbose
\```

## Connection Types

### Active Directory (LDAP)

\```python
cli.add_ad_connection_group()
\```

| Argument | Required | Description |
|----------|----------|-------------|
| `--host`, `-H` | Yes | Domain Controller hostname |
| `--user`, `-U` | Yes | User principal (user@domain.com) |
| `--password`, `-P` | Yes | Password |
| `--password-file` | No | Read password from file |
| `--base-dn`, `-b` | Yes | Search base DN |
| `--no-ssl` | No | Disable SSL |
| `--auth-method` | No | SIMPLE, NTLM, KERBEROS |

### Azure AD / Entra ID

\```python
cli.add_azure_connection_group()
\```

| Argument | Required | Description |
|----------|----------|-------------|
| `--client-id` | Yes | Azure AD Application ID |
| `--client-secret` | Yes | Application secret |
| `--secret-file` | No | Read secret from file |
| `--tenant-id` | Yes | Azure AD Tenant ID |
| `--api-version` | No | v1.0 or beta |

### Database

\```python
cli.add_db_connection_group()
\```

| Argument | Required | Description |
|----------|----------|-------------|
| `--db-type` | Yes | postgresql, mysql, oracle, sqlserver, sqlite |
| `--db-host` | No* | Database host |
| `--db-port` | No | Database port |
| `--db-name` | Yes | Database name or SQLite file |
| `--db-user` | No* | Username |
| `--db-password` | No | Password |
| `--db-connection-string` | No | Full connection string |

### FTP/SFTP

\```python
cli.add_ftp_connection_group()
\```

| Argument | Required | Description |
|----------|----------|-------------|
| `--ftp-host` | Yes | Server hostname |
| `--ftp-port` | No | Port (21/22) |
| `--ftp-user` | Yes | Username |
| `--ftp-password` | No | Password |
| `--ftp-protocol` | No | ftp, ftps, sftp |
| `--ftp-key-file` | No | SSH private key |

### Email (SMTP/IMAP)

\```python
cli.add_email_connection_group()
\```

| Argument | Required | Description |
|----------|----------|-------------|
| `--email-protocol` | Yes | smtp, imap, pop3 |
| `--email-host` | Yes | Mail server |
| `--email-user` | Yes | Account username |
| `--email-password` | No | Account password |
| `--email-ssl` | No | Use SSL (default: True) |

### SharePoint Online

\```python
cli.add_sharepoint_connection_group()
\```

| Argument | Required | Description |
|----------|----------|-------------|
| `--sp-site-url` | Yes | SharePoint site URL |
| `--sp-client-id` | Yes | Azure AD App ID |
| `--sp-client-secret` | No | App secret |
| `--sp-tenant-id` | Yes | Tenant ID |
| `--sp-library` | No | Document library |

### REST API

\```python
cli.add_api_connection_group()
\```

| Argument | Required | Description |
|----------|----------|-------------|
| `--api-base-url` | Yes | Base API URL |
| `--api-auth-type` | No | none, basic, bearer, api-key, oauth2 |
| `--api-token` | No | Bearer token or API key |
| `--api-timeout` | No | Request timeout (seconds) |

## Output Utilities

### Colored Messages

\```python
from {module_name}.gears.cli import print_success, print_error, print_warning, print_info

print_success("Operation completed")  # ✓ Green
print_error("Something failed")       # ✗ Red
print_warning("Be careful")           # ⚠ Yellow
print_info("FYI message")             # ℹ Cyan
\```

### Tables

\```python
from {module_name}.gears.cli import print_table

headers = ["Name", "Status", "Count"]
rows = [
    ["Users", "Active", 150],
    ["Groups", "Synced", 25]
]
print_table(headers, rows)
\```

### Progress Bar

\```python
from {module_name}.gears.cli import print_progress

for i in range(101):
    print_progress(i, 100, prefix="Processing", suffix="Complete")
\```

### Summary Statistics

\```python
from {module_name}.gears.cli import print_summary

stats = {'processed': 100, 'success': 95, 'errors': 5}
print_summary(stats, title="RESULTS")
\```

## API Reference

### CLIBase Class

| Method | Description |
|--------|-------------|
| `CLIBase(prog, description, version)` | Create CLI instance |
| `add_example(example)` | Add usage example |
| `add_examples(list)` | Add multiple examples |
| `add_group(name, title)` | Add custom argument group |
| `add_ad_connection_group()` | Add AD connection args |
| `add_azure_connection_group()` | Add Azure AD args |
| `add_db_connection_group()` | Add database args |
| `add_ftp_connection_group()` | Add FTP/SFTP args |
| `add_email_connection_group()` | Add email args |
| `add_sharepoint_connection_group()` | Add SharePoint args |
| `add_api_connection_group()` | Add REST API args |
| `add_export_group(formats)` | Add export config args |
| `add_import_group(formats)` | Add import config args |
| `parse_args()` | Parse command line |
| `increment_stat(name, value)` | Track statistics |
| `print_final_summary()` | Print tracked stats |
| `get_elapsed_time()` | Get execution time |
| `exit_success(message)` | Exit with code 0 |
| `exit_error(message, code)` | Exit with error |

### Factory Function

\```python
cli = create_cli(
    prog="tool_name",
    description="Tool description",
    version="1.0.0",
    connection_type="db",  # 'ad', 'azure', 'db', 'ftp', 'email', 'sharepoint', 'api'
    operation_type="export" # 'export', 'import'
)
\```

## Global Options

Available in all CLI tools:

| Option | Description |
|--------|-------------|
| `--version`, `-V` | Show version |
| `--verbose`, `-v` | Increase verbosity |
| `--quiet`, `-q` | Suppress output |
| `--no-color` | Disable colors |
| `--dry-run` | Simulate only |
| `--output-format` | table, json, csv, summary, quiet |
| `--config-file` | Load JSON config |
| `--log-file` | Write logs to file |

## Customization

### Custom Connection Group

\```python
def add_custom_connection_group(cli: CLIBase):
    group = cli.add_group("custom", "Custom Connection")
    group.add_argument('--custom-host', required=True, help="Custom host")
    group.add_argument('--custom-token', help="API token")
    return group
\```

### Override Colors

\```python
from {module_name}.gears.cli import Colors

Colors.SUCCESS = Colors.BLUE  # Change success color to blue
Colors.disable()              # Disable all colors
\```
```
