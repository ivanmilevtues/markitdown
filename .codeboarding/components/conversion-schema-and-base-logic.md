---
component_id: 2
component_name: Conversion Schema & Base Logic
---

# Conversion Schema & Base Logic

## Component Description

Defines fundamental contracts and data structures, ensuring standardized output across all converter types.

---

## Key References:

### /Users/imilev/StartUp/demo/test/markitdown/packages/markitdown/src/markitdown/_base_converter.py (lines 42-105)
```
class DocumentConverter:
    """Abstract superclass of all DocumentConverters."""

    def accepts(
        self,
        file_stream: BinaryIO,
        stream_info: StreamInfo,
        **kwargs: Any,  # Options to pass to the converter
    ) -> bool:
        """
        Return a quick determination on if the converter should attempt converting the document.
        This is primarily based `stream_info` (typically, `stream_info.mimetype`, `stream_info.extension`).
        In cases where the data is retrieved via HTTP, the `steam_info.url` might also be referenced to
        make a determination (e.g., special converters for Wikipedia, YouTube etc).
        Finally, it is conceivable that the `stream_info.filename` might be used to in cases
        where the filename is well-known (e.g., `Dockerfile`, `Makefile`, etc)

        NOTE: The method signature is designed to match that of the convert() method. This provides some
        assurance that, if accepts() returns True, the convert() method will also be able to handle the document.

        IMPORTANT: In rare cases, (e.g., OutlookMsgConverter) we need to read more from the stream to make a final
        determination. Read operations inevitably advances the position in file_stream. In these case, the position
        MUST be reset it MUST be reset before returning. This is because the convert() method may be called immediately
        after accepts(), and will expect the file_stream to be at the original position.

        E.g.,
        cur_pos = file_stream.tell() # Save the current position
        data = file_stream.read(100) # ... peek at the first 100 bytes, etc.
        file_stream.seek(cur_pos)    # Reset the position to the original position

        Parameters:
        - file_stream: The file-like object to convert. Must support seek(), tell(), and read() methods.
        - stream_info: The StreamInfo object containing metadata about the file (mimetype, extension, charset, set)
        - kwargs: Additional keyword arguments for the converter.

        Returns:
        - bool: True if the converter can handle the document, False otherwise.
        """
        raise NotImplementedError(
            f"The subclass, {type(self).__name__}, must implement the accepts() method to determine if they can handle the document."
        )

    def convert(
        self,
        file_stream: BinaryIO,
        stream_info: StreamInfo,
        **kwargs: Any,  # Options to pass to the converter
    ) -> DocumentConverterResult:
        """
        Convert a document to Markdown text.

        Parameters:
        - file_stream: The file-like object to convert. Must support seek(), tell(), and read() methods.
        - stream_info: The StreamInfo object containing metadata about the file (mimetype, extension, charset, set)
        - kwargs: Additional keyword arguments for the converter.

        Returns:
        - DocumentConverterResult: The result of the conversion, which includes the title and markdown content.

        Raises:
        - FileConversionException: If the mimetype is recognized, but the conversion fails for some other reason.
        - MissingDependencyException: If the converter requires a dependency that is not installed.
        """
        raise NotImplementedError("Subclasses must implement this method")
```

### /Users/imilev/StartUp/demo/test/markitdown/packages/markitdown/src/markitdown/_base_converter.py (lines 5-39)
```
class DocumentConverterResult:
    """The result of converting a document to Markdown."""

    def __init__(
        self,
        markdown: str,
        *,
        title: Optional[str] = None,
    ):
        """
        Initialize the DocumentConverterResult.

        The only required parameter is the converted Markdown text.
        The title, and any other metadata that may be added in the future, are optional.

        Parameters:
        - markdown: The converted Markdown text.
        - title: Optional title of the document.
        """
        self.markdown = markdown
        self.title = title

    @property
    def text_content(self) -> str:
        """Soft-deprecated alias for `markdown`. New code should migrate to using `markdown` or __str__."""
        return self.markdown

    @text_content.setter
    def text_content(self, markdown: str):
        """Soft-deprecated alias for `markdown`. New code should migrate to using `markdown` or __str__."""
        self.markdown = markdown

    def __str__(self) -> str:
        """Return the converted Markdown text."""
        return self.markdown
```

