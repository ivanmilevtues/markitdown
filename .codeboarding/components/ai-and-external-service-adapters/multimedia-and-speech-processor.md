---
component_id: 5.2
component_name: Multimedia & Speech Processor
---

# Multimedia & Speech Processor

## Component Description

Manages the conversion of audio and visual media into text. It utilizes speech-to-text models (like Whisper) for audio transcription and Large Multimodal Models (LMMs) for generating descriptive captions for images and video frames.

---

## Key References:

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
- `packages/markitdown/src/markitdown/converters/_transcribe_audio.py`

