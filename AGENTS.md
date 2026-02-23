# LightDock Agent Guidelines

LightDock is a macromolecular docking framework based on the Glowworm Swarm Optimization
(GSO) algorithm. Python 3.8+ (tested through 3.12).

## Build & Install

```bash
# Install in editable mode (builds C/Cython extensions automatically)
pip install -e .

# Alternative: compile extensions manually via per-directory compile.sh scripts
./setup.sh

# Dev dependencies (not in pyproject.toml)
pip install pytest pytest-cov flake8
```

Runtime dependencies: numpy, scipy, prody, freesasa, cython.

## Test Commands

Tests live in `lightdock/test/`, mirroring the source tree structure.
Configured in `.pytest.ini` with `testpaths = lightdock/test` and `addopts = -ra`.

```bash
# Run all tests
pytest

# Run with coverage (this is what CI runs)
pytest --cov --cov-branch --cov-report term-missing

# Single test file
pytest lightdock/test/structure/test_atom.py

# Single test class
pytest lightdock/test/structure/test_atom.py::TestAtom

# Single test method
pytest lightdock/test/structure/test_atom.py::TestAtom::test_create_empty_atom

# Tests in a directory
pytest lightdock/test/structure/
```

## Lint

```bash
flake8 lightdock/
```

Config in `.flake8`: max line length 120, max complexity 10,
ignores E501,E128,C901,E203,W503,E722. Note: CI does not run flake8.

## Project Structure

```
lightdock/              Main library package
  error/                Custom exception hierarchy
  gso/                  Glowworm Swarm Optimization algorithm
  mathutil/             Math utilities (with cython/ extensions)
  parallel/             Multiprocessing support
  pdbutil/              PDB file I/O
  post/                 Post-simulation analysis
  prep/                 Simulation setup/preparation
  scoring/              Scoring function plugins (one subpackage each)
  structure/            Molecular structure representations (Atom, Residue, Chain, Complex)
  test/                 Test suite (mirrors source structure)
  constants.py          All default parameters as UPPER_CASE constants
  version.py            CURRENT_VERSION string
bin/                    CLI entry-point scripts (lgd_setup.py, lgd_run.py, etc.)
```

## Code Style

### Imports
- Always use absolute imports: `from lightdock.error.lightdock_errors import AtomError`
- Order: standard library, third-party, local (no isort config; follow existing files)
- One import per line

### String Formatting
- Legacy library code uses %-formatting: `"[%s] %s" % (tag, msg)`
- Newer code and tests use f-strings: `f"Calculating {name}..."`
- `.format()` is rare; avoid introducing it
- Follow whichever style the surrounding code uses

### Classes
- Library classes explicitly inherit from `object`: `class Atom(object):`
- Test classes use bare class syntax: `class TestAtom:`
- Use Python 2-style `super()` in library code: `super(ClassName, self).__init__(...)`
  (exception: `bin/` scripts may use Python 3 bare `super()`)

### Naming
- **Classes**: PascalCase (`Atom`, `DockingModel`, `LightDockError`)
- **Functions/Methods**: snake_case (`is_hydrogen`, `_assign_element`)
- **Constants**: UPPER_CASE (`DEFAULT_NUM_SWARMS`, `MAX_TRANSLATION`)
- **Test classes**: `Test` prefix, PascalCase (`TestAtom`, `TestComplex`)
- **Test methods**: `test_` prefix, snake_case (`test_create_empty_atom`)
- **Private**: leading underscore (`_get_docking_model`, `_write_to_file`)

### Docstrings
- **Module-level**: required, single-line double-quoted: `"""Module to package atom representation"""`
- **Class-level**: required, single-line: `"""Represents a chemical atom"""`
- **Public methods**: short single-line docstrings: `"""Checks if this atom is of hydrogen type"""`
- **Test methods**: generally no docstrings; names should be self-descriptive
- **Test modules**: single-line: `"""Tests for Atom class"""`

### Type Annotations
- Not used in this codebase. Do not add type annotations to existing code.

### Formatting
- Max line length: 120 characters
- Double quotes for strings
- Trailing commas in multi-line argument lists and data structures
- `__init__.py` files are empty (package markers only), except `lightdock/__init__.py`

### Error Handling
- Custom hierarchy in `lightdock/error/lightdock_errors.py`
- Base: `LightDockError(Exception)` with a `cause` attribute
- Domain exceptions: `AtomError`, `StructureError`, `GSOError`, `SetupError`,
  `ScoringFunctionError`, `PDBParsingError`, `NormalModesCalculationError`, etc.
- Always raise the most specific exception with a descriptive message string

## Testing Patterns

- **No `@pytest.fixture` or `conftest.py`**. All setup uses `setup_class(self)` methods.
- **Golden data** files live in `golden_data/` directories adjacent to test files.
- Path construction uses `pathlib.Path`: `self.path = Path(__file__).absolute().parent`
- Float comparisons: `pytest.approx()` — never use `==` for floats
- Exception testing: `pytest.raises(SpecificError)`
- Array comparisons: `numpy.allclose()`
- File comparisons: `filecmp.cmp()` or `lightdock.test.support.compare_two_files()`
- Regression tests use pytest's `tmp_path` fixture for output isolation

Typical test class:
```python
"""Tests for Atom class"""
import pytest
from pathlib import Path
from lightdock.structure.atom import Atom
from lightdock.error.lightdock_errors import AtomError

class TestAtom:
    def setup_class(self):
        self.path = Path(__file__).absolute().parent
        self.golden_data_path = self.path / "golden_data"

    def test_create_empty_atom(self):
        atom = Atom()
        assert 0.0 == pytest.approx(atom.x)

    def test_not_recognized_element(self):
        with pytest.raises(AtomError):
            Atom(1, "Ty", "", "A", "BSG", 1, "", element="Ty")
```

## Scoring Function Plugins

Each scoring function is a subpackage under `lightdock/scoring/<name>/` with a `driver.py`.
Template at `lightdock/scoring/template/driver.py`. Every driver must define:

1. **`ModelAdapter` subclass** — implements `_get_docking_model(self, molecule, restraints)`
2. **`ScoringFunction` subclass** — implements `__call__(self, receptor, receptor_coordinates, ligand, ligand_coordinates)`, must return a float
3. **Two module-level registration variables** (required for dynamic import):
   ```python
   DefinedScoringFunction = MyScoringFunction
   DefinedModelAdapter = MyAdapter
   ```

Optional: a Potential class for loading energy data, a custom DockingModel subclass.

## bin/ Script Patterns

- All scripts start with `#!/usr/bin/env python3`
- Argument parsing: main scripts use classes from `lightdock.util.parser`;
  utility scripts use `argparse.ArgumentParser` directly
- No `main()` functions; logic runs under `if __name__ == "__main__":`
- Logging via `LoggingManager.get_logger("script_name")`
- Top-level try/except catches `LightDockError` and logs it
