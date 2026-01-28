# Lazy Library Import Management

Auto-detect missing libraries → install on-demand → eliminate manual dependency mgmt

## Core Function

```python
import sys, subprocess

def install_library(library_name: str, import_name: str = None) -> bool:
    """Auto-install missing libraries via pip
    Args: library_name (PyPI), import_name (module, optional)
    Returns: True if available/installed, False if failed"""
    try:
        __import__(import_name or library_name)
        return True
    except ImportError:
        print(f"Installing '{library_name}'...")
        try:
            subprocess.check_call([sys.executable, "-m", "pip", "install", library_name])
            print(f"✓ '{library_name}' installed")
            return True
        except subprocess.CalledProcessError:
            print(f"✗ Failed to install '{library_name}'")
            return False
```

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
deps = [("numpy", None), ("scikit-learn", "sklearn"), ("Pillow", "PIL")]
all_ok = all(install_library(pkg, imp) for pkg, imp in deps)
```

## Common Mappings

| PyPI               | Import      | PyPI                | Import       |
| ------------------ | ----------- | ------------------- | ------------ |
| `scikit-learn`   | `sklearn` | `Pillow`          | `PIL`      |
| `opencv-python`  | `cv2`     | `python-dateutil` | `dateutil` |
| `beautifulsoup4` | `bs4`     | `PyYAML`          | `yaml`     |

## Guidelines

**✓ Do**: Call before import · Handle return for critical deps · Use `import_name` for mismatches · Keep print feedback
**✗ Don't**: Call in loops · Ignore return · Use for stdlib (sys/os/json) · Use in production

## Error Handling Patterns

### Critical Dependency

```python
if not install_library("numpy"):
    raise RuntimeError("NumPy required but installation failed")
import numpy as np
```

### Optional Dependency

```python
HAS_MATPLOTLIB = install_library("matplotlib")
if HAS_MATPLOTLIB:
    import matplotlib.pyplot as plt
  
def plot_data(data):
    if not HAS_MA
```python
# Critical: raise if failed
if not install_library("numpy"): raise RuntimeError("NumPy required")
import numpy as np

# Optional: feature flag
HAS_MPL = install_library("matplotlib")
if HAS_MPL: import matplotlib.pyplot as plt

# Versioned
def install_ver(lib: str, ver: str = None, imp: str = None) -> bool:
    pkg = f"{lib}=={ver}" if ver else lib
    try:
        __import__(imp or lib)
        return True
    except ImportError:
        return subprocess.run([sys.executable, "-m", "pip", "install", pkg]).returncode == 0

install_ver("pandas", "2.0.0")
```

## Best Practices

Placement → module level · Grouping → batch install · Logging → keep prints · Return checks → validate critical · Testing → both. Attempting to install...")
        try:
            subprocess.check_call([sys.executable, "-m", "pip", "install", library_name])
            print(f"Library '{library_name}' installed successfully.")
            return True
        except subprocess.CalledProcessError:
            print(f"Error: Failed to install '{library_name}'.")
            return False

# Dependency initialization

print("Checking dependencies...")
deps_ok = all([
    install_library("numpy"),
    install_library("pandas"),
    install_library("scikit-learn", "sklearn")

```python
import sys, subprocess

def install_library(library_name: str, import_name: str = None) -> bool:
    try:
        __import__(import_name or library_name)
        return True
    except ImportError:
        print(f"Installing '{library_name}'...")
        try:
            subprocess.check_call([sys.executable, "-m", "pip", "install", library_name])
            return True
        except subprocess.CalledProcessError:
            return False

# Init deps
deps = [("numpy", None), ("pandas", None), ("scikit-learn", "sklearn")]
if not all(install_library(p, i) for p, i in deps):
    raise RuntimeError("Deps install failed")

import numpy as np, pandas as pd, sklearn
HAS_MPL = install_library("matplotlib")
if HAS_MPL: import matplotlib.pyplot as plt

def main():
    data = np.array([1, 2, 3])
    df = pd.DataFrame(data)
    if HAS_MPL: plt.plot(data)
    else: print("Plot unavailable")
Use Cases
**✓ Use**: Prototypes · Educational code · Notebooks · Quick CLI · Unknown env  
**✗ Avoid**: Production · Containers · CI/CD · Security-sensitive · Team projects

## Limitations
Security → no verification · Version → no pinning · Offline → needs internet · Permissions → restricted envs · Conflicts → unhandled · Performance → startup delay

## Alternatives
```txt
# requirements.txt
numpy>=1.24.0
pandas>=2.0.0
scikit-learn>=1.3.0
```
