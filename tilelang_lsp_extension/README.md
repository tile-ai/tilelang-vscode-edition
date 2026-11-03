# TileLang VS Code Edition Extension

This directory contains the Visual Studio Code client for TileLang language support and CUDA
device debugging.

## Features

- TileLang LSP integration with inlay hints, hover details, and diagnostics.
- CUDA debugging through NVIDIA Nsight and `cuda-gdb`, with breakpoints and device stepping
  mapped to TileLang Python source lines.
- Commands for debugging the active kernel and browsing generated CUDA sources.

## Build and install

Install the Python package from the repository root:

```bash
python -m pip install -e .
```

Build the extension from this directory:

```bash
npm ci
npx --no-install vsce package
```

In VS Code, choose **Install from VSIX...** and select the generated
`tilelang-vscode-edition-*.vsix` file. Set `tilelang.pythonPath` to an interpreter that can
import `tlsp`. If you installed the earlier `tilelang-type-decorator` package, uninstall it
before installing the renamed extension to avoid duplicate commands.

For CUDA debugging, also install [NVIDIA Nsight VS Code Edition](https://marketplace.visualstudio.com/items?itemName=nvidia.nsight-vscode-edition).
Remote-SSH and WSL require the Python package, TileLang extension, Nsight, and CUDA tools on
the remote side.

## Usage

### Language support

Open a TileLang Python file to enable hints and diagnostics. The main settings are:

- `tilelang.pythonPath`: Python interpreter used to start `python -m tlsp`;
- `tilelang.enableInlayHints`: enable or disable inlay hints;
- `tilelang.serverArgs`: additional server arguments;
- `tilelang.serverEnv`: environment variables for the server and analyzer.

See [the LSP guide](../docs/lsp.md) for configuration, `<typecheck>`, and target resolution.

### CUDA debugging

Install Nsight, set a breakpoint on a TileLang device statement, and run
**TileLang: Debug CUDA Kernel**. Use **TileLang: Open Generated CUDA Source** to inspect the
generated `.c` and `.cu` files.

The `tilelang.debug.*` settings control the `cuda-gdb` path, device debug information, launch
stops, script arguments, environment variables, and generated-source directory. See the
[CUDA debugging guide](../docs/cuda-debugging.md) for requirements and details.

## Development

```bash
npm run compile
npm run lint
npm run test:cuda-debug
```

Open `src/extension.ts` in VS Code and press F5 to start an Extension Development Host. See
[the extension development guide](./vsc-extension-quickstart.md) for the development workflow.

Changes are recorded in [CHANGELOG.md](./CHANGELOG.md).
