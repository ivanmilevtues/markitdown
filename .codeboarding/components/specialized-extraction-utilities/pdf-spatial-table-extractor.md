---
component_id: 4.2
component_name: PDF Spatial Table Extractor
---

# PDF Spatial Table Extractor

## Component Description

Reconstructs logical table structures from raw PDF text fragments. It uses spatial heuristics to map text coordinates into a grid-based representation and subsequently serializes that grid into a standard Markdown table format.

---

## Key References:

### /Users/imilev/StartUp/demo/test/markitdown/packages/markitdown/src/markitdown/converters/_pdf_converter.py (lines 592-622)
```
def extract_tables_from_pdf(pdf_bytes: io.IOBase) -> list[list[list[str]]]:
    """Extract all tables from a PDF as a list of row/column grids.

    Each table is represented as a list of rows, where each row is a list
    of cell strings.  This is useful when the caller needs structured data
    rather than Markdown output.
    """
    import pdfminer.high_level
    import pdfminer.layout as pdfminer_layout

    tables: list[list[list[str]]] = []
    for page_layout in pdfminer.high_level.extract_pages(pdf_bytes):
        page_words: list[dict] = []
        for element in page_layout:
            if isinstance(element, pdfminer_layout.LTTextContainer):
                for line in element:
                    if isinstance(line, pdfminer_layout.LTTextLineHorizontal):
                        text = line.get_text().strip()
                        if text:
                            page_words.append({
                                "text": text,
                                "x0": line.x0,
                                "x1": line.x1,
                                "top": line.y0,
                                "bottom": line.y1,
                            })
        if page_words:
            grid = _words_to_grid(page_words)
            if grid:
                tables.append(grid)
    return tables
```

### /Users/imilev/StartUp/demo/test/markitdown/packages/markitdown/src/markitdown/converters/_pdf_converter.py (lines 625-637)
```
def _words_to_grid(words: list[dict]) -> list[list[str]]:
    """Heuristic grouping of words into a row/column grid."""
    if not words:
        return []
    rows: dict[float, list[dict]] = {}
    for w in words:
        y_key = round(w["top"], 1)
        rows.setdefault(y_key, []).append(w)
    grid: list[list[str]] = []
    for _, row_words in sorted(rows.items()):
        row_words.sort(key=lambda w: w["x0"])
        grid.append([w["text"] for w in row_words])
    return grid
```

### /Users/imilev/StartUp/demo/test/markitdown/packages/markitdown/src/markitdown/converters/_pdf_converter.py (lines 78-117)
```
def _to_markdown_table(table: list[list[str]], include_separator: bool = True) -> str:
    """Convert a 2D list (rows/columns) into a nicely aligned Markdown table.

    Args:
        table: 2D list of cell values
        include_separator: If True, include header separator row (standard markdown).
                          If False, output simple pipe-separated rows.
    """
    if not table:
        return ""

    # Normalize None → ""
    table = [[cell if cell is not None else "" for cell in row] for row in table]

    # Filter out empty rows
    table = [row for row in table if any(cell.strip() for cell in row)]

    if not table:
        return ""

    # Column widths
    col_widths = [max(len(str(cell)) for cell in col) for col in zip(*table)]

    def fmt_row(row: list[str]) -> str:
        return (
            "|"
            + "|".join(str(cell).ljust(width) for cell, width in zip(row, col_widths))
            + "|"
        )

    if include_separator:
        header, *rows = table
        md = [fmt_row(header)]
        md.append("|" + "|".join("-" * w for w in col_widths) + "|")
        for row in rows:
            md.append(fmt_row(row))
    else:
        md = [fmt_row(row) for row in table]

    return "\n".join(md)
```


## Source Files:

- `packages/markitdown/src/markitdown/converters/_pdf_converter.py`

