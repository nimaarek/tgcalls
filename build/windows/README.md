# Build instructions for Windows

- [Build instruction for Windows x32](../../.github/workflows/build_and_publish_win_x32_wheels_for_many_python_versions.yaml).
- [Build instruction for Windows x64](../../.github/workflows/build_and_publish_win_x64_wheels_for_many_python_versions.yaml).

## What artifact do you get (`dll`/`exe`)?

From this repository, the primary Windows output is a **Python extension module**:

- `tgcalls.pyd` (this is a DLL loaded by Python)

It is packaged into wheel files (for example in CI):

- `tgcalls-<version>-cp<pyver>-cp<pyver>-win_amd64.whl`
- `pytgcalls-<version>-py3-none-any.whl` (pure Python package)

So for your question: this repo normally builds a **`.pyd`/DLL-style module inside a wheel**, not a standalone EXE.

## Typical local build flow

1. Build wheel with the existing workflow logic/toolchain.
2. Install wheel in a Python environment.
3. Verify import:

```powershell
python -c "import tgcalls; print(tgcalls.__doc__)"
```

If you need an EXE, that would be a separate wrapper app (Python or C++) that links/loads the built module.
