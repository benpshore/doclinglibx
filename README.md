# doclinglibx

Personal library/experiments built on top of [Docling](https://docling-project.github.io/docling/) (`docling>=2.123.1`), IBM's open-source document conversion toolkit. This README is a working reference for the Docling Python API surface most likely to be used in this project — not the full upstream docs.

Upstream docs: https://docling-project.github.io/docling/

> **2026-09-02** — Added this API summary and pinned `transformers>=5.10.0` via a `uv` override to fix CVE-2026-9856 (docling's own dependency pin caps it below the patched version). — Claude (Sonnet 5)
>
> **2026-09-02** — No conversion pipeline configured yet; source is a stub. — Claude (Sonnet 5)
>
> **2026-09-02** — Researched docling-serve REST API; Heron/TableFormer-accurate already default. — Claude (Sonnet 5)
>
> **2026-09-02** — Verified docling API against installed 2.123.1; docs were stale. — Claude (Opus 5)
>
> **2026-09-02** — `docling.service_client` ships in-package; docling-serve needs no extra install. — Claude (Opus 5)
>
> **2026-09-02** — Docling has no FFI; cross-language means HTTP, MCP, or subprocess. — Claude (Opus 5)
>
> **2026-09-02** — Reading order is a fixed rule-based step, not user-configurable. — Claude (Sonnet 5)

## Install

```bash
uv sync
```

## Quick start

```python
from docling.document_converter import DocumentConverter

source = "https://arxiv.org/pdf/2408.09869"  # local path, URL, or DocumentStream
converter = DocumentConverter()
doc = converter.convert(source).document

print(doc.export_to_markdown())
```

The core workflow is always two steps: **convert** a source into a `DoclingDocument`, then **use** that document (export it, chunk it, feed it to a RAG pipeline, etc.).

## `DocumentConverter`

```python
from docling.document_converter import DocumentConverter

DocumentConverter(
    allowed_formats: list[InputFormat] | None = None,   # restrict accepted input formats (default: all)
    format_options: dict[InputFormat, FormatOption] | None = None,  # per-format pipeline config
)
```

### Methods

| Method | Signature | Notes |
|---|---|---|
| `convert` | `convert(source, headers=None, raises_on_error=True, max_num_pages=maxsize, max_file_size=maxsize, page_range=DEFAULT_PAGE_RANGE) -> ConversionResult` | Convert a single `Path`, `str` (path or URL), `DocumentStream`, or `HttpSource`. |
| `convert_all` | `convert_all(source: Iterable[...], **same kwargs) -> Iterator[ConversionResult]` | Batch conversion, yields results lazily. |
| `convert_string` | `convert_string(content: str, format: InputFormat, name=None) -> ConversionResult` | Convert in-memory Markdown/HTML/DocLang text (no file needed). |
| `initialize_pipeline` | `initialize_pipeline(format: InputFormat)` | Warms up/pre-loads a format's pipeline (e.g. before timing conversions). |

`ConversionResult.document` is the resulting `DoclingDocument`; `ConversionResult.status` reports success/partial/failure when `raises_on_error=False`.

### Per-format options

Wrap a `PipelineOptions` subclass in a `*FormatOption` and register it per `InputFormat`:

```python
from docling.datamodel.accelerator_options import AcceleratorDevice, AcceleratorOptions
from docling.datamodel.base_models import InputFormat
from docling.datamodel.pipeline_options import PdfPipelineOptions, TableStructureOptions
from docling.document_converter import DocumentConverter, PdfFormatOption

pipeline_options = PdfPipelineOptions()
pipeline_options.do_ocr = True
pipeline_options.do_table_structure = True
pipeline_options.table_structure_options = TableStructureOptions(do_cell_matching=True)
pipeline_options.ocr_options.lang = ["es"]
pipeline_options.accelerator_options = AcceleratorOptions(
    num_threads=4, device=AcceleratorDevice.AUTO
)

converter = DocumentConverter(
    format_options={
        InputFormat.PDF: PdfFormatOption(pipeline_options=pipeline_options)
    }
)

result = converter.convert("some.pdf", page_range=(1, 5))
```

## Supported input formats (`InputFormat`)

| Category | Formats |
|---|---|
| Documents | PDF, DOCX/DOC, XLSX/XLS, PPTX/PPT, ODT/ODS/ODP, EPUB, Markdown, AsciiDoc, LaTeX, HTML/XHTML, CSV |
| Images | PNG, JPEG, TIFF, BMP, WEBP |
| Audio | WAV, MP3, M4A, AAC, OGG, FLAC (via ASR pipeline) |
| Video | MP4, AVI, MOV (via video pipeline) |
| Structured/XML | DocLang (`.dclg`, `.dclg.xml`), DocLang archive (`.dclx`), USPTO XML, JATS XML, XBRL XML, custom XML backends |
| Other | Apple Pages, BoxNote, Email (`.eml`, `.msg`), EBCDIC, Docling JSON, WebVTT |

## Export / output

Every `DoclingDocument` supports:

- `export_to_markdown()` — Markdown text (`strict_text=True` for plain text without formatting)
- `export_to_html()` — HTML
- `export_to_dict()` — plain dict → `json.dumps(...)` for lossless JSON
- `export_to_document_tokens()` / `export_to_doctags()` — DocTags representation
- Plain text, DocLang XML, WebVTT, LaTeX, and chunk (JSONL) export are also available depending on document type.

```python
doc.export_to_markdown()
doc.export_to_html()
json.dumps(doc.export_to_dict())
doc.export_to_doctags()
```

## Pipeline options (`docling.datamodel.pipeline_options`)

All pipeline option classes derive from `PipelineOptions`, which carries the common fields:

`accelerator_options`, `artifacts_path`, `document_timeout`, `enable_remote_services`, `allow_external_plugins`.

### `PdfPipelineOptions` (most commonly used)

Key toggles:

- `do_ocr` — run OCR on the page
- `do_table_structure` — run table structure recognition
- `do_code_enrichment` / `do_formula_enrichment` — extract code blocks / math formulas
- `do_picture_classification` / `do_picture_description` — classify/caption images
- `generate_page_images` / `generate_picture_images` / `generate_table_images` — render raster images into the output
- `images_scale` — resolution multiplier for generated images
- `table_structure_options` — a `TableStructureOptions(mode, do_cell_matching)`
- `ocr_options` — one of the OCR engine option classes below
- `layout_options`, `heading_hierarchy_options`, `chart_extraction_options`, `code_formula_options`, `picture_description_options`, `picture_classification_options`
- `force_backend_text`, `generate_parsed_pages`
- Threaded/batch tuning: `layout_batch_size`, `ocr_batch_size`, `table_batch_size`, `queue_max_size`, `batch_polling_interval_seconds`, `stage_shutdown_timeout_seconds`

Other pipeline variants exist for different processing modes: `VlmPipelineOptions` (vision-language-model pipeline), `AsrPipelineOptions` (audio transcription), `VideoPipelineOptions` (frame sampling/scene detection/diarization), `ConvertPipelineOptions`, `PaginatedPipelineOptions`, `ThreadedPdfPipelineOptions`.

### OCR engines (`ocr_options`)

| Class | Backend | Notable fields |
|---|---|---|
| `EasyOcrOptions` | EasyOCR | `lang`, `use_gpu`, `confidence_threshold`, `model_storage_directory` |
| `TesseractOcrOptions` | Tesseract (python binding) | `lang`, `psm`, `path` |
| `TesseractCliOcrOptions` | Tesseract (CLI) | `lang`, `psm`, `tesseract_cmd` |
| `RapidOcrOptions` | RapidOCR | `det_model_path`, `rec_model_path`, `cls_model_path`, `backend` |
| `NemotronOcrOptions` | NVIDIA Nemotron | `batch_size`, `merge_level` |
| `OcrMacOptions` | macOS Vision framework | `framework`, `recognition` |
| `KserveV2OcrOptions` | Remote KServe v2 endpoint | `url`, `model_name`, `transport`, `grpc_*` |
| `OcrAutoOptions` | **Default.** Auto-selects an available engine | `lang` |

Shared OCR fields: `lang`, `scale`, `mode` (`OcrMode`), `force_full_page_ocr`.

> **Verified 2026-09-02 against the installed docling 2.123.1**, not the docs:
> `PdfPipelineOptions().ocr_options` is `OcrAutoOptions`, **not** `EasyOcrOptions`.
> Docling's own bundled skill (`docling/.agents/skills/docling/references/python-sdk.md`)
> still says "default engine EasyOCR" — that is stale. `OcrAutoModel` picks in order:
> **OcrMac → Nemotron → RapidOCR → EasyOCR → RapidOCR**. In this venv only RapidOCR is
> installed, so `auto` resolves to RapidOCR on the `torch` backend.
>
> **Caveat:** on first use RapidOCR downloads its own weights from `modelscope.cn`
> **into `site-packages/rapidocr/models/`** — it ignores `artifacts_path` and is not
> covered by `docling-tools models download`. Plan for it in offline/air-gapped setups.

`OcrMode` enum: `DEFAULT`, `FULL_PAGE`, `LAYOUT_REGIONS`, `PDF_AWARE_LAYOUT_REGIONS`.

```python
from docling.datamodel.pipeline_options import OcrMode, TesseractCliOcrOptions

pipeline_options.ocr_options = TesseractCliOcrOptions(mode=OcrMode.FULL_PAGE)
```

### Table structure

- `TableStructureOptions(mode: TableFormerMode, do_cell_matching: bool)`
- `TableFormerMode`: `FAST` | `ACCURATE`
- `GraniteVisionTableStructureOptions` — VLM-based alternative

### Accelerator options

```python
from docling.datamodel.accelerator_options import AcceleratorDevice, AcceleratorOptions

AcceleratorOptions(num_threads=4, device=AcceleratorDevice.AUTO)
```

`AcceleratorDevice`: `AUTO`, `CPU`, `CUDA`, `MPS`.

### PDF backend

`PdfBackend` enum (choice of PDF parsing engine): `DLPARSE_V1`, `DLPARSE_V2`, `DLPARSE_V4`, `DOCLING_PARSE`, `PYPDFIUM2`, `THREADED_DOCLING_PARSE`.

### Processing pipeline kind

`ProcessingPipeline` enum: `STANDARD`, `VLM`, `ASR`, `LEGACY`.

## Chunking (`docling.chunking`)

For RAG/embedding workflows, split a `DoclingDocument` into chunks that carry structural metadata (headings, captions, provenance).

- **`HierarchicalChunker`** — one chunk per detected document element (merges list items by default). No tokenizer needed.
- **`HybridChunker`** — wraps the hierarchical chunker with a tokenizer-aware pass: splits oversized chunks and merges undersized adjacent chunks that share headings.
- **`LineBasedTokenChunker`** — line-oriented, tokenization-aware alternative.

```python
from docling.document_converter import DocumentConverter
from docling.chunking import HybridChunker
from docling_core.transforms.chunker.tokenizer.huggingface import HuggingFaceTokenizer
from transformers import AutoTokenizer

doc = DocumentConverter().convert("source.pdf").document

tokenizer = HuggingFaceTokenizer(
    tokenizer=AutoTokenizer.from_pretrained("sentence-transformers/all-MiniLM-L6-v2"),
    max_tokens=64,
)

chunker = HybridChunker(tokenizer=tokenizer, merge_peers=True)

for chunk in chunker.chunk(dl_doc=doc):
    print(chunk.text)                    # raw chunk text
    print(chunker.contextualize(chunk))  # text enriched with heading/caption context
```

`HybridChunker` options: `merge_peers` (default `True`), `repeat_table_header` (default `True`), `omit_header_on_overflow` (default `False`).

## Packaging: `docling` is a meta-package

`docling` 2.123.1 has exactly one dependency: `docling-slim[standard]`. All the real
code and optional deps live in `docling-slim`, selected via extras.

| Goal | Install |
|---|---|
| Batteries-included (what this repo has) | `docling` == `docling-slim[standard]` |
| Talk to a remote service only, zero local models | `docling-slim[service-client]` |
| PDF → md/json, no ML | `docling-slim[format-pdf,cli]` |
| Local VLM pipeline | `docling-slim[models-vlm-inline]` |
| Everything optional | `docling-slim[all]` |

`standard` bundles: `format-pdf`, `models-local`, `feat-ocr-rapidocr`, `format-office`,
`format-web`, `format-latex`, `format-email`, `feat-chunking`, `extract-core`,
`service-client`, `cli`.

Extras available on the `docling` meta-package itself: `asr`, `easyocr`, `htmlrender`,
`ocrmac`, `onnxruntime`, `rapidocr`, `remote-serving`, `tesserocr`, `vlm`, `xbrl`.

**Not installed here** (verified): `easyocr`, `ocrmac`, `tesserocr`, `onnxruntime`,
`mlx` / `mlx_vlm`, `vllm`, `whisper`, and every RAG framework loader. On Apple Silicon
the two most valuable additions are `docling[ocrmac]` (native Vision OCR, no download)
and `docling[vlm]` + `mlx-vlm` (local VLM on the GPU).

## The three deployment surfaces

Docling exposes the same conversion through three swappable surfaces. They share the
`ConversionResult` return type, so calling code does not change.

### 1. In-process (local models)

`DocumentConverter` — everything above. Models run on this machine from
`~/.cache/docling/models`.

### 2. Remote service — `docling.service_client` (built in, no extra package)

`docling` already ships a full sync **and** async client for a `docling-serve`
endpoint. This is the "local API / cloud API" surface: self-hosted `docling-serve`,
or a managed one (e.g. Docling for IBM watsonx). Only the URL and key differ.

```python
from docling.service_client import DoclingServiceClient

client = DoclingServiceClient(url="http://localhost:5001", api_key="")
result = client.convert("report.pdf")          # same shape as DocumentConverter.convert
print(result.document.export_to_markdown())
```

Reads `DOCLING_SERVICE_URL` / `DOCLING_SERVICE_API_KEY` from the environment or a `.env`.

| Method | Purpose |
|---|---|
| `convert` / `convert_all` | Mirror of `DocumentConverter`; `convert_all` takes `max_concurrency` |
| `chunk` / `submit_chunk` | Remote chunking, `ChunkerKind.HYBRID` or `.HIERARCHICAL` |
| `submit`, `submit_batch`, `submit_and_retrieve_each/_many` | Async job handles (`ConversionJob`) |
| `health`, `version` | Service probes |

Per-request config uses **`ConvertDocumentsOptions`** (`docling.datamodel.service.options`),
a *different, flatter* model than `PdfPipelineOptions` — it is the wire format. Notable
fields: `to_formats`, `ocr_engine`, `ocr_preset`, `table_mode`, `pdf_backend`, `pipeline`,
`image_export_mode`, `chunking_options`, `vlm_pipeline_model_api`, `picture_description_api`,
and `*_preset` / `*_custom_config` pairs for each stage.

Result targets for batch jobs: `InBodyTarget`, `ZipTarget`, `PresignedUrlTarget`,
`S3Target`, `AzureBlobTarget`, `GoogleCloudStorageTarget`, `GoogleDriveTarget`.

Async: `AsyncDoclingServiceClient`. Errors: typed, all under `DoclingServiceClientError`
(`ConversionError`, `ServiceUnavailableError`, `TaskTimeoutError`, `UsageLimitExceededError`,
`ResultExpiredError`, …).

CLI equivalent: `docling convert-remote report.pdf --service-url … --to md --output /tmp/`.

### 3. Remote *models* inside a local pipeline

Keep parsing local, send only page images or figure crops to an OpenAI-compatible
endpoint (Ollama, LM Studio, vLLM, OpenAI, or any hosted API).

```python
from docling.datamodel.pipeline_options import VlmPipelineOptions
from docling.datamodel.pipeline_options_vlm_model import ApiVlmOptions, ResponseFormat
from docling.document_converter import DocumentConverter, PdfFormatOption
from docling.datamodel.base_models import InputFormat
from docling.pipeline.vlm_pipeline import VlmPipeline

vlm = ApiVlmOptions(
    url="http://localhost:11434/v1/chat/completions",
    params={"model": "ibm/granite-docling:258m", "max_tokens": 4096},
    headers={"Authorization": "Bearer ..."},     # omit if unauthenticated
    prompt="Convert this page to docling.",
    response_format=ResponseFormat.DOCTAGS,
    timeout=120,
)
opts = VlmPipelineOptions(
    vlm_options=vlm,
    generate_page_images=True,
    enable_remote_services=True,   # REQUIRED — docling blocks outbound HTTP otherwise
)
DocumentConverter(format_options={
    InputFormat.PDF: PdfFormatOption(pipeline_cls=VlmPipeline, pipeline_options=opts)
})
```

`enable_remote_services=False` is the default and is a hard gate: any API-backed stage
raises rather than making a request. Same gate applies to
`PictureDescriptionApiOptions` (captioning figures via an API) and `KserveV2OcrOptions`
(remote OCR over KServe v2, HTTP or gRPC).

Newer engine-based equivalent: `ApiVlmEngineOptions` in
`docling.datamodel.vlm_engine_options`, with `VlmEngineType` ∈
`transformers`, `mlx`, `vllm`, `api`, `api_ollama`, `api_lmstudio`, `api_openai`,
`auto_inline`. It fills in the right default URL per variant
(Ollama `:11434`, LM Studio `:1234`, OpenAI `api.openai.com`).

## Model catalog and presets

Models are named constants, not free-form strings.

| Stage | Where | Examples |
|---|---|---|
| Layout | `docling.datamodel.layout_model_specs` | `DOCLING_LAYOUT_HERON` (default), `_HERON_101`, `_EGRET_MEDIUM/_LARGE/_XLARGE`, `DOCLING_LAYOUT_V2` |
| Table | `TableStructureOptions` / `TableStructureV2Options` | `TableFormerMode.ACCURATE` (default) / `FAST`; `GraniteVisionTableStructureOptions` |
| VLM convert | `docling.datamodel.vlm_model_specs` | `GRANITEDOCLING_{TRANSFORMERS,MLX,VLLM,OLLAMA,VLLM_API}`, `SMOLDOCLING_*`, `NANONETS_OCR2_*`, `GLMOCR_*`, `LIGHTONOCR_*`, `PIXTRAL_12B_*`, `QWEN25_VL_3B_MLX`, `GEMMA3_{12B,27B}_MLX`, `PHI4_TRANSFORMERS`, `GOT2_TRANSFORMERS`, `DOLPHIN_TRANSFORMERS`, `DEEPSEEKOCR_OLLAMA` |
| ASR | `docling.datamodel.asr_model_specs` | `WHISPER_{TINY,BASE,SMALL,MEDIUM,LARGE,TURBO}` × `{_NATIVE,_MLX,_S2T}`, `WHISPER_DISTIL_*` |
| Picture description | `PictureDescriptionVlmEngineOptions.from_preset(...)` | `smolvlm` (default), `granite_vision`, `pixtral`, `qwen` |
| Code/formula | `CodeFormulaVlmOptions.from_preset(...)` | `codeformulav2` (default), `granite_docling` |
| Picture classification | `DocumentPictureClassifierOptions.from_preset(...)` | `document_figure_classifier_v2` |

`VlmConvertOptions.list_presets()` returns 19 preset ids: `smoldocling`, `granite_docling`,
`deepseek_ocr`, `granite_vision`, `pixtral`, `got_ocr`, `phi4`, `qwen`, `nanonets_ocr2`,
`gemma_12b`, `gemma_27b`, `dolphin`, `glm_ocr`, `lightonocr`, `falcon_ocr`, `chandra_ocr2`,
`unlimited_ocr`, `dots_ocr`, `dots_mocr`.

Only `PictureDescriptionVlmEngineOptions`, `CodeFormulaVlmOptions`,
`DocumentPictureClassifierOptions` and `VlmConvertOptions` implement
`from_preset()` / `list_presets()`. `LayoutOptions`, `VlmPipelineOptions`,
`AsrPipelineOptions` and the OCR classes do **not** — set `model_spec` directly.

### Offline / pinned models

```bash
docling-tools models download --all -o ./models          # or: layout tableformer …
```

Then `PdfPipelineOptions(artifacts_path="./models")`. Downloadable ids: `layout`,
`tableformer`, `tableformerv2`, `code_formula`, `picture_classifier`, `smolvlm`,
`granitedocling`, `granitedocling_mlx`, `smoldocling`, `smoldocling_mlx`,
`granite_vision`, `granite_chart_extraction`, `granite_chart_extraction_v4`,
`rapidocr`, `easyocr`, `nemotron_ocr_v2`. (This machine's cache is already ~28 GB.)

## Extending Docling from other Python packages

Docling discovers third-party models through the **`docling` setuptools entry-point
group**. A package declares:

```toml
[project.entry-points.docling]
my_plugin = "my_pkg.docling_plugin"
```

exposing any of `ocr_engines()`, `layout_engines()`, `table_structure_engines()`,
`picture_description()`, each returning `{"<name>": [ModelClass, ...]}`.

Plugins outside the `docling*` namespace are **refused** unless the pipeline sets
`allow_external_plugins=True`. Factories live in `docling.models.factories`
(`get_ocr_factory`, `get_layout_factory`, `get_table_structure_factory`,
`get_picture_description_factory`) and are `lru_cache`d on that flag.

### RAG framework loaders

| Framework | Package | Component |
|---|---|---|
| LangChain | `langchain-docling` | `DoclingLoader` (`ExportType.DOC_CHUNKS` / `.MARKDOWN`) |
| LlamaIndex | `llama-index-readers-docling`, `llama-index-node-parser-docling` | `DoclingReader` + node parser |
| Haystack | `docling-haystack` | converter component |

None are installed here.

## Structured extraction (`DocumentExtractor`, beta)

Pull typed fields out instead of the whole document — takes a Pydantic model, a dict,
or a prompt string as the template.

```python
from docling.document_extractor import DocumentExtractor
result = DocumentExtractor().extract("invoice.pdf", template=MyPydanticModel)
```

Backed by `VlmExtractionPipelineOptions` / `NU_EXTRACT_2B_TRANSFORMERS`.

## MCP — `docling-mcp`

Separate package (**not installed here**), makes Docling agentic over the Model Context
Protocol.

```bash
uvx --from=docling-mcp docling-mcp-server --transport [stdio|sse|streamable-http]
```

MCP client config:

```json
{
  "mcpServers": {
    "docling": {
      "command": "uvx",
      "args": ["--from=docling-mcp", "docling-mcp-server"]
    }
  }
}
```

- `pip install docling-mcp` — **remote mode**, ~50 MB, no model downloads; delegates to a
  `docling-serve` instance.
- `pip install docling-mcp[local]` — local conversion, optional fallback remote → local.
- Env config, all `DOCLING_MCP_`-prefixed: `CONVERSION_MODE`, `SERVICE_URL`,
  `SERVICE_API_KEY`, `SERVICE_TIMEOUT`, `SERVICE_MAX_RETRIES`, `FALLBACK_TO_LOCAL`,
  `KEEP_IMAGES`, `IMAGES_SCALE`, `DO_OCR`, `DO_TABLE_STRUCTURE`, `IMAGE_EXPORT_MODE`,
  plus LlamaIndex (`LI_*`) and LlamaStack (`LLS_*`) RAG settings.
- Tools cover conversion (`convert_document`, `convert_document_with_images`,
  `extract_tables`, `convert_batch`), document generation
  (`create_new_docling_document`, `add_title_to_docling_document`,
  `open_list_in_docling_document`, `add_listitem_to_list_in_docling_document`,
  `close_list_in_docling_document`, `export_docling_document_to_markdown`), and
  optional Milvus / LlamaIndex / LlamaStack RAG tools.

There is **no documented Python API for embedding the MCP server** — integration is
over the MCP protocol only.

## FFI: there isn't one

Worth stating plainly, because it changes the integration plan:

- Docling is **pure Python**. The only native artifact in the whole dependency tree is
  `docling_parse/pdf_parsers.cpython-314-darwin.so`, a **pybind11** module — a CPython
  extension, not a C ABI you can `dlopen` from Rust/Go/Swift.
- There is no C header, no `cdecl` surface, no `cffi`/`ctypes` entry point.

To call Docling from a non-Python process, use one of:

| Approach | Mechanism |
|---|---|
| **HTTP** (recommended) | `docling-serve` REST; any language |
| **MCP** | `docling-mcp` over stdio / SSE / streamable-http |
| **Subprocess** | `docling in.pdf --to json --output dir/` |
| **Embed CPython** | PyO3 / cgo / `Python.h` — you are embedding an interpreter, not FFI-ing a library |

## Docling ships its own agent skill

`docling/.agents/skills/docling/` inside the installed package: `SKILL.md` plus
`references/{cli,python-sdk,extraction,rag,service-client,slim-packaging}.md`.
Authoritative and versioned with the library — but confirmed stale in at least one
place (the EasyOCR default). Treat it as a strong hint, verify against the objects.

## Notation conventions used in Docling's API

- **`InputFormat`** — enum identifying a source document type (`InputFormat.PDF`, `.DOCX`, `.HTML`, ...); used both to restrict `allowed_formats` and as keys in `format_options`.
- **`*FormatOption`** (e.g. `PdfFormatOption`) — per-format wrapper pairing a backend with its `PipelineOptions`.
- **`*PipelineOptions`** — dataclass/pydantic-model configuring a whole pipeline (OCR, tables, enrichment, resource limits).
- **`*Options`** suffix on smaller classes (e.g. `EasyOcrOptions`, `TableStructureOptions`) — configuration for one sub-component of a pipeline, assigned onto a field of a `*PipelineOptions` instance.
- **`do_*` booleans** — feature toggles (`do_ocr`, `do_table_structure`, `do_picture_description`, ...).
- **`*Mode` / `*Device` / `*Backend` enums** — closed sets of algorithm/hardware choices (`OcrMode`, `TableFormerMode`, `AcceleratorDevice`, `PdfBackend`).
- Many option classes expose `from_preset()` / `get_preset()` / `list_presets()` for choosing a bundled model configuration instead of hand-setting every field (used heavily by VLM- and model-backed options).

## References

- Docling docs: https://docling-project.github.io/docling/
- Usage guide: https://docling-project.github.io/docling/usage/
- `DocumentConverter` reference: https://docling-project.github.io/docling/reference/document_converter/
- Pipeline options reference: https://docling-project.github.io/docling/reference/pipeline_options/
- Supported formats: https://docling-project.github.io/docling/usage/supported_formats/
- Examples index: https://docling-project.github.io/docling/examples/
- Chunking concepts: https://docling-project.github.io/docling/concepts/chunking/