### /Users/imilev/StartUp/demo/test/markitdown/packages/markitdown/src/markitdown/converters/_rss_converter.py (lines 29-192)
```
class RssConverter(DocumentConverter):
    """Convert RSS / Atom type to markdown"""

    def __init__(self):
        super().__init__()
        self._kwargs = {}

    def accepts(
        self,
        file_stream: BinaryIO,
        stream_info: StreamInfo,
        **kwargs: Any,  # Options to pass to the converter
    ) -> bool:
        mimetype = (stream_info.mimetype or "").lower()
        extension = (stream_info.extension or "").lower()

        # Check for precise mimetypes and file extensions
        if extension in PRECISE_FILE_EXTENSIONS:
            return True

        for prefix in PRECISE_MIME_TYPE_PREFIXES:
            if mimetype.startswith(prefix):
                return True

        # Check for precise mimetypes and file extensions
        if extension in CANDIDATE_FILE_EXTENSIONS:
            return self._check_xml(file_stream)

        for prefix in CANDIDATE_MIME_TYPE_PREFIXES:
            if mimetype.startswith(prefix):
                return self._check_xml(file_stream)

        return False

    def _check_xml(self, file_stream: BinaryIO) -> bool:
        cur_pos = file_stream.tell()
        try:
            doc = minidom.parse(file_stream)
            return self._feed_type(doc) is not None
        except BaseException as _:
            pass
        finally:
            file_stream.seek(cur_pos)
        return False

    def _feed_type(self, doc: Any) -> str | None:
        if doc.getElementsByTagName("rss"):
            return "rss"
        elif doc.getElementsByTagName("feed"):
            root = doc.getElementsByTagName("feed")[0]
            if root.getElementsByTagName("entry"):
                # An Atom feed must have a root element of <feed> and at least one <entry>
                return "atom"
        return None

    def convert(
        self,
        file_stream: BinaryIO,
        stream_info: StreamInfo,
        **kwargs: Any,  # Options to pass to the converter
    ) -> DocumentConverterResult:
        self._kwargs = kwargs
        doc = minidom.parse(file_stream)
        feed_type = self._feed_type(doc)

        if feed_type == "rss":
            return self._parse_rss_type(doc)
        elif feed_type == "atom":
            return self._parse_atom_type(doc)
        else:
            raise ValueError("Unknown feed type")

    def _parse_atom_type(self, doc: Document) -> DocumentConverterResult:
        """Parse the type of an Atom feed.

        Returns None if the feed type is not recognized or something goes wrong.
        """
        root = doc.getElementsByTagName("feed")[0]
        title = self._get_data_by_tag_name(root, "title")
        subtitle = self._get_data_by_tag_name(root, "subtitle")
        entries = root.getElementsByTagName("entry")
        md_text = f"# {title}\n"
        if subtitle:
            md_text += f"{subtitle}\n"
        for entry in entries:
            entry_title = self._get_data_by_tag_name(entry, "title")
            entry_summary = self._get_data_by_tag_name(entry, "summary")
            entry_updated = self._get_data_by_tag_name(entry, "updated")
            entry_content = self._get_data_by_tag_name(entry, "content")

            if entry_title:
                md_text += f"\n## {entry_title}\n"
            if entry_updated:
                md_text += f"Updated on: {entry_updated}\n"
            if entry_summary:
                md_text += self._parse_content(entry_summary)
            if entry_content:
                md_text += self._parse_content(entry_content)

        return DocumentConverterResult(
            markdown=md_text,
            title=title,
        )

    def _parse_rss_type(self, doc: Document) -> DocumentConverterResult:
        """Parse the type of an RSS feed.

        Returns None if the feed type is not recognized or something goes wrong.
        """
        root = doc.getElementsByTagName("rss")[0]
        channel_list = root.getElementsByTagName("channel")
        if not channel_list:
            raise ValueError("No channel found in RSS feed")
        channel = channel_list[0]
        channel_title = self._get_data_by_tag_name(channel, "title")
        channel_description = self._get_data_by_tag_name(channel, "description")
        items = channel.getElementsByTagName("item")
        if channel_title:
            md_text = f"# {channel_title}\n"
        if channel_description:
            md_text += f"{channel_description}\n"
        for item in items:
            title = self._get_data_by_tag_name(item, "title")
            description = self._get_data_by_tag_name(item, "description")
            pubDate = self._get_data_by_tag_name(item, "pubDate")
            content = self._get_data_by_tag_name(item, "content:encoded")

            if title:
                md_text += f"\n## {title}\n"
            if pubDate:
                md_text += f"Published on: {pubDate}\n"
            if description:
                md_text += self._parse_content(description)
            if content:
                md_text += self._parse_content(content)

        return DocumentConverterResult(
            markdown=md_text,
            title=channel_title,
        )

    def _parse_content(self, content: str) -> str:
        """Parse the content of an RSS feed item"""
        try:
            # using bs4 because many RSS feeds have HTML-styled content
            soup = BeautifulSoup(content, "html.parser")
            return _CustomMarkdownify(**self._kwargs).convert_soup(soup)
        except BaseException as _:
            return content

    def _get_data_by_tag_name(
        self, element: Element, tag_name: str
    ) -> Union[str, None]:
        """Get data from first child element with the given tag name.
        Returns None when no such element is found.
        """
        nodes = element.getElementsByTagName(tag_name)
        if not nodes:
            return None
        fc = nodes[0].firstChild
        if fc:
            if hasattr(fc, "data"):
                return fc.data
        return None
```


## Source Files:

- `packages/markitdown/src/markitdown/_base_converter.py`
- `packages/markitdown/src/markitdown/_exceptions.py`
- `packages/markitdown/src/markitdown/converters/_bing_serp_converter.py`
- `packages/markitdown/src/markitdown/converters/_csv_converter.py`
- `packages/markitdown/src/markitdown/converters/_epub_converter.py`
- `packages/markitdown/src/markitdown/converters/_html_converter.py`
- `packages/markitdown/src/markitdown/converters/_ipynb_converter.py`
- `packages/markitdown/src/markitdown/converters/_markdownify.py`
- `packages/markitdown/src/markitdown/converters/_plain_text_converter.py`
- `packages/markitdown/src/markitdown/converters/_rss_converter.py`
- `packages/markitdown/src/markitdown/converters/_wikipedia_converter.py`
- `packages/markitdown/src/markitdown/converters/_yaml_converter.py`

