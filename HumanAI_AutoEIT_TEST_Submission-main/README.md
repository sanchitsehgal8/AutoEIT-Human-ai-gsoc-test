# GSoC AutoEIT — Test Submission

Author: **Sanchit Sehgal**  
**Project:** AutoEIT — Automated Scoring of Elicited Imitation Tasks  
**Contact:** human-ai@cern.ch  
**Email subject:** *Evaluation Test: AutoEIT*

---

## Overview

This repository contains my complete solutions for both evaluation tasks for the **AutoEIT GSoC project**. The broader project aims to build an end-to-end system that can (1) transcribe learner audio into text and (2) assign linguistically principled scores to each response. The goal is to ease or partially replace the very time-consuming manual grading of Spanish Elicited Imitation Tasks (EIT).

I implemented **both Test I and Test II**, so this submission covers the full pipeline: from raw speech recordings all the way to final, rubric-based scores.

---

## Repository Structure

```
gsoc_AutoEIT_test_submission/
│
├── Specific_Test-I/           ← Test I: Audio-to-text transcription
│   ├── Techical_Summary.md    ← My technical write-up for Test I
│   ├── data/                  ← 4 participant MP3s + input Excel template
│   ├── notebook/              ← Colab notebook + exported PDF
│   └── output/                ← Final transcription Excel file
│
└── Specific_Test-II/          ← Test II: Automated scoring system
    ├── Technical_Summary.md   ← My technical write-up for Test II
    ├── data/                  ← Input Excel with transcriptions
    ├── notebook/              ← Colab notebook + exported PDF
    └── output/                ← Final scores Excel file
```

---

## Test I — Audio File Transcription

> **Task:** Produce faithful transcriptions for 4 sample participant audio files (each with 30 sentences), explicitly keeping disfluencies and *not* “correcting” learner errors.

### Approach

For transcription I used OpenAI’s **Whisper** model (`medium` size) via the `faster-whisper` library. On top of the raw ASR output, I built a structured, multi-step pipeline:

| Stage | Tool | Purpose |
|---|---|---|
| Preamble trim | `pydub` | Cuts away the 12‑minute admin preamble for participant `038012-2A` |
| Waveform QA | `librosa` + `matplotlib` | Visually checks the EIT pattern of speech bursts and silences |
| ASR | `faster-whisper` (VAD on) | Turns audio into time-stamped text segments |
| Silence-gap chunking | Custom `chunk_by_silence()` | Groups segments into sentence-level utterances using a 2.5 s gap threshold |
| Slot alignment | Custom `align_to_slots()` | Forces alignment to exactly 30 sentence slots; flags under-/over-runs |
| Post-processing | `postprocess()` | Normalises noise labels to `[XXX]` while keeping all disfluencies |
| Excel output | `openpyxl` | Fills the transcription columns in each participant’s sheet |

Some key settings that make the system more robust:

- **Language lock** with `language="es"` so Whisper does not drift toward Portuguese/Catalan.  
- `condition_on_previous_text=False` so errors do not snowball across sentence boundaries.  
- `temperature=0` to make runs deterministic and easy to reproduce.

### Outputs

- 📓 **Notebook:** [Specific_Test-I/notebook/AutoEIT_Test1_Transcription.ipynb](Specific_Test-I/notebook/AutoEIT_Test1_Transcription.ipynb)  
- 📄 **PDF export:** [Specific_Test-I/notebook/AutoEIT_Test1_Transcription.ipynb - Colab.pdf](<Specific_Test-I/notebook/AutoEIT_Test1_Transcription.ipynb - Colab.pdf>)  
- 📊 **Results:** [Specific_Test-I/output/AutoEIT_Test1_Transcriptions_Completed.xlsx](Specific_Test-I/output/AutoEIT_Test1_Transcriptions_Completed.xlsx)  
- 📝 **Technical summary:** [Specific_Test-I/Techical_Summary.md](Specific_Test-I/Techical_Summary.md)

---

## Test II — Automated Scoring System ⭐

> **Task:** Build a reproducible scoring script that applies the meaning-based Ortega (2000) rubric to transcribed EIT responses, outputting sentence-level scores from 0–4 for every utterance.

This is the part of the project I spent the most time refining, and also the one I enjoyed the most. To score automatically in a linguistically honest way, the system needs to approximate *understanding* of meaning rather than just counting overlapping words.

### Core Design Philosophy

No single metric can do justice to the rubric. A purely string-based metric would unfairly punish good paraphrases, while a purely semantic metric would ignore exact reproduction. I therefore designed a **three-part hybrid** that mirrors how a human rater reads a response:

1. **Structural alignment** — *How closely does the response match the original string?*  
2. **Semantic preservation** — *Is the intended meaning still fully there, even with different wording?*  
3. **Core idea coverage** — *Which key “idea units” from the stimulus actually show up?*

### Pipeline Architecture

