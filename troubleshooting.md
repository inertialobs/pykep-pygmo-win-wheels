# Troubleshooting

This document covers common issues when **installing and running** the pre-built wheels, as well as **building from source**.

## Quick Navigation

- **Installing the pre-built wheels?** → Start with [Installation & Import Issues](#installation--import-issues)
- **Building from source?** → Start with [Build Issues](#build-issues)

---

## Installation & Import Issues

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

> If you only want to check pykep itself, you can use `pip install --no-deps` to skip dependency resolution; but pykep's `trajopt` module still needs pygmo at runtime.

### `ModuleNotFoundError: No module named 'pygmo'`

**Symptom**: `import pykep` fails with `ModuleNotFoundError: No module named 'pygmo'`.

**Solution**: Install the `pygmo` wheel first (see above).

### `ModuleNotFoundError: No module named 'numpy/sgp4/etc.'`

**Symptom**: On a clean machine, `import pykep` reports a different `ModuleNotFoundError` for each missing package, in turn: `numpy` → `sgp4` → `spiceypy` → `cloudpickle` → `scipy` → `matplotlib` → `pygmo`.

**Cause**: pykep's METADATA declares 8 dependencies; they are easy to miss when using `--no-deps` or when not following the "install pygmo first" order:

```
Requires-Dist: numpy, scipy, matplotlib, sgp4, spiceypy, pygmo, cloudpickle, certifi
```

**Solution**: Install all of them (after the pygmo wheel is installed first, pip resolves the rest automatically):
```bash
pip install numpy scipy matplotlib sgp4 spiceypy cloudpickle certifi
```

> These pure-Python dependencies are unrelated to DLLs; a missing one is an import error, not `DLL load failed`.

### `ImportError: DLL load failed`

**Symptom**:
```
ImportError: DLL load failed while importing core: The specified module could not be found.
```

**Possible causes and solutions**:

1. **Missing Visual C++ Redistributable**
   Download and install from [Microsoft](https://aka.ms/vs/17/release/vc_redist.x64.exe).
   > The system-wide VC++ Redistributable provides `msvcp140_1/2`, `vcruntime140_1`, `vcomp`, etc., complementing the `msvcp140.dll` shipped in the wheel. Verified on a clean machine: after removing the extra VC runtime parts, ipopt still works — so the wheels do **not** need the full VC runtime bundled.

2. **Missing DLLs in the `lib/` folder**
   Check if `pykep/lib/` or `pygmo/lib/` contains `.dll` files. If not, the wheel was corrupted during download – re-download it.

3. **`os.add_dll_directory` not called / wheel without the patch**
   Ensure the `__init__.py` file inside the package contains the DLL loading code. (Included in our wheels by default.)
   > `add_dll_directory` alone is **not enough** for pygmo: `asl.dll` / the Fortran runtime are MSVC **delay-load**, which only recognizes the standard search path — `__init__.py` must also **prepend `pygmo\lib`** (and the detected `<venv>\Library\bin`) to `PATH`. See [BUILD.md](BUILD.md) §4.3.

4. **Missing `mkl_rt.3.dll` (pygmo, required at import)**
   On a clean environment (no conda, no `mkl_rt`), even `import pygmo` fails: `libblas.dll`/`liblapack.dll` are conda's thin forwarding layers and do a runtime `LoadLibrary("mkl_rt.3.dll")` **at load time**. The shipped wheel includes `mkl_rt`, so import works; when minimizing manually, **must keep `mkl_rt`**.

### `FileNotFoundError: tops/_tops_cr3bp.json`

**Symptom**:
```
FileNotFoundError: .../pykep/trajopt/gym/tops/_tops_cr3bp.json
```

**Cause**: JSON data files missing from the wheel.

**Solution**: Re-download the wheel from Releases. If you built it yourself, ensure your `setup.py` includes:
```python
package_data={"pykep": ["trajopt/gym/tops/*.json"]}
```

### `ModuleNotFoundError: No module named 'networkx'`

**Symptom**: `pygmo.test.run_test_suite()`'s `free_form_test_case` reports ERROR (all others pass).

**Cause**: `networkx` is an **optional** dependency of pygmo (the `graph` extra); the wheel does not force it.

**Solution**:
```bash
pip install networkx
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

## Build Issues

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

**Solution**: Add `-Dkep3_BUILD_PYTHON_BINDINGS=ON` to your CMake command. (This is pykep's real option; pygmo has no such option — see the next item.)

### CMake: `Manually-specified variables were not used by the project: pygmo_BUILD_PYTHON_BINDINGS`

**Symptom**: When configuring pygmo with `-Dpygmo_BUILD_PYTHON_BINDINGS=ON`, CMake prints `Manually-specified variables were not used by the project`.

**Cause**: pygmo2's CMakeLists has **no** such option (it only has `PYGMO_ENABLE_IPO`); the Python bindings are built **unconditionally**. The option was confused with pykep's `kep3_BUILD_PYTHON_BINDINGS`.

**Solution**: Drop that parameter; it does not affect the build.

### CMake: `Visual Studio 18 2026` not recognized

**Symptom**:
```
CMake Error: Could not create named generator Visual Studio 18 2026
```

**Solution**: Your CMake is too old (need ≥ 4.2.0) — but the **most common cause is using the conda env's cmake**: `kep3_devel.yml` pins `cmake>=3.28,<3.31` (the 3.30.5 inside the env does not know this generator). Always configure with the winlibs CMake **full path**, not with the activated env's `cmake`:

```powershell
& "$cmake_path" .. -G "Visual Studio 18 2026" -A x64 ...
```

### `'cl' is not recognized as an internal or external command`

**Symptom**:
```
'cl' is not recognized as an internal or external command, operable program or batch file.
```

**Cause / Solution**:
Configuring and building **do not** need `cl` on PATH — CMake's VS generator locates MSVC itself (vswhere/MSBuild), so a plain PowerShell/terminal can run every step in BUILD.md. Only when using `dumpbin` do you need the VS environment:
> ```cmd
> cmd /c "call vcvars64.bat && dumpbin /dependents core.pyd"
> ```

### Python binding imports fail after building

**Symptom**: After copying `core.pyd` and the DLLs, `import pykep` fails with `DLL load failed`.

**Solution**:
1. Use `dumpbin /dependents core.pyd` to list all required DLLs.
2. Copy any missing DLLs from `%CONDA_PREFIX%\Library\bin` into your `lib/` folder.
3. Ensure `__init__.py` has the DLL search patch (add_dll_directory + PATH prepend, see BUILD.md).

> Note: dumpbin's `Image has the following delay load dependencies` section must also be treated as required (ipopt → asl/Fortran; mkl_intel_thread → mkl_core/libiomp5md). The **"file locking" method** (deleting unlocked DLLs after import) wrongly removes these delay-load DLLs — it is only a first pass and must be cross-checked with dumpbin.

### `_version.py` after build

**Symptom**: `ModuleNotFoundError: No module named 'pykep._version'` (or `pygmo._version`).

**Solution**: The two differ in where the file is generated:
- **pykep**: `_version.py` is generated by `configure_file` **directly in the source directory `pykep\`** (no copy needed);
- **pygmo**: it is generated in **`build\pygmo\`**, so copy it: `copy build\pygmo\_version.py ..\pygmo\`.

### `AttributeError: module 'pykep' has no attribute 'udpla'`

**Symptom**: `pk.udpla` raises `AttributeError`.

**Cause**: Your wheel was built without SPICE support. All wheels from this repository include it. If you built it yourself, rebuild with SPICE enabled (see build section above).

---

## Test Suite Errors

### The verification environment must be a "clean Python"

**Symptom**: Everything passes in a conda-based venv, but on a normal machine import/ipopt crashes.

**Cause**: conda's python (or a leftover `...\Library\bin` in PATH) "accidentally" provides the missing DLLs — **masking an incomplete wheel**. Any test done in such an environment is unreliable.

**Solution**: Create the venv with a **system Python** (e.g. `py -3.14`) and strip `conda|miniconda` entries from PATH in the test process. Only if the wheel passes in this kind of environment does it represent a normal user machine.

### PyGMO: `free_form_test_case` ERROR

**Symptom**: `pygmo.test.run_test_suite()` shows `ERROR` for `free_form_test_case`.

**Cause**: This test requires `networkx`, which is an optional dependency (`graph` extra). The error is expected if `networkx` is not installed.

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

### `INTEL oneMKL ERROR / Cannot load mkl_intel_thread.3.dll` (pygmo's ipopt test)

**Symptoms**:
```
INTEL oneMKL ERROR: Cannot load mkl_intel_thread.3.dll.
Intel oneMKL FATAL ERROR: Cannot load mkl_intel_thread.3.dll.
```

**Cause**: This is **expected behavior, not a wheel defect**. The MKL chain is:
```
libblas/liblapack (thin forwarding layer) → mkl_rt.3.dll (dispatcher, at import) → at runtime LoadLibrary mkl_intel_thread.3.dll (threading layer, delay-load mkl_core.3.dll + libiomp5md.dll)
```

The shipped **pygmo** wheel includes `mkl_rt.3.dll` (the dispatcher), but does **not** bundle the threading layer `mkl_intel_thread.3.dll` or its delay-load dependencies `mkl_core.3.dll` / `libiomp5md.dll` (provided by the optional `mkl` package; the pykep wheel does bundle the threading layer — its delay-load risk is covered in the next section). Therefore, without `mkl`:
- `import` works (the dispatcher is present);
- when reaching ipopt, the process **fails at that test** (no result line for that test, and the following suite summary does not appear; in some cases the process terminates silently without an error message).

**Solution**: Install the PyPI `mkl` package (same distribution as conda; it automatically pulls intel-openmp/tbb; the DLLs land in `<venv>\Library\bin`, which the wheel's `__init__.py` automatically adds to PATH):
```bash
pip install "mkl>=2026"
```
After that, the ipopt test **PASSED** (Solve_Succeeded, verified on a clean environment).

| mkl installed? | import | ipopt_test_case |
|---|---|---|
| No | ✅ | fails (no result line, no summary; may terminate silently) — expected |
| `pip install "mkl>=2026"` | ✅ |  PASSED |

### Diagnostic hint: the suite "ends without output" at ipopt

`runTest (...ipopt_test_case.runTest) ...` prints and then the test run **ends directly, with no summary**: this is the ipopt MKL dependency missing (delay-load failure) terminating the process — expected, see above. After installing `mkl>=2026` the test runs to completion and prints the summary.

### pykep's `mkl_core` / `libiomp5md` delay-load risk

**Background**: `kep3.dll` has a **PE direct dependency** on `mkl_intel_thread.3.dll` (pykep's own dependency, not pygmo's); that DLL itself delay-loads `mkl_core.3.dll` + `libiomp5md.dll`, **which are not in the pykep wheel**.

**Current status**:
- `import pykep` works; the official 89-item test suite passes **89/89 with no anomaly** (delay-load is lazy; the tests never reach the mkl_core path).
- **Risk** (real but not triggered): user code that calls a routine falling through to mkl_core (some linear-algebra/FFT paths) triggers the Windows delay-load failure exception (**0xC06D007F**) and the process terminates.

**Mitigation**:
- Users with the PyPI `mkl` installed: `mkl_core.3.dll` / `libiomp5md.dll` are in `<venv>\Library\bin` (on the process PATH) → the risk disappears automatically;
- Users without `mkl`: the current test paths all pass; the risk only exists in uncovered call paths;
- Full defense: add `mkl_core.3.dll` (68.6MB) + `libiomp5md.dll` (0.15MB) to the pykep wheel (+68.8MB); this is currently not done.

### PyKEP: all 89 tests pass

`pykep.test.run_test_suite()` passes **89/89** tests (verified on a clean environment). If any fail, ensure all dependencies (SPICE, etc.) are correctly installed and the Python deps (numpy/scipy/matplotlib/sgp4/spiceypy/pygmo/cloudpickle/certifi) are present.

---

## Still stuck?

If none of the above helps, please [open an issue](https://github.com/NonInertialObserver/pykep-pygmo-win-wheels/issues) with:

- **Which category**: Installation or Build?
- **Windows version** and **Python version**
- **Full error traceback**
- **Steps to reproduce**