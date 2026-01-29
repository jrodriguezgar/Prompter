# Lazy Import Integration

Integrate lazy import support into modules that require installable external libraries.

## Code to Integrate

Add these functions at the module level (after standard library imports):

```python
import sys
import subprocess
import os
import importlib

def _is_uv_managed_environment() -> bool:
    """Check if the current Python environment is managed by uv."""
    if os.environ.get('UV_PROJECT_ENVIRONMENT'):
        return True
    if 'uv' in sys.executable.lower():
        return True
    exe_dir = os.path.dirname(sys.executable)
    venv_dir = os.path.dirname(exe_dir)
    pyvenv_cfg = os.path.join(venv_dir, 'pyvenv.cfg')
    if os.path.exists(pyvenv_cfg):
        try:
            with open(pyvenv_cfg, 'r') as f:
                content = f.read()
                if 'uv =' in content or 'uv=' in content:
                    return True
        except:
            pass
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
    Returns: True if available/installed, False if failed"""
    check_name = import_name if import_name else library_name
    try:
        importlib.import_module(check_name)
        return True
    except ImportError:
        pass
    print(f"Installing '{library_name}'...")
    try:
        if _is_uv_managed_environment():
            project_dir = _find_pyproject_dir()
            if project_dir:
                subprocess.check_call(["uv", "add", library_name], cwd=project_dir)
            else:
                subprocess.check_call(["uv", "pip", "install", "--system", library_name])
        else:
            subprocess.check_call([sys.executable, "-m", "pip", "install", library_name])
        importlib.invalidate_caches()
        if check_name in sys.modules:
            del sys.modules[check_name]
        to_remove = [key for key in sys.modules if key.startswith(check_name + '.')]
        for key in to_remove:
            del sys.modules[key]
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

## Usage

Call `install_library()` before importing external libraries:

```python
install_library("tkcalendar")
from tkcalendar import DateEntry

install_library("Pillow", "PIL")
from PIL import Image
```

## Common Mappings Examples (PyPI → import)

| PyPI             | Import         |
| ---------------- | -------------- |
| `tkcalendar`   | `tkcalendar` |
| `tkinterweb`   | `tkinterweb` |
| `Pillow`       | `PIL`        |
| `scikit-learn` | `sklearn`    |