```
Stimulus Sentence ──┐
                    ├─► edit_distance_ratio()      ─► Score 4 (≥ 0.95)
Learner Response ───┤
                    ├─► semantic_similarity()       ─► Score 3 (sim > 0.75 AND cov > 0.70)
                    │        ↑
                    │   Sentence Transformers        ─► Score 2 (sim > 0.50 AND cov ≥ 0.45)
                    │   (multilingual MiniLM-L12)
                    │
                    └─► content_words() / coverage() ─► Score 1 (cov ≥ 0.20 OR sim > 0.25)
                              ↑                         Score 0 (otherwise)
                    RoBERTa lemmatization
                    (es_dep_news_trf)
```

### Implementation Details

**1. Transformer-based lemmatization (`es_dep_news_trf`)**

Instead of relying on simple stemming, I use spaCy’s RoBERTa-powered `es_dep_news_trf` model for context-sensitive lemmatization and POS tagging. This brings together morphological variants (for example, *fuman* → *fumar*, *corriendo* → *correr*) before computing coverage, so the system does not penalise a learner who clearly has the idea but uses a different inflection.

The `content_words()` helper keeps only **nouns, verbs, adjectives, adverbs, and proper nouns** — the parts of speech that carry propositional meaning. In this way, the notion of “Idea Units” is implemented directly in code.

**2. Levenshtein edit distance (`rapidfuzz`)**

The `edit_distance_ratio()` function provides the strict anchor for Score 4. When the ratio is ≥ 0.95, the response is treated as essentially verbatim, matching the rubric’s description of the highest proficiency band. Using `rapidfuzz` keeps this computation fast even as the dataset grows.

**3. Multilingual sentence embeddings (`sentence-transformers`)**

The step that connects Scores 2 and 3 is the most linguistically interesting. I use the `paraphrase-multilingual-MiniLM-L12-v2` model to embed both the stimulus and response into a shared semantic vector space. Their **cosine similarity** then measures how much of the meaning is preserved, independent of surface form. Synonyms, flexible word order, and small grammatical slips can still yield a high score when the underlying idea is right.

This is the “human-like” part of the system that moves it beyond simple keyword counting.

**4. Scoring decision rules**

| Score | Label | Decision criteria |
|---|---|---|
| **4** | Exact match | Edit distance ratio ≥ 0.95 |
| **3** | Meaning preserved | Semantic similarity > 0.75 **and** idea-unit coverage > 70% |
| **2** | Close meaning | Similarity > 0.50 **and** coverage ≥ 45% |
| **1** | Partial / unrelated | Coverage ≥ 20% **or** similarity > 25% |
| **0** | Abandoned / silence | When none of the above conditions are met |

### Evaluation

The scoring notebook walks through three small “sanity check” demos before the model is applied to real learner data:

- **Demo 1 – Idea-unit identification:** Shows that `content_words()` pulls out the lexical core of each sentence.  
- **Demo 2 – Synonyms and paraphrases:** Confirms that semantically equivalent responses (different surface wording, same idea) receive high similarity scores, as expected for Score 3.  
- **Demo 3 – Grammatical variation:** Illustrates that tense or agreement shifts are noticed, but meaning can still be recognised as preserved.

### Outputs

- 📓 **Notebook:** [Specific_Test-II/notebook/Test-II_implementation.ipynb](Specific_Test-II/notebook/Test-II_implementation.ipynb)  
- 📄 **PDF export:** [Specific_Test-II/notebook/Test-II_implementation.pdf](Specific_Test-II/notebook/Test-II_implementation.pdf)  
- 📊 **Results:** [Specific_Test-II/output/AutoEIT_Test2_Transcriptions_Completed.xlsx](Specific_Test-II/output/AutoEIT_Test2_Transcriptions_Completed.xlsx)  
- 📝 **Technical summary:** [Specific_Test-II/Technical_Summary.md](Specific_Test-II/Technical_Summary.md)

---

## Technology Stack

| Library | Purpose |
|---|---|
| `faster-whisper` | Multilingual transformer ASR for Test I |
| `pydub` | Controlled audio slicing and pre-processing (Test I) |
| `librosa` | Waveform analysis and visualisation for QA (Test I) |
| `spacy` + `es_dep_news_trf` | Transformer-based lemmatization and POS tagging (Test II) |
| `sentence-transformers` | Multilingual sentence embeddings for semantic similarity (Test II) |
| `rapidfuzz` | Efficient Levenshtein edit distance (Test II) |
| `scikit-learn` | Cosine similarity utilities (Test II) |
| `pandas` / `openpyxl` | Reading and writing the Excel files for both tests |

---

## How to Run

Both parts of the project are implemented as **Google Colab notebooks**, so you can run everything from the browser without configuring a local environment.

1. Open the appropriate notebook in [Google Colab](https://colab.research.google.com/).  
2. Execute the first cell that installs all required dependencies.  
3. When prompted, upload the relevant input Excel or audio files from the `data/` folders.  
4. Run the remaining cells in order. The final cell will write the completed Excel file into the `output/` folder and also offer it for download.

For quick inspection without running any code, you can open the PDF exports of the notebooks, which include all cells and outputs.
