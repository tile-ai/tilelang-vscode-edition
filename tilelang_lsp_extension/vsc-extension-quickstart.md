# Extension Development

## Install dependencies

From `tilelang_lsp_extension/`, install the locked Node dependencies:

```bash
npm ci
```

Install the `tlsp` Python package from the repository root as well:

```bash
python -m pip install -e .
```

## Start an Extension Development Host

1. Run `npm run compile`.
2. Open `src/extension.ts` in VS Code.
3. Press F5 to launch a new VS Code window with the extension loaded.
4. Open a TileLang Python file in that window and configure `tilelang.pythonPath` if needed.

Use `npm run watch` while editing TypeScript, then reload the Extension Development Host to
pick up changes. Set breakpoints in the extension source to debug the client itself.

## Checks

Run the standard checks before submitting changes:

```bash
npm run compile
npm run lint
npm run test:cuda-debug
```

From the repository root, also run the Python tests and lint hooks:

```bash
python -m unittest discover tests -v
pre-commit run --all-files
```

See `docs/lsp.md` and `docs/cuda-debugging.md` in the repository for server-side development
and CUDA-GDB integration testing.
