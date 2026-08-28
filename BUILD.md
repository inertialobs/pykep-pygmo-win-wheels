# Building from Source

This guide explains how to build Windows wheels for PyKEP and PyGMO from source, reproducing the exact binaries provided in this repository.

---

##  Prerequisites

### Required Software

| Tool | Version | Notes |
|------|---------|-------|
| **Visual Studio Build Tools** | 2026 (MSVC v19.51) | Must include "Desktop development with C++" workload; **MSVC build tools for x64/x86** must be  checked|
| **CMake** | ≥ 4.2.0 | Recommended: [winlibs](https://winlibs.com/) bundle, or `winget install Kitware.CMake` |
| **Miniconda** | Latest | For managing C++ dependencies (`heyoka`, `Boost`, `fmt`, `SPICE`, etc.) |
| **Python** | 3.14 (64-bit) | Target Python version for the wheels |
| **Git** | Latest | For cloning repositories |

### Verify Your Environment

Open **"x64 Native Tools Command Prompt for VS 2026"** and run:

```cmd
cl          # Should show Microsoft C/C++ compiler version
cmake --version   # Should be ≥ 4.2.0
conda --version   # Should be installed
python --version  # Should be 3.14.x (64-bit)
```

#### If `cl` is not recognized

The MSVC compiler component may **not installed by default** with Build Tools. To fix:

1. Open Visual Studio Installer → Modify → **"Visual Studio Build Tools 2026"**
2. Switch to **"Individual components"** tab
3. Search for **"MSVC"** and check:
   - **"MSVC build tools for x64/x86(Latest)"**
4. Click **Modify** and wait for installation

If `cmake` is too old, download a newer version (e.g., from [winlibs](https://winlibs.com/)) and add it to your `PATH` temporarily:

```cmd
set PATH=D:\path\to\winlibs\bin;%PATH%
```

---

##  Step 1: Create a Conda Environment with All Dependencies

The official PyKEP repository provides a `kep3_devel.yml` file that lists all required C++ libraries:

```cmd
git clone https://github.com/esa/pykep.git
cd pykep
conda env create -f kep3_devel.yml
conda activate kep3_devel
```

If the environment creation is slow, consider using `mamba`:

```cmd
conda install mamba -n base -c conda-forge
mamba env create -f kep3_devel.yml
```

Verify that SPICE is installed (needed for `udpla` support):

```cmd
conda list cspice
```

If not present, install it manually:

```cmd
conda install -c conda-forge cspice
```

---

##  Step 2: Build PyKEP

### 2.1 Configure CMake

```cmd
cd pykep
rmdir /s /q build
mkdir build
cd build

cmake .. -G "Visual Studio 18 2026" -A x64 -DCMAKE_BUILD_TYPE=Release -Dkep3_BUILD_PYTHON_BINDINGS=ON -DCMAKE_PREFIX_PATH="%CONDA_PREFIX%\Library"
```

> **Important**: `-DCMAKE_PREFIX_PATH` tells CMake to look for dependencies (Boost, fmt, heyoka, SPICE, etc.) inside the Conda environment's `Library` folder.

If SPICE libraries are not found automatically, manually specify them:

```cmd
cmake .. -G "Visual Studio 18 2026" -A x64 -DCMAKE_BUILD_TYPE=Release -Dkep3_BUILD_PYTHON_BINDINGS=ON -DCMAKE_PREFIX_PATH="%CONDA_PREFIX%\Library" -Dcspice_DIR="%CONDA_PREFIX%\Library\lib\cmake\cspice" -DGMP_INCLUDE_DIR="%CONDA_PREFIX%\Library\include" -DGMP_LIBRARY="%CONDA_PREFIX%\Library\lib\gmp.lib"
```

Check the output summary – it should show `python=ON` and `build_kep3=ON`.

### 2.2 Compile

```cmd
cmake --build . --config Release --parallel
```

This produces:
- `Release/kep3.dll` – the C++ core library
- `pykep/Release/core.cp314-win_amd64.pyd` – the Python extension module
- `pykep/_version.py` – the version file

> **Note**: You may see a warning `The vcpkg manifest was disabled...` – this is harmless and can be ignored.

### 2.3 Prepare the Python Package

The compiled `.pyd`, `_version.py`, and their dependencies need to be placed into the source `pykep/` folder.

Copy the `.pyd` file:

```cmd
copy pykep\Release\core.cp314-win_amd64.pyd ..\pykep\core.pyd
```

Copy the version file:

```cmd
copy pykep\_version.py ..\pykep\
```

Copy the required DLLs (from `%CONDA_PREFIX%\Library\bin`) into a `lib` subfolder:

```cmd
mkdir ..\pykep\lib
copy %CONDA_PREFIX%\Library\bin\kep3.dll ..\pykep\lib\
copy %CONDA_PREFIX%\Library\bin\heyoka.dll ..\pykep\lib\
copy %CONDA_PREFIX%\Library\bin\boost_serialization*.dll ..\pykep\lib\
copy %CONDA_PREFIX%\Library\bin\fmt*.dll ..\pykep\lib\
copy %CONDA_PREFIX%\Library\bin\spdlog*.dll ..\pykep\lib\
copy %CONDA_PREFIX%\Library\bin\tbb*.dll ..\pykep\lib\
# ... and any other DLLs reported by dumpbin /dependents on core.pyd
```

> **To obtain a minimal set of DLLs**: Use the "file locking" method – import `pykep` in a Python session and delete all DLLs in `lib` that are not locked (PermissionError). See the troubleshooting section below.

### 2.4 Pack the Data Files

The `data/` directory (containing `de440s.bsp`, `naif0012.tls`, `tle.txt`) must be included. Copy it from the project root:

```cmd
xcopy /E /I ..\data ..\pykep\data
```

Now the `pykep/` folder is ready to be packaged into a wheel.

---

##  Step 3: Build PyGMO

PyGMO depends on the `pagmo` C++ library, which is already available in the Conda environment. We can use it directly.

### 3.1 Clone the Repository

```cmd
cd ..
git clone https://github.com/esa/pygmo2.git
cd pygmo2
```

### 3.2 Configure CMake

```cmd
rmdir /s /q build
mkdir build
cd build

cmake .. -G "Visual Studio 18 2026" -A x64 -DCMAKE_BUILD_TYPE=Release -DCMAKE_PREFIX_PATH="%CONDA_PREFIX%\Library" -Dpygmo_BUILD_PYTHON_BINDINGS=ON
```

> **Important**: `-Dpygmo_BUILD_PYTHON_BINDINGS=ON` explicitly enables Python binding generation. (Some versions enable this by default, but it's safer to specify it.)

CMake will automatically find `pagmo`, `Boost`, `Eigen`, and other dependencies.

### 3.3 Compile

```cmd
cmake --build . --config Release --parallel
```

This produces:
- `pygmo/Release/core.cp314-win_amd64.pyd` – the Python extension module
- `pygmo/_version.py` – the version file

### 3.4 Prepare the Python Package

Copy the `.pyd` file:

```cmd
copy pygmo\Release\core.cp314-win_amd64.pyd ..\pygmo\core.pyd
```

Copy the version file:

```cmd
copy pygmo\_version.py ..\pygmo\
```

Copy required DLLs into a `lib` folder:

```cmd
mkdir ..\pygmo\lib
copy %CONDA_PREFIX%\Library\bin\pagmo.dll ..\pygmo\lib\
copy %CONDA_PREFIX%\Library\bin\boost_serialization*.dll ..\pygmo\lib\
copy %CONDA_PREFIX%\Library\bin\ipopt-3.dll ..\pygmo\lib\
copy %CONDA_PREFIX%\Library\bin\nlopt.dll ..\pygmo\lib\
copy %CONDA_PREFIX%\Library\bin\tbb*.dll ..\pygmo\lib\
copy %CONDA_PREFIX%\Library\bin\dmumps.dll ..\pygmo\lib\
copy %CONDA_PREFIX%\Library\bin\libblas.dll ..\pygmo\lib\
copy %CONDA_PREFIX%\Library\bin\liblapack.dll ..\pygmo\lib\
copy %CONDA_PREFIX%\Library\bin\mkl_*.dll ..\pygmo\lib\
```

(Again, use `dumpbin` or the locking method to minimize the set.)

---

##  Step 4: Create the Wheels

Now that the `pykep/` and `pygmo/` directories are complete with all binaries and Python files, we can package them.

### 4.1 Create `setup.py` for PyKEP

Place a `setup.py` in **the project root directory** (the directory that contains the `pykep/` folder). For example, if your project root is `D:\code\pykep\` and it contains `pykep/` and `data/` as subdirectories, then `setup.py` goes into `D:\code\pykep\`.

Run `python setup.py bdist_wheel` from this same directory.

A minimal `setup.py` is:

```python
from setuptools import setup, find_packages

setup(
    name="pykep",
    version="3.0.1",
    packages=find_packages(include=["pykep", "pykep.*"]),
    package_data={
        "pykep": [
            "core.pyd",
            "lib/*",
            "_vendor/*",
            "_vendor/heyoka/*",
            "trajopt/gym/tops/*.json",
            "data/*",
        ],
    },
    install_requires=[
        "numpy",
        "scipy",
        "matplotlib",
        "sgp4",
        "spiceypy",
        "pygmo",           # Note: pygmo must be installed manually first (no Windows wheel on PyPI)
        "cloudpickle",
        "certifi",
    ],
    python_requires="==3.14.*",  # ABI-specific – only for Python 3.14
    has_ext_modules=lambda: True,
)
```

> **Important**: `pygmo` is listed as a dependency, but the official PyPI does **not** provide a Windows wheel. Users must install the provided `pygmo` wheel **before** installing `pykep`.

### 4.2 Build the Wheel

```cmd
python setup.py bdist_wheel
```

The wheel will be created in the `dist/` subdirectory.

### 4.3 Repeat for PyGMO

Similarly, create a `setup.py` for PyGMO in its own project root directory:

```python
from setuptools import setup, find_packages

setup(
    name="pygmo",
    version="2.19.8",
    packages=find_packages(include=["pygmo", "pygmo.*"]),
    package_data={
        "pygmo": ["core.pyd", "lib/*"],
    },
    install_requires=[
        "numpy",
        "scipy",
        "cloudpickle",
    ],
    extras_require={
        "plot": ["matplotlib>=3.0"],
        "graph": ["networkx>=2.5"],
        "all": ["matplotlib>=3.0", "networkx>=2.5"],
    },
    python_requires="==3.14.*",
    has_ext_modules=lambda: True,
)
```

Build:

```cmd
python setup.py bdist_wheel
```

---

##  Step 5: Test the Wheels

Create a fresh virtual environment and install the wheels in the correct order:

```cmd
python -m venv testenv
testenv\Scripts\activate
pip install pygmo-2.19.8-cp314-cp314-win_amd64.whl
pip install pykep-3.0.1-cp314-cp314-win_amd64.whl
```

Run the test suites:

```python
import pykep
pykep.test.run_test_suite()   # Should pass 89/89 tests

import pygmo
pygmo.test.run_test_suite()   # Should pass all core tests
```

> **Note**: The `free_form` test will error if `networkx` is not installed; this is expected because it's optional. Install `networkx` to make it pass:
> ```bash
> pip install networkx
> ```

---

##  Troubleshooting

### CMake cannot find SPICE or GMP

Manually set the paths as shown in the configuration steps above. Ensure the `cspice` and `gmp` packages are installed in Conda:

```cmd
conda install -c conda-forge cspice gmp
```

### Missing DLLs at runtime

Use `dumpbin /dependents core.pyd` to list required DLLs. Copy any missing ones from `%CONDA_PREFIX%\Library\bin` into the `lib` folder.

### The `pykep` wheel fails to import due to missing `pygmo`

PyKEP depends on PyGMO. On Windows, you must install the provided `pygmo` wheel first:

```bash
pip install pygmo-2.19.8-cp314-cp314-win_amd64.whl
pip install pykep-3.0.1-cp314-cp314-win_amd64.whl
```

### `ModuleNotFoundError: No module named 'pykep._version'`

The `_version.py` file was not copied during preparation. Copy it from `build/pykep/` or `build/pygmo/`:

```cmd
copy build\pykep\_version.py pykep\
```

### The wheel is too large

The `de440s.bsp` file is about 32 MB. If you don't need SPICE support, you can omit it, but then `udpla.de440s` will not work.

### `vcpkg manifest was disabled` warning

This is harmless – it does not affect the build. You can safely ignore it.

---

##  References

- [PyKEP GitHub](https://github.com/esa/pykep)
- [PyGMO GitHub](https://github.com/esa/pygmo2)
- [Conda Forge](https://conda-forge.org/)
- [winlibs](https://winlibs.com/)
- [Visual Studio Build Tools](https://visualstudio.microsoft.com/downloads/#build-tools-for-visual-studio-2022)

---

Happy building! 🚀