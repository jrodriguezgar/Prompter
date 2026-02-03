# Lazy Library Import Management

Auto-detect missing libraries → install on-demand → eliminate manual dependency mgmt

**Supports both pip and uv package managers with automatic detection.**

## Core Functions

```python
import sys, subprocess, os

def _is_uv_managed_environment() -> bool:
    """Check if the current Python environment is managed by uv."""
    # Check if running via 'uv run' (UV_PROJECT_ENVIRONMENT is set)
    if os.environ.get('UV_PROJECT_ENVIRONMENT'):
        return True
    # Check if python is in uv's managed path
    if 'uv' in sys.executable.lower():
        return True
    # Check pyvenv.cfg in the venv directory (from executable path)
    # sys.executable is typically .venv/Scripts/python.exe on Windows
    exe_dir = os.path.dirname(sys.executable)
    venv_dir = os.path.dirname(exe_dir)  # Go up from Scripts to .venv
    pyvenv_cfg = os.path.join(venv_dir, 'pyvenv.cfg')
    if os.path.exists(pyvenv_cfg):
        try:
            with open(pyvenv_cfg, 'r') as f:
                content = f.read()
                # uv adds 'uv = x.x.x' line to pyvenv.cfg
                if 'uv =' in content or 'uv=' in content:
                    return True
        except:
            pass
    # Fallback: Check VIRTUAL_ENV environment variable
    venv = os.environ.get('VIRTUAL_ENV', '')
    if venv and os.path.exists(os.path.join(venv, 'pyvenv.cfg')):
        try:
            with open(os.path.join(venv, 'pyvenv.cfg'), 'r') as f:
                content = f.read()
                if 'uv =' in content or 'uv=' in content:
                    return True
        except:
            pass
    return False


def _find_pyproject_dir() -> str:
    """Find the directory containing pyproject.toml."""
    current = os.getcwd()
    while current != os.path.dirname(current):
        if os.path.exists(os.path.join(current, 'pyproject.toml')):
            return current
        current = os.path.dirname(current)
    return None


def install_library(library_name: str, import_name: str = None, package_name: str = None) -> bool:
    """Auto-install missing libraries via pip or uv (auto-detected).
    
    Args:
        library_name: The name to use for import check (e.g., 'google.genai')
        import_name: Deprecated, use library_name for import check instead
        package_name: The pip/uv package name to install (e.g., 'google-genai').
                      If not specified, uses library_name for installation.
    
    Returns: True if available/installed, False if failed
    
    Uses importlib.invalidate_caches() after installation to ensure
    newly installed packages are discoverable in the current session.
    
    Examples:
        # Same name for import and pip
        install_library("pandas")
        
        # Different import name (legacy pattern)
        install_library("scikit-learn", "sklearn")
        
        # Different pip package name (new pattern - preferred)
        install_library("google.genai", package_name="google-genai")
    """
    import importlib
    check_name = import_name if import_name else library_name
    pip_name = package_name if package_name else library_name
    
    # Try to import the module first
    try:
        importlib.import_module(check_name)
        return True
    except ImportError:
        pass
    
    # Module not found, try to install it
    print(f"Library '{check_name}' not found. Installing '{pip_name}'...")
    try:
        if _is_uv_managed_environment():
            project_dir = _find_pyproject_dir()
            if project_dir:
                # Use 'uv add' - updates pyproject.toml and installs
                subprocess.check_call(["uv", "add", pip_name], cwd=project_dir)
            else:
                subprocess.check_call(["uv", "pip", "install", "--system", pip_name])
        else:
            subprocess.check_call([sys.executable, "-m", "pip", "install", pip_name])
        
        # CRITICAL: Refresh import system to find newly installed packages
        importlib.invalidate_caches()
        
        # Remove from sys.modules if it was cached as failed/None
        # This is necessary because Python may have cached a failed import
        if check_name in sys.modules:
            del sys.modules[check_name]
        # Also remove any submodules that might have been partially loaded
        to_remove = [key for key in sys.modules if key.startswith(check_name + '.')]
        for key in to_remove:
            del sys.modules[key]
        
        # Verify the import works after installation
        try:
            importlib.import_module(check_name)
            print(f"✓ '{pip_name}' installed and '{check_name}' loaded")
            return True
        except ImportError as e:
            print(f"✗ '{pip_name}' installed but import '{check_name}' failed: {e}")
            return False
            
    except subprocess.CalledProcessError:
        print(f"✗ Failed to install '{pip_name}'")
        return False
    except FileNotFoundError:
        # uv not found, fallback to pip
        try:
            subprocess.check_call([sys.executable, "-m", "pip", "install", pip_name])
            importlib.invalidate_caches()
            if check_name in sys.modules:
                del sys.modules[check_name]
            try:
                importlib.import_module(check_name)
                print(f"✓ '{pip_name}' installed and '{check_name}' loaded")
                return True
            except ImportError:
                return False
        except subprocess.CalledProcessError:
            print(f"✗ Failed. For uv envs: uv add {pip_name}")
            return False
```

