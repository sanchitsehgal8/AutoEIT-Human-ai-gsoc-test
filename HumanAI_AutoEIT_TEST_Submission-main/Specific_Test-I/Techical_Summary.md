

﻿#              Technical Summary

## Problem Statement

The goal of this component is to build a robust, fully reproducible pipeline for transcribing Spanish Elicited Imitation Task (EIT) recordings. The transcriptions must follow the participant’s speech *as it is produced*, including disfluencies, false starts, and code-switching, rather than “cleaning up” or correcting the learner.

## Transformer-Based ASR Model

For automatic speech recognition I chose OpenAI’s **Whisper** model (`medium` size), accessed through the `faster-whisper` inference library. Although this model is relatively heavy, it provides several properties that are especially important for EIT data:

* **Multilingual robustness:** The `medium` model is trained on a broad multilingual corpus and handles Spanish EIT recordings with strong word error rates, correctly dealing with accent marks (e.g., *¿Cómo?*) as well as occasional English intrusions.
* **Integrated Voice Activity Detection (VAD):** Whisper’s VAD support (`min_silence_duration_ms=1500`) is well suited to EIT, where long silent gaps (4–8 s) appear between sentences while the learner listens. This helps prevent the model from hallucinating content during these pauses.
* **Language locking (`language="es"`):** For short Spanish utterances it is easy for a multilingual model to drift into Portuguese or Catalan. Explicitly locking the decoder to Spanish removes this failure mode.

Taken together, these choices move the system away from a generic ASR setup toward a **task-aware, linguistically informed transcription pipeline** tuned for the specific structure of EIT recordings.

## Multi-Stage Pipeline Methodology

Inspecting the EIT protocol made it clear that Whisper’s raw output alone cannot guarantee a clean, slot-by-slot transcription. Whisper segments audio based on acoustic cues, whereas the task requires a neatly aligned grid of 30 stimulus–response slots. My aim was to enforce that structure in a principled, largely automatic way.

To do that, I implemented a multi-stage processing pipeline:

1. **Preamble trimming (pre-processing):** Using `pydub`, I programmatically remove the 12‑minute administrative introduction from participant `038012-2A` before running ASR. This ensures that only the relevant stimulus–response region is transcribed.
2. **Silence-gap utterance chunking (segmentation):** A custom `chunk_by_silence()` function re-combines Whisper’s raw segments into sentence-level utterances using a 2.5‑second inter-segment gap threshold. Short hesitation pauses (<2 s) stay inside a sentence; the longer EIT pauses (4–8 s) mark new slots.
3. **Slot alignment (structure enforcement):** An `align_to_slots()` routine then maps these chunks onto the fixed set of 30 sentence slots. If there are too few chunks, the remaining slots are filled with `[no response]`; if there are too many, the extras are flagged for manual inspection.
4. **Disfluency-preserving post-processing:** Finally, a `postprocess()` function standardises Whisper’s noise tokens (e.g., `[inaudible]`) into the protocol’s `[XXX]` marker, while deliberately *keeping* false starts (trailing hyphens), ellipses, and learner errors.

These stages—acoustic pre-processing, VAD-aware segmentation, structural alignment, and protocol-compliant post-processing—together reproduce the logic of careful manual EIT transcription, but in a reproducible, scripted way.

## Implementation Details of Pipeline Stages

**1. Pre-processing (preamble trimming)**

* **Tool:** I use `pydub` to load each MP3 and cut it at millisecond precision.  
* **Logic:** A small `TRIM_START_MS` dictionary stores the per-participant trim offsets. Only `038012-2A` needs trimming (12 minutes = 720,000 ms).  
* **Goal:** Feed Whisper only the actual EIT portion, so its context window and confidence budget are not wasted on administrative chatter.

**2. Whisper ASR (automatic speech recognition)**

* **Tool:** `faster-whisper` (`WhisperModel`, `medium`), with GPU used when available and CPU with `int8` quantisation as a fallback.  
* **Parameters:** `beam_size=5` for stable best-path decoding, `temperature=0` for reproducibility, `word_timestamps=True` for later alignment, and `condition_on_previous_text=False` to keep errors from propagating across sentence boundaries.  
* **Initial prompt:** A short bilingual, EIT-focused prompt nudges the model toward the right Spanish vocabulary and explicitly requests that disfluencies be preserved, reducing the tendency to “polish” the learner’s speech.

**3. Silence-gap chunking and slot alignment**

* **Chunking threshold:** I use a `gap_threshold=2.5` seconds between segments to decide when a new sentence begins. Gaps shorter than this are treated as internal hesitations; longer gaps mark new utterances.  
* **Confidence tracking:** For each merged chunk, I keep the lowest `avg_logprob` and highest `no_speech_prob` across all constituent segments, giving a conservative confidence estimate per utterance.  
* **Alignment safeguards:** If the number of chunks exceeds 30, the extra ones are truncated with a clear `WARNING`; if fewer than 30 appear, the missing positions are padded with `[no response]`. Both scenarios are highlighted so a human can double-check if needed.

**4. Post-processing (protocol compliance)**

* **Noise tokens:** Any bracketed tokens Whisper emits (e.g., `[music]`, `[applause]`) are normalised to `[XXX]` via a simple regular expression, aligning with the official EIT unintelligibility marker.  
* **Disfluency preservation:** The post-processing step intentionally avoids stripping trailing hyphens, ellipses, or unconventional lexical choices, so that the transcriptions remain faithful to the phonological and fluency characteristics of the learner’s speech.  
* **Minimal normalisation:** Apart from light whitespace clean-up (collapsing multiple spaces), spelling, grammar, and accentuation are left exactly as produced by Whisper.

## Final Pipeline Summary

| Stage | Tool / method | Purpose |
| ----- | ------------- | ------- |
| **Preamble trim** | `pydub` millisecond slicing | Removes non-EIT audio before ASR to avoid wasting model capacity. |
| **Waveform inspection** | `librosa` + `matplotlib` | Visual QA of the speech–silence pattern and verification of trim points. |
| **ASR** | `faster-whisper` (`medium`, VAD on) | Produces time-aligned text segments with confidence scores. |
| **Silence-gap chunking** | Custom `chunk_by_silence()` (2.5 s gap) | Groups segments into sentence-level utterances aligned to the EIT structure. |
| **Slot alignment** | Custom `align_to_slots()` (30 slots) | Enforces the fixed 30-sentence grid and flags under-/over-runs. |
| **Post-processing** | `postprocess()` with regex + protocol rules | Normalises noise tokens to `[XXX]` and preserves all learner disfluencies and errors. |
| **Excel output** | `openpyxl` | Writes the final transcriptions into Column C of each participant’s sheet in the submission workbook. |
