# Document Intelligence Processing Pipeline

Extraction and validation pipeline for vehicle financing, leasing, and registration documents (Freesbe task). It reads Hebrew documents (text, PDF, DOCX, or photos/scans), extracts structured fields with Google Gemini, validates them against business rules, and routes each document to `APPROVED`, `NEEDS_REVIEW`, or `REJECTED`.

## Features

- **Multi-format input:** `.txt`, `.pdf` (digital and scanned), `.docx`, `.png`/`.jpg`/`.webp`, or a raw text string.
- **Schema-constrained extraction:** Gemini returns JSON matching the `ExtractedVehicleDocument` Pydantic model.
- **Model fallback:** tries several Gemini models in order if one fails or is rate-limited.
- **Business validation:**
  - Israeli license plates normalized to 7 or 8 digits.
  - Israeli National ID (Teudat Zehut) checked with the modulo-10 algorithm.
  - Monetary amounts must be positive.
  - Damage reports must include an estimated damage amount.
- **Routing rules:**
  - `APPROVED`: valid, all audit fields present, amount up to 250,000 ILS.
  - `NEEDS_REVIEW`: missing `customer_id`, `deal_amount`, or `contract_date`, or amount above 250,000 ILS.
  - `REJECTED`: fails validation, with field-level error details.

## Project Layout

| File | Purpose |
| :--- | :--- |
| `document_intelligence_processing_pipeline.py` | Main pipeline: schemas, validation, ingestion, Gemini calls, demo scenarios |
| `document_intelligence_design_technical_specification.md` | Technical specification |
| `full_pipeline_design.md` | Architecture and design notes |
| `requirements.txt` | Python dependencies |
| `.env` | Local secrets (not committed) |

## Setup

Requires Python 3.10+.

```powershell
python -m venv venv
.\venv\Scripts\Activate.ps1
python -m pip install -r requirements.txt
```

On macOS/Linux, activate with `source venv/bin/activate`.

Create a `.env` file in the project root for live Gemini calls:

```env
GEMINI_API_KEY=your_gemini_api_key_here
```

## Usage

Run the demonstration scenarios:

```powershell
python document_intelligence_processing_pipeline.py
```

Scenarios 1 to 4 use built-in sample data and run offline. Scenario 5 sends a sample Hebrew contract to Gemini and runs only when `GEMINI_API_KEY` is set.

To process a file or text from your own code:

```python
from document_intelligence_processing_pipeline import process_document_source

result = process_document_source("contract.pdf")
print(result["status"], result["errors"], result["missing_fields"])
```

`result` has the keys `status`, `document`, `missing_fields`, and `errors`.

## Dependencies

- `pydantic`: data models and validation
- `google-genai`: Gemini API client
- `python-dotenv`: loads `.env`
- `pymupdf`: PDF text extraction
- `python-docx`: DOCX text and table extraction
