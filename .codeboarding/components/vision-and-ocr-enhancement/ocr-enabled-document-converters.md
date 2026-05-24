---
component_id: 6.2
component_name: OCR-Enabled Document Converters
---

# OCR-Enabled Document Converters

## Component Description

A collection of format-specific converters (PDF, PPTX, XLSX, DOCX) that identify visual elements and delegate text extraction to the OCR Service Engine.

---

## Key References:

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

### /Users/imilev/StartUp/demo/test/markitdown/packages/markitdown-ocr/src/markitdown_ocr/_pptx_converter_with_ocr.py (lines 27-249)
```
class PptxConverterWithOCR(DocumentConverter):
    """Enhanced PPTX Converter with OCR fallback."""

    def __init__(self, ocr_service: Optional[LLMVisionOCRService] = None):
        super().__init__()
        self._html_converter = HtmlConverter()
        self.ocr_service = ocr_service

    def accepts(
        self,
        file_stream: BinaryIO,
        stream_info: StreamInfo,
        **kwargs: Any,
    ) -> bool:
        mimetype = (stream_info.mimetype or "").lower()
        extension = (stream_info.extension or "").lower()

        if extension == ".pptx":
            return True

        if mimetype.startswith(
            "application/vnd.openxmlformats-officedocument.presentationml"
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
                    extension=".pptx",
                    feature="pptx",
                )
            ) from _dependency_exc_info[1].with_traceback(
                _dependency_exc_info[2]
            )  # type: ignore[union-attr]

        # Get OCR service (from kwargs or instance)
        ocr_service: Optional[LLMVisionOCRService] = (
            kwargs.get("ocr_service") or self.ocr_service
        )
        llm_client = kwargs.get("llm_client")

        presentation = pptx.Presentation(file_stream)
        md_content = ""
        slide_num = 0

        for slide in presentation.slides:
            slide_num += 1
            md_content += f"\\n\\n<!-- Slide number: {slide_num} -->\\n"

            title = slide.shapes.title

            def get_shape_content(shape, **kwargs):
                nonlocal md_content

                # Pictures
                if self._is_picture(shape):
                    # Get image data
                    image_stream = io.BytesIO(shape.image.blob)

                    # Try LLM description first if available
                    llm_description = ""
                    if llm_client and kwargs.get("llm_model"):
                        try:
                            from ._llm_caption import llm_caption

                            image_filename = shape.image.filename
                            image_extension = None
                            if image_filename:
                                import os

                                image_extension = os.path.splitext(image_filename)[1]

                            image_stream_info = StreamInfo(
                                mimetype=shape.image.content_type,
                                extension=image_extension,
                                filename=image_filename,
                            )

                            llm_description = llm_caption(
                                image_stream,
                                image_stream_info,
                                client=llm_client,
                                model=kwargs.get("llm_model"),
                                prompt=kwargs.get("llm_prompt"),
                            )
                        except Exception:
                            pass

                    # Try OCR if LLM failed or not available
                    ocr_text = ""
                    if not llm_description and ocr_service:
                        try:
                            image_stream.seek(0)
                            ocr_result = ocr_service.extract_text(image_stream)
                            if ocr_result.text.strip():
                                ocr_text = ocr_result.text.strip()
                        except Exception:
                            pass

                    # Format extracted content using unified OCR block format
                    content = (llm_description or ocr_text or "").strip()
                    if content:
                        md_content += f"\n*[Image OCR]\n{content}\n[End OCR]*\n"

                # Tables
                if self._is_table(shape):
                    md_content += self._convert_table_to_markdown(shape.table, **kwargs)

                # Charts
                if shape.has_chart:
                    md_content += self._convert_chart_to_markdown(shape.chart)

                # Text areas
                elif shape.has_text_frame:
                    if shape == title:
                        md_content += "# " + shape.text.lstrip() + "\\n"
                    else:
                        md_content += shape.text + "\\n"

                # Group Shapes
                if shape.shape_type == pptx.enum.shapes.MSO_SHAPE_TYPE.GROUP:
                    sorted_shapes = sorted(
                        shape.shapes,
                        key=lambda x: (
                            float("-inf") if not x.top else x.top,
                            float("-inf") if not x.left else x.left,
                        ),
                    )
                    for subshape in sorted_shapes:
                        get_shape_content(subshape, **kwargs)

            sorted_shapes = sorted(
                slide.shapes,
                key=lambda x: (
                    float("-inf") if not x.top else x.top,
                    float("-inf") if not x.left else x.left,
                ),
            )
            for shape in sorted_shapes:
                get_shape_content(shape, **kwargs)

            md_content = md_content.strip()

            if slide.has_notes_slide:
                md_content += "\\n\\n### Notes:\\n"
                notes_frame = slide.notes_slide.notes_text_frame
                if notes_frame is not None:
                    md_content += notes_frame.text
                md_content = md_content.strip()

        return DocumentConverterResult(markdown=md_content.strip())

    def _is_picture(self, shape):
        if shape.shape_type == pptx.enum.shapes.MSO_SHAPE_TYPE.PICTURE:
            return True
        if shape.shape_type == pptx.enum.shapes.MSO_SHAPE_TYPE.PLACEHOLDER:
            if hasattr(shape, "image"):
                return True
        return False

    def _is_table(self, shape):
        if shape.shape_type == pptx.enum.shapes.MSO_SHAPE_TYPE.TABLE:
            return True
        return False

    def _convert_table_to_markdown(self, table, **kwargs):
        import html

        html_table = "<html><body><table>"
        first_row = True
        for row in table.rows:
            html_table += "<tr>"
            for cell in row.cells:
                if first_row:
                    html_table += "<th>" + html.escape(cell.text) + "</th>"
                else:
                    html_table += "<td>" + html.escape(cell.text) + "</td>"
            html_table += "</tr>"
            first_row = False
        html_table += "</table></body></html>"

        return (
            self._html_converter.convert_string(html_table, **kwargs).markdown.strip()
            + "\\n"
        )

    def _convert_chart_to_markdown(self, chart):
        try:
            md = "\\n\\n### Chart"
            if chart.has_title:
                md += f": {chart.chart_title.text_frame.text}"
            md += "\\n\\n"
            data = []
            category_names = [c.label for c in chart.plots[0].categories]
            series_names = [s.name for s in chart.series]
            data.append(["Category"] + series_names)

            for idx, category in enumerate(category_names):
                row = [category]
                for series in chart.series:
                    row.append(series.values[idx])
                data.append(row)

            markdown_table = []
            for row in data:
                markdown_table.append("| " + " | ".join(map(str, row)) + " |")
            header = markdown_table[0]
            separator = "|" + "|".join(["---"] * len(data[0])) + "|"
            return md + "\\n".join([header, separator] + markdown_table[1:])
        except ValueError as e:
            if "unsupported plot type" in str(e):
                return "\\n\\n[unsupported chart]\\n\\n"
        except Exception:
            return "\\n\\n[unsupported chart]\\n\\n"
```

