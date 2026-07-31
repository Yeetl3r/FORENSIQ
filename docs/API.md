# FORENSIQ API reference

Extracted from the README to keep the front page readable.

### `POST /analyze`
`multipart/form-data`, field `file`. Accepts PDF, PNG, JPG, WEBP, TIFF.

```jsonc
// Response (abridged)
{
  "verdict": "FORGED",              // GENUINE | SUSPICIOUS | FORGED
  "confidence": 0.87,
  "reason": "Pixel compression artifacts detected in strip 3/10...",
  "session_id": "A3F8C12E",
  "signals": {
    "ela":      { "score": 0.72, "confidence": 0.81, "heatmap_url": "...", "regional_scores": [] },
    "cnn":      { "score": 0.91, "confidence": 0.88, "backbone": "EfficientNet-B0", "trained": true },
    "font":     { "score": 0.33, "confidence": 0.62, "flagged_regions": [], "total_regions": 184 },
    "metadata": { "score": 0.60, "confidence": 0.55, "anomalies": ["editor fingerprint: photoshop"] }
  },
  "gradcam_url": "/static/heatmaps/gc_…png",
  "original_url": "/static/originals/orig_…png",
  "regional_language": {
    "detected": true,
    "script": "hindi",
    "language_name": "Hindi",
    "confidence": 0.96,
    "ocr_text": "...",
    "ocr_lines": [{ "text": "...", "confidence": 0.94, "segments": [] }],
    "flagged_characters": [],
    "glyph_count": 143,
    "kerning_heatmap": [[...]],   // 8×16 float grid
    "source": "gemini-2.5-flash"
  },
  "pages": [{ "page_number": 1, "gradcam_url": "...", "ela_heatmap_url": "...", "timeline": [] }],
  "bounding_boxes": [{ "x": 120, "y": 340, "width": 90, "height": 45, "confidence": 0.89, "label": "Region A" }],
  "elapsed_ms": 2340,
  "sha256": "..."
}
```

### `POST /enrich`
Body: `{ "result": <full /analyze response> }`
Returns `{ "enrichment": "<3-paragraph forensic narrative (ASCII)>" }`
Used by PDF export. Thinking disabled (`thinking_budget=0`) to prevent token budget exhaustion.

### `POST /enrich/ocr`
Body: `{ "result": <full /analyze response> }`
Returns `{ "enrichment": "<LANGUAGE / TRANSLITERATION / TRANSLATION / OBSERVATIONS>" }`
Converts non-Latin OCR text to ASCII-safe forensic insights for jsPDF rendering.

### `POST /ocr`
Body: `{ "file_base64": "<base64>", "filename": "doc.pdf" }`
Runs OCR-only pipeline (Gemini Vision → EasyOCR fallback) without re-running ELA/CNN/font/metadata.

### `POST /adversarial/apply`
```jsonc
{
  "file_base64": "<base64>",
  "operation": "brightness" | "jpeg_compress" | "copymove",
  "intensity": 0.5,      // -1.0 → 1.0
  "filename": "doc.jpg"  // optional
}
```
Returns full `/analyze` response plus `modified_image_base64`, `source_operation`, `source_intensity`.

---

## OCR Engine

OCR runs two-tier:

1. **Gemini 2.5 Flash Vision (primary)** — sends the document image directly to the model, which recognises all six scripts natively without per-script model downloads. Returns structured JSON (language name, script, confidence, lines). `thinking_budget=0` keeps latency low and avoids hidden thinking tokens eating the output budget.

2. **EasyOCR (fallback)** — activated when the Gemini API is unavailable or rate-limited. Uses a lazy per-script reader pool: English pass runs first; if the output contains Indic Unicode codepoints, a dedicated Indic reader for that script re-runs on the same image.

Both paths return the same dict shape so the frontend `RegionalLanguagePanel` component is engine-agnostic.

---

## PDF Export

Click **⌘ P** (or `Ctrl+P`) on the Verdict screen to export. The report is generated entirely client-side by jsPDF and includes:

- Styled FORENSIQ header + timestamp
- Verdict block with ensemble confidence
- Signal breakdown table (ELA / CNN / Font / Metadata scores + status)
- GradCAM snapshot captured from the live Forensics Viewer
- Forgery localisation bounding boxes
- **OCR ANALYSIS** — AI-generated language identification, Roman transliteration, English translation, and forensic observations (ASCII-safe; never raw Unicode)
- **AI FORENSIC ENRICHMENT** — 3-paragraph formal forensic narrative referencing actual signal scores, anomalies, and recommended next steps

Long sections flow across pages automatically with `(CONTINUED)` headers.

---
