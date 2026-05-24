---
component_id: 5
component_name: AI & External Service Adapters
---

# AI & External Service Adapters

## Component Description

Manages integrations with external APIs for cloud document intelligence, audio transcription, and YouTube scraping.

---

## Key References:

### /Users/imilev/StartUp/demo/test/markitdown/packages/markitdown/src/markitdown/converters/_doc_intel_converter.py (lines 130-254)
```
class DocumentIntelligenceConverter(DocumentConverter):
    """Specialized DocumentConverter that uses Document Intelligence to extract text from documents."""

    def __init__(
        self,
        *,
        endpoint: str,
        api_version: str = "2024-07-31-preview",
        credential: AzureKeyCredential | TokenCredential | None = None,
        file_types: List[DocumentIntelligenceFileType] = [
            DocumentIntelligenceFileType.DOCX,
            DocumentIntelligenceFileType.PPTX,
            DocumentIntelligenceFileType.XLSX,
            DocumentIntelligenceFileType.PDF,
            DocumentIntelligenceFileType.JPEG,
            DocumentIntelligenceFileType.PNG,
            DocumentIntelligenceFileType.BMP,
            DocumentIntelligenceFileType.TIFF,
        ],
    ):
        """
        Initialize the DocumentIntelligenceConverter.

        Args:
            endpoint (str): The endpoint for the Document Intelligence service.
            api_version (str): The API version to use. Defaults to "2024-07-31-preview".
            credential (AzureKeyCredential | TokenCredential | None): The credential to use for authentication.
            file_types (List[DocumentIntelligenceFileType]): The file types to accept. Defaults to all supported file types.
        """

        super().__init__()
        self._file_types = file_types

        # Raise an error if the dependencies are not available.
        # This is different than other converters since this one isn't even instantiated
        # unless explicitly requested.
        if _dependency_exc_info is not None:
            raise MissingDependencyException(
                "DocumentIntelligenceConverter requires the optional dependency [az-doc-intel] (or [all]) to be installed. E.g., `pip install markitdown[az-doc-intel]`"
            ) from _dependency_exc_info[
                1
            ].with_traceback(  # type: ignore[union-attr]
                _dependency_exc_info[2]
            )

        if credential is None:
            if os.environ.get("AZURE_API_KEY") is None:
                credential = DefaultAzureCredential()
            else:
                credential = AzureKeyCredential(os.environ["AZURE_API_KEY"])

        self.endpoint = endpoint
        self.api_version = api_version
        self.doc_intel_client = DocumentIntelligenceClient(
            endpoint=self.endpoint,
            api_version=self.api_version,
            credential=credential,
        )

    def accepts(
        self,
        file_stream: BinaryIO,
        stream_info: StreamInfo,
        **kwargs: Any,  # Options to pass to the converter
    ) -> bool:
        mimetype = (stream_info.mimetype or "").lower()
        extension = (stream_info.extension or "").lower()

        if extension in _get_file_extensions(self._file_types):
            return True

        for prefix in _get_mime_type_prefixes(self._file_types):
            if mimetype.startswith(prefix):
                return True

        return False

    def _analysis_features(self, stream_info: StreamInfo) -> List[str]:
        """
        Helper needed to determine which analysis features to use.
        Certain document analysis features are not availiable for
        office filetypes (.xlsx, .pptx, .html, .docx)
        """
        mimetype = (stream_info.mimetype or "").lower()
        extension = (stream_info.extension or "").lower()

        # Types that don't support ocr
        no_ocr_types = [
            DocumentIntelligenceFileType.DOCX,
            DocumentIntelligenceFileType.PPTX,
            DocumentIntelligenceFileType.XLSX,
            DocumentIntelligenceFileType.HTML,
        ]

        if extension in _get_file_extensions(no_ocr_types):
            return []

        for prefix in _get_mime_type_prefixes(no_ocr_types):
            if mimetype.startswith(prefix):
                return []

        return [
            DocumentAnalysisFeature.FORMULAS,  # enable formula extraction
            DocumentAnalysisFeature.OCR_HIGH_RESOLUTION,  # enable high resolution OCR
            DocumentAnalysisFeature.STYLE_FONT,  # enable font style extraction
        ]

    def convert(
        self,
        file_stream: BinaryIO,
        stream_info: StreamInfo,
        **kwargs: Any,  # Options to pass to the converter
    ) -> DocumentConverterResult:
        # Extract the text using Azure Document Intelligence
        poller = self.doc_intel_client.begin_analyze_document(
            model_id="prebuilt-layout",
            body=AnalyzeDocumentRequest(bytes_source=file_stream.read()),
            features=self._analysis_features(stream_info),
            output_content_format=CONTENT_FORMAT,  # TODO: replace with "ContentFormat.MARKDOWN" when the bug is fixed
        )
        result: AnalyzeResult = poller.result()

        # remove comments from the markdown content generated by Doc Intelligence and append to markdown string
        markdown_text = re.sub(r"<!--.*?-->", "", result.content, flags=re.DOTALL)
        return DocumentConverterResult(markdown=markdown_text)
```

