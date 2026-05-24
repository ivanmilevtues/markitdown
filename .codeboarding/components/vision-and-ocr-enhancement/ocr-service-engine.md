---
component_id: 6.1
component_name: OCR Service Engine
---

# OCR Service Engine

## Component Description

The central abstraction layer that manages the interaction with LLM vision models, converting raw image data into text descriptions or OCR results.

---

## Key References:

### /Users/imilev/StartUp/demo/test/markitdown/packages/markitdown-ocr/src/markitdown_ocr/_ocr_service.py (lines 14-20)
```
class OCRResult:
    """Result from OCR extraction."""

    text: str
    confidence: float | None = None
    backend_used: str | None = None
    error: str | None = None
```


## Source Files:

- `packages/markitdown-ocr/src/markitdown_ocr/_ocr_service.py`
- `packages/markitdown-ocr/src/markitdown_ocr/_pdf_converter_with_ocr.py`