### /Users/imilev/StartUp/demo/test/markitdown/packages/markitdown-ocr/src/markitdown_ocr/_xlsx_converter_with_ocr.py (lines 27-225)
```
class XlsxConverterWithOCR(DocumentConverter):
    """
    Enhanced XLSX Converter with OCR support for embedded images.
    Extracts images with their cell positions and performs OCR.
    """

    def __init__(self, ocr_service: Optional[LLMVisionOCRService] = None):
        super().__init__()
        self._html_converter = HtmlConverter()
        self.ocr_service = ocr_service

    def accepts(
        self,
        file_stream: BinaryIO,
        stream_info: StreamInfo,
        **kwargs: Any,
    ) -> bool:
        mimetype = (stream_info.mimetype or "").lower()
        extension = (stream_info.extension or "").lower()

        if extension == ".xlsx":
            return True

        if mimetype.startswith(
            "application/vnd.openxmlformats-officedocument.spreadsheetml"
        ):
            return True

        return False

    def convert(
        self,
        file_stream: BinaryIO,
        stream_info: StreamInfo,
        **kwargs: Any,
    ) -> DocumentConverterResult:
        if _xlsx_dependency_exc_info is not None:
            raise MissingDependencyException(
                MISSING_DEPENDENCY_MESSAGE.format(
                    converter=type(self).__name__,
                    extension=".xlsx",
                    feature="xlsx",
                )
            ) from _xlsx_dependency_exc_info[1].with_traceback(
                _xlsx_dependency_exc_info[2]
            )  # type: ignore[union-attr]

        # Get OCR service if available (from kwargs or instance)
        ocr_service: Optional[LLMVisionOCRService] = (
            kwargs.get("ocr_service") or self.ocr_service
        )

        if ocr_service:
            # Remove ocr_service from kwargs to avoid duplicate argument error
            kwargs_without_ocr = {k: v for k, v in kwargs.items() if k != "ocr_service"}
            return self._convert_with_ocr(
                file_stream, ocr_service, **kwargs_without_ocr
            )
        else:
            return self._convert_standard(file_stream, **kwargs)

    def _convert_standard(
        self, file_stream: BinaryIO, **kwargs: Any
    ) -> DocumentConverterResult:
        """Standard conversion without OCR."""
        file_stream.seek(0)
        sheets = pd.read_excel(file_stream, sheet_name=None, engine="openpyxl")
        md_content = ""

        for sheet_name in sheets:
            md_content += f"## {sheet_name}\n"
            html_content = sheets[sheet_name].to_html(index=False)
            md_content += (
                self._html_converter.convert_string(
                    html_content, **kwargs
                ).markdown.strip()
                + "\n\n"
            )

        return DocumentConverterResult(markdown=md_content.strip())

    def _convert_with_ocr(
        self, file_stream: BinaryIO, ocr_service: LLMVisionOCRService, **kwargs: Any
    ) -> DocumentConverterResult:
        """Convert XLSX with image OCR."""
        file_stream.seek(0)
        wb = load_workbook(file_stream)

        md_content = ""

        for sheet_name in wb.sheetnames:
            sheet = wb[sheet_name]
            md_content += f"## {sheet_name}\n\n"

            # Convert sheet data to markdown table
            file_stream.seek(0)
            try:
                df = pd.read_excel(
                    file_stream, sheet_name=sheet_name, engine="openpyxl"
                )
                html_content = df.to_html(index=False)
                md_content += (
                    self._html_converter.convert_string(
                        html_content, **kwargs
                    ).markdown.strip()
                    + "\n\n"
                )
            except Exception:
                # If pandas fails, just skip the table
                pass

            # Extract and OCR images in this sheet
            images_with_ocr = self._extract_and_ocr_sheet_images(sheet, ocr_service)

            if images_with_ocr:
                md_content += "### Images in this sheet:\n\n"
                for img_info in images_with_ocr:
                    ocr_text = img_info["ocr_text"]
                    md_content += f"*[Image OCR]\n{ocr_text}\n[End OCR]*\n\n"

        return DocumentConverterResult(markdown=md_content.strip())

    def _extract_and_ocr_sheet_images(
        self, sheet: Any, ocr_service: LLMVisionOCRService
    ) -> list[dict]:
        """
        Extract and OCR images from an Excel sheet.

        Args:
            sheet: openpyxl worksheet
            ocr_service: OCR service

        Returns:
            List of dicts with 'cell_ref' and 'ocr_text'
        """
        results = []

        try:
            # Check if sheet has images
            if hasattr(sheet, "_images"):
                for img in sheet._images:
                    try:
                        # Get image data
                        if hasattr(img, "_data"):
                            image_data = img._data()
                        elif hasattr(img, "image"):
                            # Some versions store it differently
                            image_data = img.image
                        else:
                            continue

                        # Create image stream
                        image_stream = io.BytesIO(image_data)

                        # Get cell reference
                        cell_ref = "unknown"
                        if hasattr(img, "anchor"):
                            anchor = img.anchor
                            if hasattr(anchor, "_from"):
                                from_cell = anchor._from
                                if hasattr(from_cell, "col") and hasattr(
                                    from_cell, "row"
                                ):
                                    # Convert column number to letter
                                    col_letter = self._column_number_to_letter(
                                        from_cell.col
                                    )
                                    cell_ref = f"{col_letter}{from_cell.row + 1}"

                        # Perform OCR
                        ocr_result = ocr_service.extract_text(image_stream)

                        if ocr_result.text.strip():
                            results.append(
                                {
                                    "cell_ref": cell_ref,
                                    "ocr_text": ocr_result.text.strip(),
                                    "backend": ocr_result.backend_used,
                                }
                            )

                    except Exception:
                        continue

        except Exception:
            pass

        return results

    @staticmethod
    def _column_number_to_letter(n: int) -> str:
        """Convert column number to Excel column letter (0-indexed)."""
        result = ""
        n = n + 1  # Make 1-indexed
        while n > 0:
            n -= 1
            result = chr(65 + (n % 26)) + result
            n //= 26
        return result
```

