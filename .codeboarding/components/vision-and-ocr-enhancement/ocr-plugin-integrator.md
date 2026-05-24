---
component_id: 6.3
component_name: OCR Plugin Integrator
---

# OCR Plugin Integrator

## Component Description

Manages the registration and discovery of OCR-enhanced converters, ensuring they take precedence when the OCR plugin is active.

---

## Key References:

### /Users/imilev/StartUp/demo/test/markitdown/packages/markitdown-ocr/src/markitdown_ocr/_plugin.py (lines 19-68)
```
def register_converters(markitdown: MarkItDown, **kwargs: Any) -> None:
    """
    Register OCR-enhanced converters with MarkItDown.

    This plugin provides OCR support for PDF, DOCX, PPTX, and XLSX files.
    The converters are registered with priority -1.0 to run BEFORE built-in
    converters (which have priority 0.0), effectively replacing them when
    the plugin is enabled.

    Args:
        markitdown: MarkItDown instance to register converters with
        **kwargs: Additional keyword arguments that may include:
            - llm_client: OpenAI-compatible client for LLM-based OCR (required for OCR to work)
            - llm_model: Model name (e.g., 'gpt-4o')
            - llm_prompt: Custom prompt for text extraction
    """
    # Create OCR service — reads the same llm_client/llm_model kwargs
    # that MarkItDown itself already accepts for image descriptions
    llm_client = kwargs.get("llm_client")
    llm_model = kwargs.get("llm_model")
    llm_prompt = kwargs.get("llm_prompt")

    ocr_service: LLMVisionOCRService | None = None
    if llm_client and llm_model:
        ocr_service = LLMVisionOCRService(
            client=llm_client,
            model=llm_model,
            default_prompt=llm_prompt,
        )

    # Register converters with priority -1.0 (before built-ins at 0.0)
    # This effectively "replaces" the built-in converters when plugin is installed
    # Pass the OCR service to each converter's constructor
    PRIORITY_OCR_ENHANCED = -1.0

    markitdown.register_converter(
        PdfConverterWithOCR(ocr_service=ocr_service), priority=PRIORITY_OCR_ENHANCED
    )

    markitdown.register_converter(
        DocxConverterWithOCR(ocr_service=ocr_service), priority=PRIORITY_OCR_ENHANCED
    )

    markitdown.register_converter(
        PptxConverterWithOCR(ocr_service=ocr_service), priority=PRIORITY_OCR_ENHANCED
    )

    markitdown.register_converter(
        XlsxConverterWithOCR(ocr_service=ocr_service), priority=PRIORITY_OCR_ENHANCED
    )
```


## Source Files:

- `packages/markitdown-ocr/src/markitdown_ocr/_docx_converter_with_ocr.py`
- `packages/markitdown-ocr/src/markitdown_ocr/_ocr_service.py`
- `packages/markitdown-ocr/src/markitdown_ocr/_pdf_converter_with_ocr.py`
- `packages/markitdown-ocr/src/markitdown_ocr/_plugin.py`
- `packages/markitdown-ocr/src/markitdown_ocr/_pptx_converter_with_ocr.py`
- `packages/markitdown-ocr/src/markitdown_ocr/_xlsx_converter_with_ocr.py`

