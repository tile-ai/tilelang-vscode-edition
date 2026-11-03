# TileLang LSP

[English version](./lsp.md)

## 概述

`tlsp` 是 TileLang VS Code Edition 使用的 Language Server Protocol 实现。它在后台子进程中
分析 TileLang kernel，提供 buffer 和 kernel 提示、hover 详情及诊断。无法完整编译时，
会逐级简化分析方式，而不是直接停止语言服务。

## 功能

- **行尾提示（inlay hints）**：
  - `alloc`：`T.alloc_*` 的 shape、dtype、scope，以及推断的 `Layout` 或 `Fragment`；
  - `param`：kernel 签名参数；
  - `op`：`T.copy`、`T.gemm` 等操作涉及的布局摘要；
  - `loop`：`T.Pipelined` 和 `T.Parallel` 的流水与布局信息；
  - `kernel`：`T.Kernel` 的 grid 维度和线程数。
- **Hover**：在提示行上悬停可查看完整 layout 或 fragment 文本。
- **诊断**：编译失败时，仍可按不同精度提供诊断。
- **缓存**：使用内存 LRU 和 `~/.cache/tilelang_lsp/` 磁盘缓存，可通过 `TLSP_CACHE_DIR`
  修改位置。缓存键包含文件内容、target 和 TileLang 版本。

## 安装

使用 Python 3.8 或更高版本，在仓库根目录安装：

```bash
python -m pip install -e .
```

安装了 TileLang 的解释器可进行完整分析；没有 TileLang 时，仍可提供基于 AST 的静态
提示和诊断。

构建并安装 VS Code 扩展：

```bash
cd tilelang_lsp_extension
npm ci
npx --no-install vsce package
```

在 VS Code 中选择 **Install from VSIX...**，安装生成的 `tilelang-vscode-edition-*.vsix`。

## VS Code 配置

将 `tilelang.pythonPath` 设置为能够导入 `tlsp` 的解释器。扩展通过该解释器启动
`python -m tlsp`，相关配置变化后会自动重启语言服务。

| 配置项 | 默认值 | 作用 |
|---|---|---|
| `tilelang.pythonPath` | `"python"` | 运行语言服务的 Python 解释器。 |
| `tilelang.enableInlayHints` | `true` | 启用或禁用行尾提示。 |
| `tilelang.serverArgs` | `[]` | 附加服务参数，例如 `["--target", "c"]`。 |
| `tilelang.serverEnv` | `{"TILELANG_ENABLE_DATA_RACE_CHECK": "1"}` | 传给语言服务和 analyzer 的环境变量；自定义值替换整个默认对象。 |

默认的 `tilelang.serverEnv` 开启数据竞争诊断。删除 `TILELANG_ENABLE_DATA_RACE_CHECK`
条目即可关闭。`PYTHONPATH` 由扩展管理。

## `<typecheck>` 指令

analyzer 需要具体的 kernel 实参才能展开 TIR。无法推断实参时，在 kernel 定义附近添加：

```python
# <typecheck> M=1024, N=1024, K=1024, block_M=128, kernel=matmul
```

值通过 `ast.literal_eval` 解析，支持 `"float16"` 等 dtype 字符串。`kernel=` 用于选择
要分析的 kernel，缺省选择第一个 `@tilelang.jit` kernel。

实参按以下顺序解析：

1. 最近且适用的 `<typecheck>` 指令。
2. 从 `<name>.compile(...)` 调用和 kernel 直接调用收集的关键字参数。
3. 函数签名默认值。

分析 target 独立按以下顺序解析：

1. 适用的 `<typecheck>` 指令中的 `target=`。
2. `<kernel>.compile(...)` 调用点的 `target=`。
3. 字面量 `@tilelang.jit(target=...)` 装饰器。
4. 服务参数 `--target` 的默认值；未设置时为 `cuda`。

缺少必需参数时会产生要求补充 `<typecheck>` 的诊断。通过 `T.const("M, N")` 声明的
符号变量可以保持未绑定。

## Neovim

```lua
vim.lsp.config['tlsp'] = {
  cmd = { 'python', '-m', 'tlsp' },  -- 无 CUDA 的机器加 '--target', 'c'
  filetypes = { 'python' },
  root_markers = { 'pyproject.toml', '.git' },
}
vim.lsp.config['tlsp'].capabilities = vim.lsp.protocol.make_client_capabilities()
vim.lsp.enable('tlsp')
```

在 VS Code 之外使用时，在语言服务进程环境中设置 `TILELANG_ENABLE_DATA_RACE_CHECK=1`
可开启数据竞争诊断。

## 命令行分析

直接运行 analyzer，以单行 JSON 输出完整的 schema-v2 分析结果：

```bash
python -m tlsp.analyzer --once path/to/kernel.py --target c
```

## 工作原理

每次打开或保存文件时，analyzer 使用第一种可用的分析模式：

| 模式 | 条件 | 结果 |
|---|---|---|
| `full` | `get_tir` 和前端 pass 成功执行到 LayoutInference | 全部提示，包括推断的 allocation 布局 |
| `partial` | `get_tir` 成功，但后续 pass 失败 | 使用最后一份有效 IR 的提示，以及源码位置的 L2 诊断 |
| `tracer` | IR span 不可用 | 通过 monkeypatch 的 `T.*` 调用 tracer 生成提示，并尽可能按 buffer 名关联布局 |
| `static` | import 错误、代码不完整或其他前端错误导致 `get_tir` 失败 | AST 静态提示、shape mismatch 警告和 traceback 重映射的 L1 诊断 |

如果 analyzer 子进程崩溃或 TileLang 不可用，服务使用静态模式并报告文件级 L3 诊断。
编辑过程中，进程内静态分析会立即提供提示，无需等待后台编译。

## 开发

在仓库根目录运行标准单元测试：

```bash
python -m unittest discover tests -v
```

集成测试需要安装了 TileLang 的解释器：

```bash
TLSP_TEST_PYTHON=/path/to/venv/bin/python python -m unittest tests.test_integration -v
# 使用尚未安装到该环境的源码目录时：
# TLSP_TEST_TILELANG=/path/to/tilelang/repo
```

构建并检查 VS Code 扩展：

```bash
cd tilelang_lsp_extension
npm ci
npm run compile
npm run lint
npm run test:cuda-debug
```

在仓库根目录运行全部 lint hooks：

```bash
pre-commit run --all-files
```

CUDA-GDB 集成测试见 [CUDA 调试指南](./cuda-debugging_zh.md#linux--nvidia-gpu-自动验收)。

## 示例

![TileLang LSP 提示](../tilelang_lsp_extension/assets/demo.png)

无法完整编译时的诊断：

![TileLang LSP 诊断](../tilelang_lsp_extension/assets/partial.png)
