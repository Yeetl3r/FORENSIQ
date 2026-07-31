# FORENSIQ operations notes

Keyboard shortcuts and troubleshooting, extracted from the README.

## Keyboard Shortcuts

| Shortcut | Action |
|---|---|
| `⌘/Ctrl + U` | Focus upload zone |
| `⌘/Ctrl + P` | Export PDF |
| `⌘/Ctrl + R` | Retry last analysis |
| `⌘/Ctrl + K` | Open keyboard shortcuts modal |
| `←` / `→` | Previous / next page (multi-page PDFs) |
| `E` | Toggle ELA heatmap |
| `G` | Toggle GradCAM heatmap |

---

## Troubleshooting

**`torch` wheel fails on Windows**
```powershell
pip install torch==2.6.0 torchvision==0.21.0 --index-url https://download.pytorch.org/whl/cpu
pip install -r requirements.txt
```

**PyMuPDF errors on PDF upload**
Ensure `PyMuPDF>=1.24.10` is installed. The older `fitz` package on PyPI is an unrelated library — do not install it.

**Gemini OCR returns English for a non-Latin image**
The free-tier Flash 2.5 limit is 20 requests/day. When exhausted, `/analyze` falls back to EasyOCR. Either use a paid-tier key or wait for the daily reset. The PDF's AI enrichment will still attempt to identify the language from available OCR data.

**`429 RESOURCE_EXHAUSTED` in logs**
Your Gemini API key has hit its daily quota. Swap in a new key in `backend/.env` and restart the server — no code changes needed.

**EasyOCR slow on first call**
Expected — EasyOCR downloads ~200 MB of recognition checkpoints on first use and caches them in `%USERPROFILE%\.EasyOCR\`. Subsequent calls reuse the in-memory reader pool.

**Frontend can't reach backend**
CORS is locked to `http://localhost:5173` and `http://127.0.0.1:5173`. If you serve the Vite dev server on a different host/port, add the origin to `allow_origins` in `backend/main.py`.

**`GOOGLE_API_KEY` warning in logs**
The `google-genai` SDK prefers `GOOGLE_API_KEY` over `GEMINI_API_KEY` if both env vars are set. FORENSIQ explicitly passes the key from `GEMINI_API_KEY` directly to `genai.Client(api_key=...)`, so this warning is harmless and can be ignored.

---