## Package Manager Detection

| Detection Method | Condition |
|-----------------|-----------|
| `UV_PROJECT_ENVIRONMENT` env var | Running via `uv run` |
| `sys.executable` contains "uv" | uv-managed Python directly |
| `pyvenv.cfg` contains "uv =" | venv created by uv (checked from exe path) |
| `VIRTUAL_ENV/pyvenv.cfg` | Fallback check via env var |

**Detection priority:**
1. Check `UV_PROJECT_ENVIRONMENT` environment variable
2. Check if "uv" is in `sys.executable` path
3. Check `pyvenv.cfg` next to Python executable (`.venv/Scripts/../pyvenv.cfg`)
4. Fallback: Check `VIRTUAL_ENV` environment variable

**Behavior by environment:**
- **pip environment**: Uses `pip install`
- **uv environment with pyproject.toml**: Uses `uv add` (updates dependencies)
- **uv environment without pyproject.toml**: Uses `uv pip install --system`

## Usage Patterns

```python
# Same name: PyPI = import
install_library("pandas"); import pandas as pd

# Different names (legacy): import_name parameter
install_library("scikit-learn", "sklearn"); import sklearn
install_library("Pillow", "PIL"); from PIL import Image

# Different names (preferred): package_name parameter
# Use when import path differs from pip package name
install_library("google.genai", package_name="google-genai")
from google import genai

install_library("PIL", package_name="Pillow")
from PIL import Image

# Conditional with fallback
if install_library("cv2", package_name="opencv-python"):
    import cv2
else:
    print("Using fallback")

# Batch install (updated for package_name)
deps = [
    ("numpy", None, None),
    ("pandas", None, None), 
    ("sklearn", None, "scikit-learn"),
    ("google.genai", None, "google-genai"),
]
all_ok = all(install_library(lib, imp, pkg) for lib, imp, pkg in deps)
```

## Recommended Pattern: Try Import First

**Best practice for critical dependencies**: Try importing first, only install if import fails.
This avoids false negatives from `install_library()` detection issues.

```python
# RECOMMENDED: Try import first, install only on failure with multi-strategy fallback
try:
    from ldap3 import Server, Connection, SUBTREE
    from ldap3.core.exceptions import LDAPException, LDAPBindError
except ImportError:
    print(f"ldap3 not found. Installing for: {sys.executable}")
    installed = False
    
    # Strategy 1: uv add (for uv-managed projects with pyproject.toml)
    if _is_uv_managed_environment():
        try:
            project_dir = _find_pyproject_dir()
            if project_dir:
                subprocess.check_call(["uv", "add", "ldap3"], cwd=project_dir)
            else:
                subprocess.check_call(["uv", "pip", "install", "ldap3", "--python", sys.executable])
            installed = True
        except (subprocess.CalledProcessError, FileNotFoundError):
            pass
    
    # Strategy 2: Standard pip
    if not installed:
        try:
            subprocess.check_call([sys.executable, "-m", "pip", "install", "ldap3"])
            installed = True
        except subprocess.CalledProcessError:
            # Strategy 3: pip with --break-system-packages (for externally-managed envs)
            try:
                subprocess.check_call([sys.executable, "-m", "pip", "install", "ldap3", "--break-system-packages"])
                installed = True
            except subprocess.CalledProcessError:
                pass
    
    if installed:
        from ldap3 import Server, Connection, SUBTREE
        from ldap3.core.exceptions import LDAPException, LDAPBindError
        print("✓ ldap3 installed successfully")
    else:
        raise RuntimeError(
            f"ldap3 is required but installation failed.\n"
            f"Python interpreter: {sys.executable}\n"
            f"Please install manually using one of:\n"
            f"  - uv add ldap3\n"
            f"  - uv pip install ldap3 --python {sys.executable}\n"
            f"  - {sys.executable} -m pip install ldap3 --break-system-packages"
        )
```

**Installation Strategy Priority:**

| Priority | Strategy | When to Use |
|----------|----------|-------------|
| 1 | `uv add <pkg>` | uv-managed project with `pyproject.toml` |
| 2 | `uv pip install <pkg> --python <exe>` | uv-managed Python without project file |
| 3 | `pip install <pkg>` | Standard pip environments |
| 4 | `pip install <pkg> --break-system-packages` | Externally-managed environments (PEP 668) |

**Why this pattern is superior:**
- Avoids false negatives from `install_library()` detection issues
- Works correctly when package is already installed in a different Python path
- Faster execution (no subprocess calls when package exists)
- More reliable across different environment configurations
- **Handles PEP 668 externally-managed environments** (uv, system Python)
- **Provides clear error messages** with multiple installation options

**When to use:**
- Critical dependencies that must be present
- Packages with multiple submodule imports
- Connectors and adapters to external systems
- **Projects that may run in uv-managed environments**

## Common Mappings