### /Users/imilev/StartUp/demo/test/markitdown/packages/markitdown/src/markitdown/converters/_youtube_converter.py (lines 37-238)
```
class YouTubeConverter(DocumentConverter):
    """Handle YouTube specially, focusing on the video title, description, and transcript."""

    def accepts(
        self,
        file_stream: BinaryIO,
        stream_info: StreamInfo,
        **kwargs: Any,  # Options to pass to the converter
    ) -> bool:
        """
        Make sure we're dealing with HTML content *from* YouTube.
        """
        url = stream_info.url or ""
        mimetype = (stream_info.mimetype or "").lower()
        extension = (stream_info.extension or "").lower()

        url = unquote(url)
        url = url.replace(r"\?", "?").replace(r"\=", "=")

        if not url.startswith("https://www.youtube.com/watch?"):
            # Not a YouTube URL
            return False

        if extension in ACCEPTED_FILE_EXTENSIONS:
            return True

        for prefix in ACCEPTED_MIME_TYPE_PREFIXES:
            if mimetype.startswith(prefix):
                return True

        # Not HTML content
        return False

    def convert(
        self,
        file_stream: BinaryIO,
        stream_info: StreamInfo,
        **kwargs: Any,  # Options to pass to the converter
    ) -> DocumentConverterResult:
        # Parse the stream
        encoding = "utf-8" if stream_info.charset is None else stream_info.charset
        soup = bs4.BeautifulSoup(file_stream, "html.parser", from_encoding=encoding)

        # Read the meta tags
        metadata: Dict[str, str] = {}

        if soup.title and soup.title.string:
            metadata["title"] = soup.title.string

        for meta in soup(["meta"]):
            if not isinstance(meta, bs4.Tag):
                continue

            for a in meta.attrs:
                if a in ["itemprop", "property", "name"]:
                    key = str(meta.get(a, ""))
                    content = str(meta.get("content", ""))
                    if key and content:  # Only add non-empty content
                        metadata[key] = content
                    break

        # Try reading the description
        try:
            for script in soup(["script"]):
                if not isinstance(script, bs4.Tag):
                    continue
                if not script.string:  # Skip empty scripts
                    continue
                content = script.string
                if "ytInitialData" in content:
                    match = re.search(r"var ytInitialData = ({.*?});", content)
                    if match:
                        data = json.loads(match.group(1))
                        attrdesc = self._findKey(data, "attributedDescriptionBodyText")
                        if attrdesc and isinstance(attrdesc, dict):
                            metadata["description"] = str(attrdesc.get("content", ""))
                    break
        except Exception as e:
            print(f"Error extracting description: {e}")
            pass

        # Start preparing the page
        webpage_text = "# YouTube\n"

        title = self._get(metadata, ["title", "og:title", "name"])  # type: ignore
        assert isinstance(title, str)

        if title:
            webpage_text += f"\n## {title}\n"

        stats = ""
        views = self._get(metadata, ["interactionCount"])  # type: ignore
        if views:
            stats += f"- **Views:** {views}\n"

        keywords = self._get(metadata, ["keywords"])  # type: ignore
        if keywords:
            stats += f"- **Keywords:** {keywords}\n"

        runtime = self._get(metadata, ["duration"])  # type: ignore
        if runtime:
            stats += f"- **Runtime:** {runtime}\n"

        if len(stats) > 0:
            webpage_text += f"\n### Video Metadata\n{stats}\n"

        description = self._get(metadata, ["description", "og:description"])  # type: ignore
        if description:
            webpage_text += f"\n### Description\n{description}\n"

        if IS_YOUTUBE_TRANSCRIPT_CAPABLE:
            ytt_api = YouTubeTranscriptApi()
            transcript_text = ""
            parsed_url = urlparse(stream_info.url)  # type: ignore
            params = parse_qs(parsed_url.query)  # type: ignore
            if "v" in params and params["v"][0]:
                video_id = str(params["v"][0])
                transcript_list = ytt_api.list(video_id)
                languages = ["en"]
                for transcript in transcript_list:
                    languages.append(transcript.language_code)
                    break
                try:
                    youtube_transcript_languages = kwargs.get(
                        "youtube_transcript_languages", languages
                    )
                    # Retry the transcript fetching operation
                    transcript = self._retry_operation(
                        lambda: ytt_api.fetch(
                            video_id, languages=youtube_transcript_languages
                        ),
                        retries=3,  # Retry 3 times
                        delay=2,  # 2 seconds delay between retries
                    )

                    if transcript:
                        transcript_text = " ".join(
                            [part.text for part in transcript]
                        )  # type: ignore
                except Exception as e:
                    # No transcript available
                    if len(languages) == 1:
                        print(f"Error fetching transcript: {e}")
                    else:
                        # Translate transcript into first kwarg
                        transcript = (
                            transcript_list.find_transcript(languages)
                            .translate(youtube_transcript_languages[0])
                            .fetch()
                        )
                        transcript_text = " ".join([part.text for part in transcript])
            if transcript_text:
                webpage_text += f"\n### Transcript\n{transcript_text}\n"

        title = title if title else (soup.title.string if soup.title else "")
        assert isinstance(title, str)

        return DocumentConverterResult(
            markdown=webpage_text,
            title=title,
        )

    def _get(
        self,
        metadata: Dict[str, str],
        keys: List[str],
        default: Union[str, None] = None,
    ) -> Union[str, None]:
        """Get first non-empty value from metadata matching given keys."""
        for k in keys:
            if k in metadata:
                return metadata[k]
        return default

    def _findKey(self, json: Any, key: str) -> Union[str, None]:  # TODO: Fix json type
        """Recursively search for a key in nested dictionary/list structures."""
        if isinstance(json, list):
            for elm in json:
                ret = self._findKey(elm, key)
                if ret is not None:
                    return ret
        elif isinstance(json, dict):
            for k, v in json.items():
                if k == key:
                    return json[k]
                if result := self._findKey(v, key):
                    return result
        return None

    def _retry_operation(self, operation, retries=3, delay=2):
        """Retries the operation if it fails."""
        attempt = 0
        while attempt < retries:
            try:
                return operation()  # Attempt the operation
            except Exception as e:
                print(f"Attempt {attempt + 1} failed: {e}")
                if attempt < retries - 1:
                    time.sleep(delay)  # Wait before retrying
                attempt += 1
        # If all attempts fail, raise the last exception
        raise Exception(f"Operation failed after {retries} attempts.")
```

