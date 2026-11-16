# AI Invoice Extractor: Next.js frontend + Python (FastAPI) backend

Upload an invoice image. A Python backend sends it to Gemini, validates the numbers with plain Python,
and the Next.js page shows the table, validation result and JSON (copy / download).

## Features
- Drag-and-drop upload, preview, "Load sample invoice", and **Run demo (no AI)** (works without an API key)
- Extracts invoice number, dates, supplier, currency, line items, subtotal, tax, total
- Validation (no AI, exact Decimal math): required fields, qty x price = line amount, items sum = subtotal, subtotal + tax = total
- Structured JSON logs, clean typed errors, retries with backoff, per-IP rate limit

## Run locally
```bash
python -m venv .venv && source .venv/bin/activate      # Windows: .venv\Scripts\activate
pip install -r requirements.txt -r requirements-dev.txt
npm install
cp .env.example .env.local                              # Windows: copy .env.example .env.local
# put your key in GEMINI_API_KEY (free: https://aistudio.google.com/apikey); the FastAPI process reads it from the environment:
export GEMINI_API_KEY="your-key"                        # Windows PowerShell: $env:GEMINI_API_KEY="your-key"
npm run dev                                             # starts Next.js (:3000) and FastAPI (:8000) together
```
Open http://localhost:3000. API docs: http://localhost:3000/api/py/docs
Tests (offline, no key needed): `npm test`  (needs `pip install httpx`)

## Deploy on Vercel (free Hobby plan)
1. Push this folder to GitHub and import it in Vercel (framework: Next.js is detected).
2. Add environment variable `GEMINI_API_KEY`. Optional: `GEMINI_MODEL` (default `gemini-flash-latest`).
3. Deploy, then check `https://YOUR-APP.vercel.app/api/py/health` returns `{"status":"ok","has_api_key":true}`.
4. Open the app, press "Load sample invoice" then "Extract data".

## Architecture
```
Browser (app/page.tsx)
   │  POST /api/py/extract  (multipart)
   ▼
next.config.mjs rewrite:  dev -> uvicorn :8000   |   Vercel -> Python function
   ▼
api/index.py (FastAPI)
   ├─ input checks (type, empty, <= 4 MB), per-IP rate limit
   ├─ Gemini call: JSON response schema, temperature 0, 25 s timeout, 3 attempts with backoff
   └─ validate(): pure Python, Decimal arithmetic
```
The model only reads what is printed; all arithmetic is checked in code, so a misread digit is caught.

## Errors and logging
Every failure returns `{success:false, error_type, error}` with a matching HTTP status:
`input_error` (400), `rate_limit_error` (429), `config_error` (500), `api_error` (502/503),
`model_output_error` (502), `unexpected_error` (500, details only in logs).
Logs are one-line JSON on stdout (`extract_started`, `model_call_ok`, `model_call_failed`, `request_failed`, ...),
visible in the terminal locally and in Vercel's Logs tab in production. The API key never leaves the server.
# invoice-host
