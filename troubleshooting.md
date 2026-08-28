# Troubleshooting

This document covers common issues when **installing and running** the pre-built wheels, as well as **building from source**.


##  Quick Navigation

- **Installing the pre-built wheels?** → Start with [Installation & Import Issues](#installation--import-issues)
- **Building from source?** → Start with [Build Issues](#build-issues)


##  Installation & Import Issues

> For users downloading and installing the pre-built wheels from Releases.


### `ERROR: pygmo ... does not appear to be a Python project`

**Symptom**: When you run `pip install pykep-3.0.1-cp314-cp314-win_amd64.whl` directly (without first installing the provided `pygmo` wheel), you see:
```
Collecting pygmo (from pykep==3.0.1)
  Downloading pygmo-v2.19.0.tar.gz (3.0 MB)
ERROR: pygmo from ... does not appear to be a Python project: neither 'setup.py' nor 'pyproject.toml' found.
```

**Why this happens**:  
This is not a bug in these wheels – it's an upstream issue.
- `pykep` declares `pygmo` as a dependency in its `install_requires`.  
- On Windows, there is **no official `pygmo` wheel** on PyPI – only a source tarball (`tar.gz`) which is **not pip-installable** (it lacks a `setup.py`).  
- `pip` sees that `pygmo` is required, finds the source tarball, attempts to install it, and fails.

**Solution**:  
**Always install the provided `pygmo` wheel before installing `pykep`.** This tells `pip` that the dependency is already satisfied, so it won't try to download the broken source package.

```bash
# 1. Install pygmo first
pip install pygmo-2.19.8-cp314-cp314-win_amd64.whl

# 2. Then install pykep
pip install pykep-3.0.1-cp314-cp314-win_amd64.whl
```

### `ModuleNotFoundError: No module named 'pygmo'`

**Symptom**: `import pykep` fails with `ModuleNotFoundError: No module named 'pygmo'`.

**Solution**: Install the `pygmo` wheel first (see above).

### `ImportError: DLL load failed`

**Symptom**:
```
ImportError: DLL load failed while importing core: 找不到指定的模块。
```

**Possible causes and solutions**:

1. **Missing Visual C++ Redistributable**  
   Download and install from [Microsoft](https://aka.ms/vs/17/release/vc_redist.x64.exe).

2. **Missing DLLs in the `lib/` folder**  
   Check if `pykep/lib/` or `pygmo/lib/` contains `.dll` files. If not, the wheel was corrupted during download – re-download it.

3. **`os.add_dll_directory` not called**  
   Ensure the `__init__.py` file inside the package contains the DLL loading code. (Included in our wheels by default.)

### `FileNotFoundError: tops/_tops_cr3bp.json`

**Symptom**:
```
FileNotFoundError: .../pykep/trajopt/gym/tops/_tops_cr3bp.json
```

**Cause**: JSON data files missing from the wheel (should be fixed in our wheels).

**Solution**: Re-download the wheel from Releases. If you built it yourself, ensure your `setup.py` includes:
```python
package_data={"pykep": ["trajopt/gym/tops/*.json"]}
```

### `ModuleNotFoundError: No module named 'spiceypy'` / `sgp4` / `networkx`

**Symptom**: Importing `pygmo` or using specific features fails with a missing module.

**Solution**:
```bash
# Install the optional dependencies (not auto-installed for pygmo)
pip install networkx   # for free_form topology
pip install matplotlib # for plotting utilities
```


### Performance limitations on AMD CPUs

**Symptom**: PyKEP/PyGMO runs noticeably slower on AMD CPUs compared to Intel CPUs, but no error is reported.

**Cause**: Intel MKL intentionally uses slower code paths on non-Intel CPUs (e.g., SSE instead of AVX2). This is a known upstream behavior of the MKL library, not a bug in these wheels.

**Status**:
- The commonly suggested workaround (`MKL_DEBUG_CPU_TYPE=5`) **no longer works** on recent MKL versions (≥ 2020 Update 1).
- If you require optimal performance on AMD hardware, consider:
  - Using AMD's [AOCL](https://www.amd.com/en/developer/aocl.html) instead of Intel MKL (requires rebuilding from source).
  - Using open-source alternatives like [BLIS](https://github.com/flame/blis).

**Note**: The wheels in this repository are built with Intel MKL. The performance difference is a limitation of the upstream library, not something we can fix at the packaging level.


---

##  Build Issues

> For users building from source, following the steps in [`BUILD.md`](BUILD.md).

### CMake: `Could NOT find mp++_GMP`

**Symptom**:
```
Could NOT find mp++_GMP (missing: MPPP_GMP_INCLUDE_DIR MPPP_GMP_LIBRARY)
```

**Solution**:
```bash
conda install -c conda-forge gmp mp++
```
Or manually specify:
```
-DGMP_INCLUDE_DIR="%CONDA_PREFIX%\Library\include" -DGMP_LIBRARY="%CONDA_PREFIX%\Library\lib\gmp.lib"
```

### CMake: `Could NOT find cspice`

**Symptom**:
```
Could not find a package configuration file provided by "cspice"
```

**Solution**:
```bash
conda install -c conda-forge cspice
-Dcspice_DIR="%CONDA_PREFIX%\Library\lib\cmake\cspice"
```

### CMake: `python=OFF` in summary

**Symptom**: CMake summary shows `python=OFF` and no `core.pyd` is generated.

**Solution**: Add `-Dkep3_BUILD_PYTHON_BINDINGS=ON` to your CMake command.

### CMake: `Visual Studio 18 2026` not recognized

**Symptom**:
```
CMake Error: Could not create named generator Visual Studio 18 2026
```

**Solution**: Your CMake version is too old (needs ≥ 4.2.0). Update CMake (e.g., from [winlibs](https://winlibs.com/)) and try again.

### `'cl' is not recognized as an internal or external command`

**Symptom**:
```
'cl' is not recognized as an internal or external command, operable program or batch file.
```

**Cause**: You are not running the command in the correct Visual Studio developer command prompt, or the MSVC compiler is not installed.

**Solution**:

1. **Open the correct command prompt** – from the Start menu, search for and open:
   - **"x64 Native Tools Command Prompt for VS 2026"**
   - Do **not** use a regular Command Prompt or PowerShell.

2. **Check that the MSVC compiler is installed** – ensure Visual Studio Build Tools includes the component:
   - **Install Build Tools** (if not already installed):
   ```bash
   winget install Microsoft.VisualStudio.BuildTools
   ```
   - Open Visual Studio Installer, find the installed "Build Tools for Visual Studio 2026", and click Modify.
   - check **"MSVC build tools for x64/x86(latest)"** 

3. **Verify the environment** – after opening the correct prompt, run:
   ```cmd
   cl
   ```
   You should see compiler version information, not an error.


### Python binding imports fail after building

**Symptom**: After copying `core.pyd` and DLLs, `import pykep` fails with `DLL load failed`.

**Solution**:

1. Use `dumpbin /dependents core.pyd` to list all required DLLs.
2. Copy any missing DLLs from `%CONDA_PREFIX%\Library\bin` into your `lib/` folder.
3. Ensure `__init__.py` has `os.add_dll_directory(_lib_dir)`.

### Missing `_version.py` after build

**Symptom**: `ModuleNotFoundError: No module named 'pygmo._version'` or `pykep._version`.

**Solution**: The `_version.py` file is generated during the build. It should exist in `build/pykep/` or `build/pygmo/`. Copy it to the same directory with `core.pyd`.

### `AttributeError: module 'pykep' has no attribute 'udpla'`

**Symptom**: `pk.udpla` raises `AttributeError`.

**Cause**: Your wheel was built without SPICE support. All wheels from this repository include it. If you built it yourself, rebuild with SPICE enabled (see build section below).

---

##  Test Suite Errors

### PyGMO: `free_form_test_case` ERROR

**Symptom**: `pygmo.test.run_test_suite()` shows `ERROR` for `free_form_test_case`.

**Cause**: This test requires `networkx`, which is an optional dependency. The error is expected if `networkx` is not installed.

**Solution**: Install `networkx` if you need `free_form` topology support:
```bash
pip install networkx
```

### PyGMO: `algorithm_test_case` ERROR

**Symptom**: `pygmo.test.run_test_suite()` shows `ERROR` for `algorithm_test_case`.

**Cause**: `scipy` is missing – it's used by the `scipy_optimize` algorithm wrapper.

**Solution**:
```bash
pip install scipy
```
### `INTEL oneMKL ERROR: Cannot load mkl_intel_thread.3.dll`

**Symptoms**:
```
INTEL oneMKL ERROR: Cannot load mkl_intel_thread.3.dll.
Intel oneMKL FATAL ERROR: Cannot load mkl_intel_thread.3.dll.
```

**Cause**: The Intel MKL thread library DLL is missing from the system's search path. This can happen if:
- You built the wheel yourself and did not copy the required MKL DLLs into the `lib/` folder.
- You are running a PyInstaller-packaged application that did not include all MKL dependencies.
- The Conda environment that provided the DLLs is not active.

**Solution**:
1. Copy `mkl_intel_thread.3.dll` from a Conda environment's `Library\bin` into your application's `lib/` folder.
2. Alternatively, ensure that the Conda environment's `Library\bin` is in your system `PATH`:
   ```bash
   set PATH=%CONDA_PREFIX%\Library\bin;%PATH%
   ```

### PyKEP: All 89 tests pass

 `pykep.test.run_test_suite()` should pass **89/89** tests. If any fail, ensure all dependencies (SPICE, etc.) are correctly installed.

---

## 📌 Still stuck?

If none of the above helps, please [open an issue](https://github.com/your-username/windows-pykep-pygmo-build/issues) with:

- **Which category**: Installation or Build?
- **Windows version** and **Python version**
- **Full error traceback**
- **Steps to reproduce**
