# Windows Pre-built Wheels for PyKEP & PyGMO

[![License](https://img.shields.io/badge/License-MPL--2.0-blue.svg)](https://www.mozilla.org/en-US/MPL/2.0/)
[![Python](https://img.shields.io/badge/python-3.14-blue)](https://www.python.org/)
[![Platform](https://img.shields.io/badge/platform-Windows%20x64-lightgrey)](https://www.microsoft.com/en-us/windows)

> **Community-maintained Windows wheels** for [PyKEP](https://github.com/esa/pykep) and [PyGMO](https://github.com/esa/pygmo2).  
> *Not an ESA official release.*

Pre-compiled binary wheels for **Windows 64-bit** – no C++ compiler or conda required. Just `pip install` and start using it.

## Features

- **Self-contained wheels** – includes all required DLLs
- **No source compilation** – works on clean Windows systems
- **All dependencies vendored** – including `heyoka`, SPICE kernels, etc.
- **Minimal Python dependencies** – auto-installed via pip

## Download & Install

### 1. Check System Requirements

Before installing, ensure your system meets the following prerequisites:

| Requirement | Check Command |
|-------------|---------------|
| **Windows 10/11 (64-bit)** | System settings → About |
| **Python 3.14 (64-bit)** | `python --version` |
| **Visual C++ Redistributable** | Check in "Apps & features" for "Microsoft Visual C++ Redistributable" |

```bash
# Verify Python version and architecture
python --version
# Should output: Python 3.14.x

python -c "import struct; print(struct.calcsize('P') * 8)"
# Should output: 64
```

If you don't have Python 3.14 or the VC++ runtime installed:

- **Python 3.14**: Download from [python.org](https://www.python.org/downloads/)
- **VC++ Redistributable**: Download from [Microsoft](https://aka.ms/vs/17/release/vc_redist.x64.exe)

### 2. Download the wheels from [Releases](https://github.com/NonInertialObserver/pykep-pygmo-win-wheels/releases)

| Package | File |
|---------|------|
| PyKEP | `pykep-{version}-cp314-cp314-win_amd64.whl` |
| PyGMO | `pygmo-{version}-cp314-cp314-win_amd64.whl` |

> `{version}` is the release you downloaded — the exact filenames are listed on the [Releases](https://github.com/NonInertialObserver/pykep-pygmo-win-wheels/releases) page.

### 3. Install using pip

> **Important**: PyKEP depends on `pygmo`. On Windows, you must install the provided `pygmo` wheel **first**, then install `pykep`. Installing `pykep` alone makes pip try to build `pygmo` from its PyPI source tarball, which fails on Windows.

```bash
# Step 1: install the downloaded PyGMO wheel
pip install pygmo-{version}-cp314-cp314-win_amd64.whl

# Step 2: install the downloaded PyKEP wheel
pip install pykep-{version}-cp314-cp314-win_amd64.whl
```

### 4. Verify Installation

```python
import pykep as pk
import pygmo as pg

print(f"PyKEP version: {pk.__version__}")
print(f"PyGMO version: {pg.__version__}")
```

### 5. Try SPICE functionality (PyKEP)

```python
import pykep as pk

ud = pk.udpla.de440s("EARTH", "ECLIPJ2000", "SSB")
earth = pk.planet(ud)
print(earth)
```

## Dependencies

To maintain consistency with the official Linux wheels, the dependency list for the Windows wheel follows the same `install_requires` definition as the upstream projects.

### Automatically installed via pip

| Package | Required by | Notes |
|---------|-------------|-------|
| `numpy` | Both | Core numerical library |
| `scipy` | Both | Required by PyKEP and PyGMO |
| `matplotlib` | PyKEP | Plotting utilities |
| `sgp4` | PyKEP | TLE propagation |
| `spiceypy` | PyKEP | SPICE toolkit bindings |
| `pygmo` | PyKEP | PyKEP depends on PyGMO |
| `cloudpickle` | PyKEP (vendored heyoka) | Serialization for heyoka |
| `certifi` | PyKEP (vendored heyoka) | SSL certificates for heyoka |

### Optional Dependencies (PyGMO)

| Package | Usage | Install command |
|---------|-------|-----------------|
| `networkx` | `free_form` topology (and `graph` extra, required by the `free_form` test) | `pip install networkx` |
| `mkl` | Full ipopt solver + BLAS/LAPACK performance in PyGMO (not bundled in the wheel) | `pip install "mkl>=2026"` |

> **ipopt note**: The `ipopt` functionality of PyGMO needs the MKL threading layer (`mkl_intel_thread.3.dll`) and its delay-load dependencies (`mkl_core.3.dll`, `libiomp5md.dll`), which are not bundled in the wheel. Install the optional `mkl` package to enable it. Without it, `import pygmo` works and all core tests pass, but the `ipopt_test` in the suite fails at runtime (the suite may end silently at that test) — this is expected and documented in [`troubleshooting.md`](troubleshooting.md).

### Optional Dependencies (PyKEP)

The `heyoka` integration is vendored inside the wheel, so no additional dependency is needed for that. Any other dependency is auto-installed via pip.

---

## Test Status

| Suite | Status | Notes |
|-------|--------|-------|
| PyKEP | Full upstream test suite passes | Verified on a clean Windows 11 venv (system Python 3.14, no conda) |
| PyGMO | Full upstream test suite passes | Verified on a clean system Python 3.14 venv with `networkx` + `mkl`; `free_form` needs `networkx`, `ipopt` needs the optional `mkl` package (see above) |

> **Note**: The PyGMO `free_form` topology test (`free_form_test_case`) requires `networkx`, which is not declared as a dependency in the official PyGMO metadata (it is optional). This matches the official Linux wheel behavior. After `networkx` (and `mkl` for ipopt) is installed, all tests pass.

## Testing on a Clean Machine

To ensure the wheels work on machines without conda, test using a **clean Python** environment (e.g. `py -3.14`, a system Python, not conda), with `conda|miniconda` stripped from PATH in the test process — otherwise conda's `Library\bin` masks missing DLLs and a "pass" does not prove the wheel is self-contained.

---

## Build & Test Environment

| Item | Details |
|------|---------|
| **Build Host** | Windows 11 Home 25H2 (x64) |
| **Build Tools** | Visual Studio 2026 Build Tools, winlibs CMake (≥4.2), Conda |
| **Test System** | Windows 11 Home 25H2 – clean Python 3.14 venv (PATH stripped of conda) |
| **Python Version** | CPython 3.14.x (64-bit) |

All wheels were verified in a **fresh virtual environment** (`python -m venv`) on the build host.

---

## Documentation

- **Build from source**: See [`BUILD.md`](BUILD.md) for the complete step-by-step guide
- **Troubleshooting**: Check [`troubleshooting.md`](troubleshooting.md)

### Quick Links
- [PyKEP Official Docs](https://esa.github.io/pykep/)
- [PyGMO Official Docs](https://esa.github.io/pygmo2/)
- [SPICE Toolkit](https://naif.jpl.nasa.gov/naif/)

---

## Important Notes

### Python Version Compatibility
These wheels are **strictly for CPython 3.14 (64-bit)**. They will not work with Python 3.13, 3.15, or 32-bit Python.

### System Architecture
- **x64 (AMD64) only** – no x86/32-bit support
- ARM64 is not supported at this time (test hardware not available)

### Installation Order
- PyKEP depends on PyGMO. On Windows, you must install the provided `pygmo` wheel **before** installing `pykep`.

### Third-Party Licenses
This repository contains:
- **PyKEP** / **PyGMO** – [MPL-2.0](https://www.mozilla.org/en-US/MPL/2.0/)
- **heyoka** – [MPL-2.0](https://www.mozilla.org/en-US/MPL/2.0/)
- **CSPICE** – [NAIF JPL](https://naif.jpl.nasa.gov/naif/copyright.html) (free for scientific use)

All bundled binaries retain their original license notices.

---

## Getting Help

If you encounter issues:

1. **Check the FAQ**: [`troubleshooting.md`](troubleshooting.md)
2. **Open an Issue**: Use the [GitHub Issues](https://github.com/NonInertialObserver/pykep-pygmo-win-wheels/issues) page
3. **Include details**: Python version, Windows version, full error traceback

---

## Contributing

While this repository primarily serves as a distribution channel, contributions are welcome!

### How to help
- **Test** – Try the wheels on different Windows versions and report results
- **Document** – Improve build instructions or add examples
- **Build for other Python versions** – 3.12, 3.13, 3.15 if possible

---

## License

This project is licensed under the **MPL-2.0** license – the same as the upstream projects. See the [`LICENSE`](LICENSE) file for details.

---

## Acknowledgements

- **ESA Advanced Concepts Team** – For creating PyKEP and PyGMO
- **NAIF/JPL** – For the SPICE toolkit and DE440 ephemeris
- All upstream contributors and maintainers

---

## Support

If this repository helped you, consider:
- Starring the repo
- Linking it in related discussions/issues
- Sharing your experience with the community

---

*Built for Python 3.14 on Windows x64*