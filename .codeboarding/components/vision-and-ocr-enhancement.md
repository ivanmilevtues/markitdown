---
component_id: 6
component_name: Vision & OCR Enhancement
---

# Vision & OCR Enhancement

## Component Description

An optional layer that adds computer vision and OCR capabilities to extract text from visual elements in documents.

---

## Key References:

### /Users/imilev/StartUp/demo/test/markitdown/packages/markitdown-ocr/src/markitdown_ocr/_ocr_service.py (lines 23-110)
```
class LLMVisionOCRService:
    """OCR service using LLM vision models (OpenAI-compatible)."""

    def __init__(
        self,
        client: Any,
        model: str,
        default_prompt: str | None = None,
    ) -> None:
        """
        Initialize LLM Vision OCR service.

        Args:
            client: OpenAI-compatible client
            model: Model name (e.g., 'gpt-4o', 'gemini-2.0-flash')
            default_prompt: Default prompt for OCR extraction
        """
        self.client = client
        self.model = model
        self.default_prompt = default_prompt or (
            "Extract all text from this image. "
            "Return ONLY the extracted text, maintaining the original "
            "layout and order. Do not add any commentary or description."
        )

    def extract_text(
        self,
        image_stream: BinaryIO,
        prompt: str | None = None,
        stream_info: StreamInfo | None = None,
        **kwargs: Any,
    ) -> OCRResult:
        """Extract text using LLM vision."""
        if self.client is None:
            return OCRResult(
                text="",
                backend_used="llm_vision",
                error="LLM client not configured",
            )

        try:
            image_stream.seek(0)

            content_type: str | None = None
            if stream_info:
                content_type = stream_info.mimetype

            if not content_type:
                try:
                    from PIL import Image

                    image_stream.seek(0)
                    img = Image.open(image_stream)
                    fmt = img.format.lower() if img.format else "png"
                    content_type = f"image/{fmt}"
                except Exception:
                    content_type = "image/png"

            image_stream.seek(0)
            base64_image = base64.b64encode(image_stream.read()).decode("utf-8")
            data_uri = f"data:{content_type};base64,{base64_image}"

            actual_prompt = prompt or self.default_prompt
            response = self.client.chat.completions.create(
                model=self.model,
                messages=[
                    {
                        "role": "user",
                        "content": [
                            {"type": "text", "text": actual_prompt},
                            {
                                "type": "image_url",
                                "image_url": {"url": data_uri},
                            },
                        ],
                    }
                ],
            )

            text = response.choices[0].message.content
            return OCRResult(
                text=text.strip() if text else "",
                backend_used="llm_vision",
            )
        except Exception as e:
            return OCRResult(text="", backend_used="llm_vision", error=str(e))
        finally:
            image_stream.seek(0)
```

