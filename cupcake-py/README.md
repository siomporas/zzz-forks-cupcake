# cupcake (Python)

Python bindings for the [Cupcake](https://cupcake.eqtylab.io) policy engine — governance and safety for AI coding agents.

Built with [PyO3](https://pyo3.rs) + [maturin](https://www.maturin.rs), wrapping the same Rust `cupcake-core` engine used by the TypeScript bindings and the CLI.

## Installation

> **Note:** `cupcake` is not yet published to PyPI. Install from source using one of the methods below.

### From source (maturin)

```bash
# Clone the repo
git clone https://github.com/eqtylab/cupcake.git
cd cupcake/cupcake-py

# Build and install in one step (editable/dev mode)
pip install maturin
maturin develop --release

# Or build a wheel and install it
maturin build --release
pip install ../target/wheels/cupcake-*.whl
```

### From source (Nix)

```bash
nix build .#cupcake-py
```

### Using just

```bash
just build-python        # Build bindings
just build-python-wheel  # Build distributable wheel
```

See [Development](#development) for full prerequisites and details.

## Quick Start

### Async (recommended)

```python
from cupcake import Cupcake

cupcake = Cupcake()
await cupcake.init(".cupcake")

decision = await cupcake.evaluate({
    "hookEventName": "PreToolUse",
    "tool_name": "Bash",
    "command": "rm -rf /",
})

if "Deny" in decision:
    print(f"Blocked: {decision['Deny']['reason']}")
```

### Sync (CLI scripts)

```python
from cupcake import Cupcake

cupcake = Cupcake()
cupcake.init_sync(".cupcake")

decision = cupcake.evaluate_sync({
    "hookEventName": "PreToolUse",
    "tool_name": "Bash",
    "command": "ls -la",
})
```

### Module-level API (singleton)

```python
import cupcake

await cupcake.init(".cupcake")
decision = await cupcake.evaluate(event)
```

## API Reference

### `Cupcake` class

| Method | Description |
|---|---|
| `await init(path, harness)` | Initialize engine (async, non-blocking) |
| `init_sync(path, harness)` | Initialize engine (sync, blocks thread) |
| `await evaluate(event)` | Evaluate event (async, non-blocking) |
| `evaluate_sync(event)` | Evaluate event (sync, blocks thread) |
| `version` | Engine version string (property) |
| `is_ready` | Whether engine is ready (property) |

### Parameters

- **path** — Path to project directory or `.cupcake` folder (default: `".cupcake"`)
- **harness** — AI agent harness type: `"claude"`, `"cursor"`, `"factory"`, or `"opencode"` (default: `"claude"`)
- **event** — Dict with hook event data (your policies define the schema)

### Decision Format

The engine returns a dict with the decision variant as key:

```python
# Allow
{"Allow": {"context": ["Looks safe"]}}

# Deny
{"Deny": {"reason": "Destructive command blocked", "agent_messages": []}}

# Ask
{"Ask": {"reason": "Confirm before proceeding", "agent_messages": []}}

# Modify
{"Modify": {"reason": "Sanitized", "updated_input": {...}, "agent_messages": []}}
```

## Development

### Prerequisites

- **Rust toolchain** — Install via [rustup](https://rustup.rs/) (the workspace pins Rust 1.91.0 via `rust-toolchain.toml`)
- **Python 3.9+** — CPython (PyPy is not supported by PyO3)
- **OPA v1.7.1+** — Required for policy compilation. The bindings auto-download it, or install manually:
  ```bash
  # macOS
  brew install opa

  # Linux x86_64 (non-static build required for WASM support)
  curl -L -o opa https://github.com/open-policy-agent/opa/releases/download/v1.7.1/opa_linux_amd64
  chmod +x opa && sudo mv opa /usr/local/bin/
  ```
- **maturin** — The Rust-to-Python build tool:
  ```bash
  pip install maturin
  ```

### Building from source

```bash
# Clone the cupcake repo (if you haven't already)
git clone https://github.com/eqtylab/cupcake.git
cd cupcake

# Install maturin and test dependencies
pip install maturin pytest pytest-asyncio

# Build the native module in development mode (editable install)
cd cupcake-py
maturin develop --release

# Verify it works
python -c "from cupcake import Cupcake; print(Cupcake())"
```

### Using just (recommended)

The workspace [justfile](../justfile) has convenience commands:

```bash
# Build Python bindings (creates venv, installs deps, builds)
just build-python

# Run Python tests
just test-python

# Build a distributable wheel
just build-python-wheel

# Run everything (Rust + TypeScript + Python)
just test-all

# Install all dev dependencies (Rust tools + Python venv)
just install-dev
```

### Using Nix

The `flake.nix` provides a dev shell with all dependencies and a `cupcake-py` package:

```bash
# Enter dev shell (has rust, python, maturin, pytest)
nix develop

# Build the Python wheel via Nix
nix build .#cupcake-py
```

### Running tests

```bash
cd cupcake-py

# Run all tests
pytest tests/ -v

# Run just the error/API tests (no engine needed)
pytest tests/ -v -k "Error or NoInit or repr or version_before"
```

**Platform note:** OPA's static ARM64 Linux builds lack WASM compilation support. On ARM64, tests that require engine initialization will be automatically skipped. All tests pass on x86_64 (the CI target platform).

### Building a wheel

```bash
cd cupcake-py

# Build release wheel for current platform
maturin build --release

# Wheel is in ../target/wheels/
ls ../target/wheels/cupcake-*.whl
```

## Architecture

```
cupcake-py/
├── Cargo.toml              # Rust crate config (PyO3)
├── pyproject.toml           # Python packaging (maturin backend)
├── src/
│   └── lib.rs              # PyO3 bindings → BindingEngine
├── python/
│   └── cupcake/
│       ├── __init__.py      # Python API (class + module-level)
│       ├── _installer.py    # OPA binary auto-installer
│       ├── _native.pyi      # Type stubs for native module
│       └── py.typed         # PEP 561 marker
└── tests/
    ├── conftest.py          # pytest config
    └── test_basic.py        # Test suite (mirrors cupcake-ts tests)
```

The Python wrapper follows the same pattern as the TypeScript bindings:

1. **Rust layer** (`src/lib.rs`) — PyO3 `PolicyEngine` class wrapping `cupcake_core::bindings::BindingEngine`
2. **Python layer** (`python/cupcake/`) — Ergonomic `Cupcake` wrapper with async support via `ThreadPoolExecutor`, error handling, and OPA auto-installation

### How it maps to the TS bindings

| TypeScript (`cupcake-ts`) | Python (`cupcake-py`) |
|---|---|
| NAPI-RS (`napi`, `napi-derive`) | PyO3 (`pyo3`) |
| `src/lib.rs` → `#[napi] PolicyEngine` | `src/lib.rs` → `#[pyclass] PolicyEngine` |
| `index.ts` → `Cupcake` class | `__init__.py` → `Cupcake` class |
| `installer.ts` → OPA auto-download | `_installer.py` → OPA auto-download |
| `evaluateAsync` → libuv threadpool | `evaluate()` → `ThreadPoolExecutor` |
| `evaluateSync` → blocks event loop | `evaluate_sync()` → blocks thread |

## License

MIT
