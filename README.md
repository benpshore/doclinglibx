# doclinglibx

A lightweight project for configuring and extending [IBM Docling](https://github.com/docling-project/docling) — document conversion, MCP tooling, and a local serve API.

## Stack

- Python 3.14, managed entirely with [`uv`](https://docs.astral.sh/uv/)
- Apple Silicon (ARM64) macOS as the primary target
- [`docling`](https://github.com/docling-project/docling), [`docling-mcp`](https://github.com/docling-project/docling-mcp), and [`docling-serve`](https://github.com/docling-project/docling-serve)

PyTorch (a Docling dependency) ships with Metal/MPS support on Apple Silicon, so GPU acceleration is available when running on real Apple Silicon hardware without sandbox restrictions. This repo does not configure or force MPS itself — it relies on Docling's/PyTorch's own device selection. Verify with:

```sh
uv run python -c "import torch; print(torch.backends.mps.is_available())"
```

## Security note

`pyproject.toml` pins `transformers>=5.10.0` via `[tool.uv].override-dependencies`. This overrides docling/docling-core's own `transformers<5.9.0` upper bound to force in the patched release for [GHSA-xrqw-3rrv-vx5w](https://github.com/advisories/GHSA-xrqw-3rrv-vx5w) (CVE-2026-9856, a path traversal in `save_pretrained()`). Re-verify Docling's OCR/VLM/chunking paths whenever `docling` is upgraded, in case the override introduces an incompatibility.

## Setup

```sh
uv sync
```

## Usage

```sh
uv run docling --help
uv run docling-mcp-server --help
uv run docling-serve --help
```

## Development

```sh
uv run ruff check .
uv run pytest
```