### /Users/imilev/StartUp/demo/test/markitdown/packages/markitdown-ocr/src/markitdown_ocr/_pdf_converter_with_ocr.py (lines 129-422)
```
class PdfConverterWithOCR(DocumentConverter):
    """
    Enhanced PDF Converter with OCR support for embedded images.
    Maintains document structure while extracting text from images inline.
    """

    def __init__(self, ocr_service: Optional[LLMVisionOCRService] = None):
        super().__init__()
        self.ocr_service = ocr_service

    def accepts(
        self,
        file_stream: BinaryIO,
        stream_info: StreamInfo,
        **kwargs: Any,
    ) -> bool:
        mimetype = (stream_info.mimetype or "").lower()
        extension = (stream_info.extension or "").lower()

        if extension == ".pdf":
            return True

        if mimetype.startswith("application/pdf") or mimetype.startswith(
            "application/x-pdf"
        ):
            return True

        return False

    def convert(
        self,
        file_stream: BinaryIO,
        stream_info: StreamInfo,
        **kwargs: Any,
    ) -> DocumentConverterResult:
        if _dependency_exc_info is not None:
            raise MissingDependencyException(
                MISSING_DEPENDENCY_MESSAGE.format(
                    converter=type(self).__name__,
                    extension=".pdf",
                    feature="pdf",
                )
            ) from _dependency_exc_info[1].with_traceback(
                _dependency_exc_info[2]
            )  # type: ignore[union-attr]

        # Get OCR service if available (from kwargs or instance)
        ocr_service: LLMVisionOCRService | None = (
            kwargs.get("ocr_service") or self.ocr_service
        )

        # Read PDF into BytesIO
        file_stream.seek(0)
        pdf_bytes = io.BytesIO(file_stream.read())

        markdown_content = []

        try:
            with pdfplumber.open(pdf_bytes) as pdf:
                for page_num, page in enumerate(pdf.pages, 1):
                    markdown_content.append(f"\n## Page {page_num}\n")

                    # If OCR is enabled, interleave text and images by position
                    if ocr_service:
                        images_on_page = self._extract_page_images(pdf_bytes, page_num)

                        if images_on_page:
                            # Extract text lines with Y positions
                            chars = page.chars
                            if chars:
                                # Group chars into lines based on Y position
                                lines_with_y = []
                                current_line = []
                                current_y = None

                                for char in sorted(
                                    chars, key=lambda c: (c["top"], c["x0"])
                                ):
                                    y = char["top"]
                                    if current_y is None:
                                        current_y = y
                                    elif abs(y - current_y) > 2:  # New line threshold
                                        if current_line:
                                            text = "".join(
                                                [c["text"] for c in current_line]
                                            )
                                            lines_with_y.append(
                                                {"y": current_y, "text": text.strip()}
                                            )
                                        current_line = []
                                        current_y = y
                                    current_line.append(char)

                                # Add last line
                                if current_line:
                                    text = "".join([c["text"] for c in current_line])
                                    lines_with_y.append(
                                        {"y": current_y, "text": text.strip()}
                                    )
                            else:
                                # Fallback: use simple text extraction
                                text_content = page.extract_text() or ""
                                lines_with_y = [
                                    {"y": i * 10, "text": line}
                                    for i, line in enumerate(text_content.split("\n"))
                                ]

                            # OCR all images
                            image_data = []
                            for img_info in images_on_page:
                                ocr_result = ocr_service.extract_text(
                                    img_info["stream"]
                                )
                                if ocr_result.text.strip():
                                    image_data.append(
                                        {
                                            "y_pos": img_info["y_pos"],
                                            "name": img_info["name"],
                                            "ocr_text": ocr_result.text,
                                            "backend": ocr_result.backend_used,
                                            "type": "image",
                                        }
                                    )

                            # Add text items
                            content_items = [
                                {
                                    "y_pos": item["y"],
                                    "text": item["text"],
                                    "type": "text",
                                }
                                for item in lines_with_y
                                if item["text"]
                            ]
                            content_items.extend(image_data)

                            # Sort all items by Y position (top to bottom)
                            content_items.sort(key=lambda x: x["y_pos"])

                            # Build markdown by interleaving text and images
                            for item in content_items:
                                if item["type"] == "text":
                                    markdown_content.append(item["text"])
                                else:  # image
                                    ocr_text = item["ocr_text"]
                                    img_marker = (
                                        f"\n\n*[Image OCR]\n{ocr_text}\n[End OCR]*\n"
                                    )
                                    markdown_content.append(img_marker)
                        else:
                            # No images detected - just extract regular text
                            text_content = page.extract_text() or ""
                            if text_content.strip():
                                markdown_content.append(text_content.strip())
                    else:
                        # No OCR, just extract text
                        text_content = page.extract_text() or ""
                        if text_content.strip():
                            markdown_content.append(text_content.strip())

                # Build final markdown
                markdown = "\n\n".join(markdown_content).strip()

                # Fallback to pdfminer if empty
                if not markdown:
                    pdf_bytes.seek(0)
                    markdown = pdfminer.high_level.extract_text(pdf_bytes)

        except Exception:
            # Fallback to pdfminer
            try:
                pdf_bytes.seek(0)
                markdown = pdfminer.high_level.extract_text(pdf_bytes)
            except Exception:
                markdown = ""

        # Final fallback: If still empty/whitespace and OCR is available,
        # treat as scanned PDF and OCR full pages
        if ocr_service and (not markdown or not markdown.strip()):
            pdf_bytes.seek(0)
            markdown = self._ocr_full_pages(pdf_bytes, ocr_service)

        return DocumentConverterResult(markdown=markdown)

    def _extract_page_images(self, pdf_bytes: io.BytesIO, page_num: int) -> list[dict]:
        """
        Extract images from a PDF page using pdfplumber.

        Args:
            pdf_bytes: PDF file as BytesIO
            page_num: Page number (1-indexed)

        Returns:
            List of image info dicts with 'stream', 'bbox', 'name', 'y_pos'
        """
        images = []

        try:
            pdf_bytes.seek(0)
            with pdfplumber.open(pdf_bytes) as pdf:
                if page_num <= len(pdf.pages):
                    page = pdf.pages[page_num - 1]  # 0-indexed
                    images = _extract_images_from_page(page)
        except Exception:
            pass

        # Sort by vertical position (top to bottom)
        images.sort(key=lambda x: x["y_pos"])

        return images

    def _ocr_full_pages(
        self, pdf_bytes: io.BytesIO, ocr_service: LLMVisionOCRService
    ) -> str:
        """
        Fallback for scanned PDFs: Convert entire pages to images and OCR them.
        Used when text extraction returns empty/whitespace results.

        Args:
            pdf_bytes: PDF file as BytesIO
            ocr_service: OCR service to use

        Returns:
            Markdown text extracted from OCR of full pages
        """
        markdown_parts = []

        try:
            pdf_bytes.seek(0)
            with pdfplumber.open(pdf_bytes) as pdf:
                for page_num, page in enumerate(pdf.pages, 1):
                    try:
                        markdown_parts.append(f"\n## Page {page_num}\n")

                        # Render page to image
                        page_img = page.to_image(resolution=300)
                        img_stream = io.BytesIO()
                        page_img.original.save(img_stream, format="PNG")
                        img_stream.seek(0)

                        # Run OCR
                        ocr_result = ocr_service.extract_text(img_stream)

                        if ocr_result.text.strip():
                            text = ocr_result.text.strip()
                            markdown_parts.append(f"*[Image OCR]\n{text}\n[End OCR]*")
                        else:
                            markdown_parts.append(
                                "*[No text could be extracted from this page]*"
                            )

                    except Exception as e:
                        markdown_parts.append(
                            f"*[Error processing page {page_num}: {str(e)}]*"
                        )
                        continue

        except Exception:
            # pdfplumber failed (e.g. malformed EOF) — try PyMuPDF for rendering
            markdown_parts = []
            try:
                import fitz  # PyMuPDF

                pdf_bytes.seek(0)
                doc = fitz.open(stream=pdf_bytes.read(), filetype="pdf")
                for page_num in range(1, doc.page_count + 1):
                    try:
                        markdown_parts.append(f"\n## Page {page_num}\n")
                        page = doc[page_num - 1]
                        mat = fitz.Matrix(300 / 72, 300 / 72)  # 300 DPI
                        pix = page.get_pixmap(matrix=mat)
                        img_stream = io.BytesIO(pix.tobytes("png"))
                        img_stream.seek(0)

                        ocr_result = ocr_service.extract_text(img_stream)

                        if ocr_result.text.strip():
                            text = ocr_result.text.strip()
                            markdown_parts.append(f"*[Image OCR]\n{text}\n[End OCR]*")
                        else:
                            markdown_parts.append(
                                "*[No text could be extracted from this page]*"
                            )

                    except Exception as e:
                        markdown_parts.append(
                            f"*[Error processing page {page_num}: {str(e)}]*"
                        )
                        continue
                doc.close()
            except Exception:
                return "*[Error: Could not process scanned PDF]*"

        return "\n\n".join(markdown_parts).strip()
```


## Source Files:

- `packages/markitdown-ocr/src/markitdown_ocr/_docx_converter_with_ocr.py`
- `packages/markitdown-ocr/src/markitdown_ocr/_ocr_service.py`
- `packages/markitdown-ocr/src/markitdown_ocr/_pdf_converter_with_ocr.py`
- `packages/markitdown-ocr/src/markitdown_ocr/_plugin.py`
- `packages/markitdown-ocr/src/markitdown_ocr/_pptx_converter_with_ocr.py`
- `packages/markitdown-ocr/src/markitdown_ocr/_xlsx_converter_with_ocr.py`

