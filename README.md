# doclinglibx

Personal library/experiments built on top of [Docling](https://docling-project.github.io/docling/) (`docling>=2.123.1`), IBM's open-source document conversion toolkit. This README is a working reference for the Docling Python API surface most likely to be used in this project — not the full upstream docs.

Upstream docs: https://docling-project.github.io/docling/

> **2026-09-02** — Added this API summary and pinned `transformers>=5.10.0` via a `uv` override to fix CVE-2026-9856 (docling's own dependency pin caps it below the patched version). — Claude (Sonnet 5)
>
> **2026-09-02** — No conversion pipeline configured yet; source is a stub. — Claude (Sonnet 5)
>
> **2026-09-02** — Researched docling-serve REST API; Heron/TableFormer-accurate already default. — Claude (Sonnet 5)
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
| `EasyOcrOptions` | EasyOCR (default) | `lang`, `use_gpu`, `confidence_threshold`, `model_storage_directory` |
| `TesseractOcrOptions` | Tesseract (python binding) | `lang`, `psm`, `path` |
| `TesseractCliOcrOptions` | Tesseract (CLI) | `lang`, `psm`, `tesseract_cmd` |
| `RapidOcrOptions` | RapidOCR | `det_model_path`, `rec_model_path`, `cls_model_path`, `backend` |
| `NemotronOcrOptions` | NVIDIA Nemotron | `batch_size`, `merge_level` |
| `OcrMacOptions` | macOS Vision framework | `framework`, `recognition` |
| `KserveV2OcrOptions` | Remote KServe v2 endpoint | `url`, `model_name`, `transport`, `grpc_*` |
| `OcrAutoOptions` | Auto-selects an available engine | `lang` |

Shared OCR fields: `lang`, `scale`, `mode` (`OcrMode`), `force_full_page_ocr`.

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
