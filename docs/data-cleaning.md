# Data-cleaning approach

`nasa_text_cleaners.py` applies source-aware rules rather than one destructive
generic cleanup pass. It aims to:

- remove repeated page furniture and obvious OCR noise;
- preserve readable narrative, transcript turns, and useful technical text;
- avoid embedding unusable chart or table fragments;
- retain mission, source type, filepath, page/section information, and raw
  provenance needed to audit a retrieved chunk.

The cleaners return cleaned report blocks or transcript turns.
`embedding_pipeline.py` then aggregates those semantic records by source file
before chunking, so overlap never crosses file boundaries. Cleaning happens
before chunking because embeddings cannot repair missing content or distinguish
meaningful text from severe OCR artifacts after the two have already been
mixed together.