### /Users/imilev/StartUp/demo/test/markitdown/packages/markitdown-ocr/src/markitdown_ocr/_docx_converter_with_ocr.py (lines 33-189)
```
class DocxConverterWithOCR(HtmlConverter):
    """
    Enhanced DOCX Converter with OCR support for embedded images.
    Maintains document flow while extracting text from images inline.
    """

    def __init__(self, ocr_service: Optional[LLMVisionOCRService] = None):
        super().__init__()
        self._html_converter = HtmlConverter()
        self.ocr_service = ocr_service

    def accepts(
        self,
        file_stream: BinaryIO,
        stream_info: StreamInfo,
        **kwargs: Any,
    ) -> bool:
        mimetype = (stream_info.mimetype or "").lower()
        extension = (stream_info.extension or "").lower()

        if extension == ".docx":
            return True

        if mimetype.startswith(
            "application/vnd.openxmlformats-officedocument.wordprocessingml"
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
                    extension=".docx",
                    feature="docx",
                )
            ) from _dependency_exc_info[1].with_traceback(
                _dependency_exc_info[2]
            )  # type: ignore[union-attr]

        # Get OCR service if available (from kwargs or instance)
        ocr_service: Optional[LLMVisionOCRService] = (
            kwargs.get("ocr_service") or self.ocr_service
        )

        if ocr_service:
            # 1. Extract and OCR images — returns raw text per image
            file_stream.seek(0)
            image_ocr_map = self._extract_and_ocr_images(file_stream, ocr_service)

            # 2. Convert DOCX → HTML via mammoth
            file_stream.seek(0)
            pre_process_stream = pre_process_docx(file_stream)
            html_result = mammoth.convert_to_html(
                pre_process_stream, style_map=kwargs.get("style_map")
            ).value

            # 3. Replace <img> tags with plain placeholder tokens so that
            #    mammoth's HTML→markdown step never escapes our OCR markers.
            html_with_placeholders, ocr_texts = self._inject_placeholders(
                html_result, image_ocr_map
            )

            # 4. Convert HTML → markdown
            md_result = self._html_converter.convert_string(
                html_with_placeholders, **kwargs
            )
            md = md_result.markdown

            # 5. Swap placeholders for the actual OCR blocks (post-conversion
            #    so * and _ are never escaped by the markdown converter).
            for i, raw_text in enumerate(ocr_texts):
                placeholder = _PLACEHOLDER.format(i)
                ocr_block = f"*[Image OCR]\n{raw_text}\n[End OCR]*"
                md = md.replace(placeholder, ocr_block)

            return DocumentConverterResult(markdown=md)
        else:
            # Standard conversion without OCR
            style_map = kwargs.get("style_map", None)
            pre_process_stream = pre_process_docx(file_stream)
            return self._html_converter.convert_string(
                mammoth.convert_to_html(pre_process_stream, style_map=style_map).value,
                **kwargs,
            )

    def _extract_and_ocr_images(
        self, file_stream: BinaryIO, ocr_service: LLMVisionOCRService
    ) -> dict[str, str]:
        """
        Extract images from DOCX and OCR them.

        Returns:
            Dict mapping image relationship IDs to raw OCR text (no markers).
        """
        ocr_map = {}

        try:
            file_stream.seek(0)
            doc = Document(file_stream)

            for rel in doc.part.rels.values():
                if "image" in rel.target_ref.lower():
                    try:
                        image_bytes = rel.target_part.blob
                        image_stream = io.BytesIO(image_bytes)
                        ocr_result = ocr_service.extract_text(image_stream)

                        if ocr_result.text.strip():
                            # Store raw text only — markers added later
                            ocr_map[rel.rId] = ocr_result.text.strip()

                    except Exception:
                        continue

        except Exception:
            pass

        return ocr_map

    def _inject_placeholders(
        self, html: str, ocr_map: dict[str, str]
    ) -> tuple[str, list[str]]:
        """
        Replace <img> tags with numbered placeholder tokens.

        Returns:
            (html_with_placeholders, ordered list of raw OCR texts)
        """
        if not ocr_map:
            return html, []

        ocr_texts = list(ocr_map.values())
        used: list[int] = []

        def replace_img(match: re.Match) -> str:  # type: ignore[type-arg]
            for i in range(len(ocr_texts)):
                if i not in used:
                    used.append(i)
                    return f"<p>{_PLACEHOLDER.format(i)}</p>"
            return ""  # remove image if all OCR texts already used

        result = re.sub(r"<img[^>]*>", replace_img, html)

        # Any OCR texts that had no matching <img> tag go at the end
        for i in range(len(ocr_texts)):
            if i not in used:
                result += f"<p>{_PLACEHOLDER.format(i)}</p>"

        return result, ocr_texts
```


## Source Files:

- `packages/markitdown-ocr/src/markitdown_ocr/_docx_converter_with_ocr.py`
- `packages/markitdown-ocr/src/markitdown_ocr/_pptx_converter_with_ocr.py`
- `packages/markitdown-ocr/src/markitdown_ocr/_xlsx_converter_with_ocr.py`