| PyPI (package_name)  | Import (library_name) | Example Call |
| -------------------- | --------------------- | ------------ |
| `google-genai`       | `google.genai`        | `install_library("google.genai", package_name="google-genai")` |
| `scikit-learn`       | `sklearn`             | `install_library("sklearn", package_name="scikit-learn")` |
| `Pillow`             | `PIL`                 | `install_library("PIL", package_name="Pillow")` |
| `opencv-python`      | `cv2`                 | `install_library("cv2", package_name="opencv-python")` |
| `python-dateutil`    | `dateutil`            | `install_library("dateutil", package_name="python-dateutil")` |
| `beautifulsoup4`     | `bs4`                 | `install_library("bs4", package_name="beautifulsoup4")` |
| `PyYAML`             | `yaml`                | `install_library("yaml", package_name="PyYAML")` |

## Guidelines

**✓ Do**: 
- Call before import 
- Handle return for critical deps 
- Use `import_name` for mismatches 
- Use `uv run` for uv-managed projects

**✗ Don't**: 
- Call in loops 
- Ignore return 
- Use for stdlib (sys/os/json) 
- Run uv Python directly (use `uv run`)

## Error Handling Patterns

```python
# Critical: raise if failed
if not install_library("numpy"): 
    raise RuntimeError("NumPy required")
import numpy as np

# Optional: feature flag
HAS_MPL = install_library("matplotlib")
if HAS_MPL: 
    import matplotlib.pyplot as plt

# Versioned (pip only)
def install_ver(lib: str, ver: str = None, imp: str = None) -> bool:
    pkg = f"{lib}=={ver}" if ver else lib
    try:
        __import__(imp or lib)
        return True
    except ImportError:
        return subprocess.run([sys.executable, "-m", "pip", "install", pkg]).returncode == 0
```

## Running in uv-managed Projects

```powershell
# ✓ Correct - uses virtual environment with dependencies
uv run python script.py

# ✗ Wrong - externally-managed error
C:\...\uv\python\...\python.exe script.py
```

## Best Practices

| Aspect | Recommendation |
|--------|---------------|
| Placement | Module level, before imports |
| Grouping | Batch install related deps |
| Logging | Keep print feedback |
| Return checks | Validate critical deps |
| uv projects | Always use `uv run` |
| Import method | Use `importlib.import_module()` not `__import__()` |
| Cache refresh | Always call `importlib.invalidate_caches()` after install |
| sys.modules | Clear module and submodules from `sys.modules` after install |
| Critical deps | Use "try import first" pattern (see above) |

### Critical: importlib.invalidate_caches()

When installing packages at runtime, Python's import system may have cached that a module doesn't exist. After installing, you **must** call `importlib.invalidate_caches()` to refresh the finder caches, otherwise the import will fail even though the package is installed.

### Critical: Clear sys.modules cache

If a previous import attempt failed, Python may have cached a `None` or partial entry in `sys.modules`. You must remove these before re-importing:

```python
import importlib
import subprocess
import sys

# Install the package
subprocess.check_call([sys.executable, "-m", "pip", "install", "some-package"])

# CRITICAL: Refresh finder caches
importlib.invalidate_caches()

# CRITICAL: Remove failed import cache from sys.modules
check_name = "some_package"
if check_name in sys.modules:
    del sys.modules[check_name]
# Also remove any submodules
to_remove = [key for key in sys.modules if key.startswith(check_name + '.')]
for key in to_remove:
    del sys.modules[key]

# Now the import will work
import some_package
```

## Use Cases

**✓ Use**: Prototypes · Educational code · Notebooks · Quick CLI · Unknown env  
**✗ Avoid**: Production · Containers · CI/CD · Security-sensitive · Team projects

## Limitations

| Limitation | Description |
|------------|-------------|
| Security | No package verification |
| Version | No pinning (use pyproject.toml) |
| Offline | Requires internet |
| Permissions | May fail in restricted envs |
| Conflicts | Unhandled dependency conflicts |
| PEP 668 | Externally-managed envs require `--break-system-packages` or uv |

## Handling PEP 668 Externally-Managed Environments

Python 3.11+ and uv-managed installations mark themselves as "externally managed" per PEP 668.
Direct pip install fails with `error: externally-managed-environment`.

**Solutions:**

```python
# Option 1: Use uv for installation
subprocess.check_call(["uv", "pip", "install", "package", "--python", sys.executable])

# Option 2: Use --break-system-packages (last resort)
subprocess.check_call([sys.executable, "-m", "pip", "install", "package", "--break-system-packages"])

# Option 3: Create a virtual environment (recommended for projects)
# uv venv .venv --python 3.11
# source .venv/bin/activate  # or .venv\Scripts\activate on Windows
```

## Alternatives

```toml
# pyproject.toml (recommended for uv)
[project]
dependencies = [
    "numpy>=1.24.0",
    "pandas>=2.0.0",
]
```

```txt
# requirements.txt (for pip)
numpy>=1.24.0
pandas>=2.0.0
```