### /Users/imilev/StartUp/demo/test/markitdown/packages/markitdown/src/markitdown/converters/_transcribe_audio.py (lines 23-49)
```
def transcribe_audio(file_stream: BinaryIO, *, audio_format: str = "wav") -> str:
    # Check for installed dependencies
    if _dependency_exc_info is not None:
        raise MissingDependencyException(
            "Speech transcription requires installing MarkItdown with the [audio-transcription] optional dependencies. E.g., `pip install markitdown[audio-transcription]` or `pip install markitdown[all]`"
        ) from _dependency_exc_info[
            1
        ].with_traceback(  # type: ignore[union-attr]
            _dependency_exc_info[2]
        )

    if audio_format in ["wav", "aiff", "flac"]:
        audio_source = file_stream
    elif audio_format in ["mp3", "mp4"]:
        audio_segment = pydub.AudioSegment.from_file(file_stream, format=audio_format)

        audio_source = io.BytesIO()
        audio_segment.export(audio_source, format="wav")
        audio_source.seek(0)
    else:
        raise ValueError(f"Unsupported audio format: {audio_format}")

    recognizer = sr.Recognizer()
    with sr.AudioFile(audio_source) as source:
        audio = recognizer.record(source)
        transcript = recognizer.recognize_google(audio).strip()
        return "[No speech detected]" if transcript == "" else transcript
```


## Source Files:

- `packages/markitdown/src/markitdown/converters/_audio_converter.py`
- `packages/markitdown/src/markitdown/converters/_doc_intel_converter.py`
- `packages/markitdown/src/markitdown/converters/_exiftool.py`
- `packages/markitdown/src/markitdown/converters/_image_converter.py`
- `packages/markitdown/src/markitdown/converters/_transcribe_audio.py`
- `packages/markitdown/src/markitdown/converters/_youtube_converter.py`

