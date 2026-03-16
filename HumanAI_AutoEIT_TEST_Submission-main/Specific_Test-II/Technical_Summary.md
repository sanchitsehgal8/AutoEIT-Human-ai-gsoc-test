
﻿#             Technical Summary

## Problem Statement

This component focuses on designing a reliable, reproducible pipeline for **automatically grading** Spanish Elicited Imitation Tasks (EIT) in line with the detailed Ortega (2000) rubric.

## Transformer-Based Preprocessing

As a first step, I use the RoBERTa-based spaCy model `es_dep_news_trf` for linguistic preprocessing. While this model is relatively heavy, it gives two key advantages that are important for fair scoring:

* **Context-aware lemmatization:** Inflected Spanish forms (for example, verbs and nouns) are reduced to their base lemmas (e.g., *fuman* → *fumar*). This keeps the system from penalising learners for simple morphological variation when the underlying idea is correct.  
* **Selective part-of-speech (POS) filtering:** I focus on **nouns, verbs, adjectives, adverbs, and proper nouns** as the core “idea-bearing” categories.

Instead of counting raw word overlap, the system evaluates **concepts** (Idea Units). The main question becomes: did the learner successfully reproduce the **propositional meaning** of the sentence?

## Hybrid Scoring Methodology

Reading the Ortega (2000) rubric made it clear that no single metric could reflect a human rater’s judgment. I wanted the system to differentiate rote memorisation from genuine understanding, not just tally string matches.

To achieve that, I combine three complementary signals:

1. **Structural alignment (strictness anchor):** A **Levenshtein edit distance** ratio gives a tight measure of how close the learner’s response is to the original string—perfect for catching near-verbatim, Score‑4 repetitions.  
2. **Semantic preservation (meaning bridge):** Transformer-based **semantic similarity** looks at how close the two sentences are in conceptual space. This “human-style” piece prevents the system from punishing small slips, synonyms, or word order changes when the intended meaning is intact.  
3. **Core idea coverage (gist capture):** An *Idea Unit extraction* step checks how many of the crucial content lemmas from the stimulus appear in the response. This is especially important for distinguishing borderline Score 1 vs. Score 2 cases.

The combination of **structure, meaning, and core idea coverage** leads to a scoring pipeline that behaves much more like a thoughtful teacher than a brittle, single-metric script.

## Implementation Details of Metrics

**1. Structural metric (Levenshtein distance)**

* **Tool:** I use `rapidfuzz` to compute the Levenshtein similarity ratio between the target stimulus and the learner’s answer.  
* **Logic (first pass filter):** When the ratio is ≥ 0.95, the response is treated as a “near-perfect match” and assigned **Score 4** immediately.  
* **Goal:** Quickly identify verbatim responses and reserve more expensive semantic calculations for the remaining, less obvious cases.

**2. Linguistic coverage (Idea Unit extraction)**

* **Tool:** The `es_dep_news_trf` transformer model provides POS tags and lemmas for Spanish text.  
* **Filter:** From each sentence I extract only lemmas tagged as NOUN, VERB, ADJ, or ADV, treating these as candidate Idea Units.  
* **Calculation:** I compute the overlap between the sets of content-word lemmas for the stimulus and the response.  
* **Threshold:** This coverage value is a key signal for **Score 2** (coverage > 50%) versus **Score 1** (coverage < 50%).

**3. Semantic metric (cosine similarity)**

* **Model:** A Sentence-Transformer model (`paraphrase-multilingual-MiniLM-L12-v2`) maps entire sentences into dense vectors in a shared semantic space.  
* **Metric:** I then compute **cosine similarity** between the stimulus and response vectors.  
* **Decision (Score‑3 bridge):** A high similarity score (> 0.75), together with strong idea-unit coverage, is the main condition for assigning **Score 3**, signalling that the learner has preserved the full meaning despite surface differences.

## Final Scoring Function

| Score | Description | Criteria (internal logic) |
| ----- | ----------- | ------------------------- |
| **4** | **Exact match** | Response is almost identical to the target (very high edit distance ratio, > 95%). |
| **3** | **Meaning preserved** | Same meaning with different wording (high semantic similarity > 75% **and** high Idea Unit coverage > 70%). |
| **2** | **Close meaning** | More than half of the key propositional content is present (similarity > 50% **and** Idea Unit coverage > 45%). |
| **1** | **Partial / unrelated** | Only a small part of the target meaning is captured (coverage > 20% **or** similarity > 25%). |
| **0** | **Abandoned / silence** | Little to no meaningful content, or no response at all. |
