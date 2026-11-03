# TileLang LSP

[中文版](./lsp_zh.md)

## Overview

`tlsp` is the Language Server Protocol implementation used by TileLang VS Code Edition. It
provides buffer and kernel hints, hover details, and diagnostics by analyzing TileLang kernels
in a background subprocess. If full compilation is unavailable, it degrades to progressively
simpler analysis instead of disabling language features.

## Features

- **Inlay hints** at the end of a line:
  - `alloc`: shape, dtype, scope, and inferred `Layout` or `Fragment` for `T.alloc_*` calls;
  - `param`: kernel signature parameters;
  - `op`: layout summaries for operations such as `T.copy` and `T.gemm`;
  - `loop`: pipelining and layout information for `T.Pipelined` and `T.Parallel`;
  - `kernel`: grid dimensions and thread counts for `T.Kernel`.
- **Hover** on a hint line displays the complete layout or fragment text.
- **Diagnostics** remain available at different precision levels when compilation fails.
- **Caching** uses an in-memory LRU and a disk cache under `~/.cache/tilelang_lsp/`. Set
  `TLSP_CACHE_DIR` to override the location. Entries are keyed by file content, target, and
  TileLang version.

## Installation

Use Python 3.8 or later and install the package from the repository root:

```bash
python -m pip install -e .
```

An interpreter with TileLang installed enables full analysis. Without TileLang, the server
still provides static AST-based hints and diagnostics.

Build and install the VS Code extension:

```bash
cd tilelang_lsp_extension
npm ci
npx --no-install vsce package
```

Choose **Install from VSIX...** in VS Code and select the generated
`tilelang-vscode-edition-*.vsix` file.

## VS Code configuration

Set `tilelang.pythonPath` to the interpreter that can import `tlsp`. The extension starts
`python -m tlsp` with that interpreter and restarts the server when relevant settings change.

| Setting | Default | Purpose |
|---|---|---|
| `tilelang.pythonPath` | `"python"` | Python interpreter for the server. |
| `tilelang.enableInlayHints` | `true` | Enable or disable inlay hints. |
| `tilelang.serverArgs` | `[]` | Additional server arguments, such as `["--target", "c"]`. |
| `tilelang.serverEnv` | `{"TILELANG_ENABLE_DATA_RACE_CHECK": "1"}` | Environment variables inherited by the server and analyzer. Custom values replace the whole object. |

The default `tilelang.serverEnv` enables data-race diagnostics. Remove the
`TILELANG_ENABLE_DATA_RACE_CHECK` entry to opt out. `PYTHONPATH` is managed by the extension.

## The `<typecheck>` directive

The analyzer needs concrete kernel arguments to elaborate TIR. Add a directive near the kernel
definition when arguments cannot be inferred:

```python
# <typecheck> M=1024, N=1024, K=1024, block_M=128, kernel=matmul
```

Values are parsed with `ast.literal_eval`, including dtype strings such as `"float16"`.
`kernel=` selects which kernel to analyze; by default, the first `@tilelang.jit` kernel is used.

Arguments are resolved in this order:

1. The nearest applicable `<typecheck>` directive.
2. Keyword arguments collected from `<name>.compile(...)` calls and direct kernel calls.
3. Function signature defaults.

The analysis target is resolved independently in this order:

1. `target=` in an applicable `<typecheck>` directive.
2. `target=` at a `<kernel>.compile(...)` call site.
3. A literal `@tilelang.jit(target=...)` decorator.
4. The server `--target` default, which is `cuda` when unset.

Missing required parameters produce a diagnostic requesting a `<typecheck>` directive. Symbolic
variables declared with `T.const("M, N")` may remain unbound.

## Neovim

```lua
vim.lsp.config['tlsp'] = {
  cmd = { 'python', '-m', 'tlsp' },  -- add '--target', 'c' on CUDA-free machines
  filetypes = { 'python' },
  root_markers = { 'pyproject.toml', '.git' },
}
vim.lsp.config['tlsp'].capabilities = vim.lsp.protocol.make_client_capabilities()
vim.lsp.enable('tlsp')
```

Outside VS Code, set `TILELANG_ENABLE_DATA_RACE_CHECK=1` in the server process environment to
enable data-race diagnostics.

## Command-line analysis

Run the analyzer directly to print the complete schema-v2 result as one JSON line:

```bash
python -m tlsp.analyzer --once path/to/kernel.py --target c
```

## How it works

On every open or save, the analyzer uses the first available analysis mode:

| Mode | Condition | Result |
|---|---|---|
| `full` | `get_tir` and frontend passes through LayoutInference succeed | All hints, including inferred allocation layouts |
| `partial` | `get_tir` succeeds but a later pass fails | Hints from the last valid IR state and an L2 diagnostic at the source location |
| `tracer` | IR spans are unavailable | Hints from a monkeypatched `T.*` call tracer, with layouts joined by buffer name when available |
| `static` | `get_tir` fails because of import errors, incomplete code, or other frontend errors | AST-based hints, shape-mismatch warnings, and an L1 diagnostic remapped from the traceback |

If the analyzer subprocess crashes or TileLang is unavailable, the server uses static mode and
reports a file-level L3 diagnostic. While you type, in-process static analysis provides hints
without waiting for background compilation.

## Development

Run the standard unit suite from the repository root:

```bash
python -m unittest discover tests -v
```

Integration tests require an interpreter with TileLang installed:

```bash
TLSP_TEST_PYTHON=/path/to/venv/bin/python python -m unittest tests.test_integration -v
# When using a source checkout that is not installed in that environment:
# TLSP_TEST_TILELANG=/path/to/tilelang/repo
```

Build and check the VS Code extension:

```bash
cd tilelang_lsp_extension
npm ci
npm run compile
npm run lint
npm run test:cuda-debug
```

Run all configured lint hooks from the repository root:

```bash
pre-commit run --all-files
```

The CUDA-GDB integration test is documented in the
[CUDA debugging guide](./cuda-debugging.md#automated-linux--nvidia-gpu-acceptance).

## Examples

![TileLang LSP hints](../tilelang_lsp_extension/assets/demo.png)

Diagnostics when full compilation is unavailable:

![TileLang LSP diagnostics](../tilelang_lsp_extension/assets/partial.png)
