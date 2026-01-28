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


def install_library(library_name: str, import_name: str = None) -> bool:
    """Auto-install missing libraries via pip or uv (auto-detected).
    Args: library_name (PyPI), import_name (module, optional)
    Returns: True if available/installed, False if failed
    
    Uses importlib.invalidate_caches() after installation to ensure
    newly installed packages are discoverable in the current session."""
    import importlib
    check_name = import_name if import_name else library_name
    
    # Try to import the module first
    try:
        importlib.import_module(check_name)
        return True
    except ImportError:
        pass
    
    # Module not found, try to install it
    print(f"Installing '{library_name}'...")
    try:
        if _is_uv_managed_environment():
            project_dir = _find_pyproject_dir()
            if project_dir:
                # Use 'uv add' - updates pyproject.toml and installs
                subprocess.check_call(["uv", "add", library_name], cwd=project_dir)
            else:
                subprocess.check_call(["uv", "pip", "install", "--system", library_name])
        else:
            subprocess.check_call([sys.executable, "-m", "pip", "install", library_name])
        
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
            print(f"✓ '{library_name}' installed and loaded")
            return True
        except ImportError as e:
            print(f"✗ '{library_name}' installed but import failed: {e}")
            return False
            
    except subprocess.CalledProcessError:
        print(f"✗ Failed to install '{library_name}'")
        return False
    except FileNotFoundError:
        # uv not found, fallback to pip
        try:
            subprocess.check_call([sys.executable, "-m", "pip", "install", library_name])
            importlib.invalidate_caches()
            if check_name in sys.modules:
                del sys.modules[check_name]
            try:
                importlib.import_module(check_name)
                print(f"✓ '{library_name}' installed and loaded")
                return True
            except ImportError:
                return False
        except subprocess.CalledProcessError:
            print(f"✗ Failed. For uv envs: uv add {library_name}")
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

# Different names: PyPI ≠ import  
install_library("scikit-learn", "sklearn"); import sklearn
install_library("Pillow", "PIL"); from PIL import Image

# Conditional with fallback
if install_library("opencv-python", "cv2"):
    import cv2
else:
    print("Using fallback")

# Batch install
deps = [("numpy", None), ("pandas", None), ("scikit-learn", "sklearn")]
all_ok = all(install_library(pkg, imp) for pkg, imp in deps)
```

## Common Mappings

| PyPI               | Import      | PyPI                | Import       |
| ------------------ | ----------- | ------------------- | ------------ |
| `scikit-learn`   | `sklearn` | `Pillow`          | `PIL`      |
| `opencv-python`  | `cv2`     | `python-dateutil` | `dateutil` |
| `beautifulsoup4` | `bs4`     | `PyYAML`          | `yaml`     |
| `tkcalendar`     | `tkcalendar` | `tkinterweb`   | `tkinterweb` |

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
