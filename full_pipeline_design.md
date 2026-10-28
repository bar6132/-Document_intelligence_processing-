# Enterprise Document Intelligence Processing Pipeline - Architecture & Design
**Document ID:** `DOC-DESIGN-PIPELINE-FULL`  
**Target:** Freesbe Enterprise Multimodal Document Intelligence System  
**File Reference:** [`document_intelligence_processing_pipeline.py`](file:///c:/Users/Bar_PC/Desktop/freesbe/document_intelligence_processing_pipeline.py)

---

## 1. System Overview & Objective

This pipeline demonstrates a production-ready, enterprise-grade document intelligence system designed for Freesbe's vehicle financing, leasing, and registration operations. 

Unlike simple script demos, this full pipeline bridges the gap between **unstructured real-world document ingestion** (e.g., Hebrew camera photos or scanned contracts) and **deterministic business validation**:
1. **Multimodal Extraction Tier:** Leverages Google Gemini 2.5 / 3.5 Flash via the official `google-genai` SDK with native schema-constrained decoding (`response_schema`).
2. **Resilience & Fallback Tier:** Implements model fallback sequences across candidate models (`gemini-3.5-flash-lite`, `gemini-3.1-flash-lite`, etc.) to absorb rate limits and API version transitions.
3. **Deterministic Business Enforcement Tier:** Normalizes Israeli license plates, performs cross-field conditional validation, identifies missing audit fields, and routes documents through status gates (`APPROVED`, `NEEDS_REVIEW`, `REJECTED`).

---

## 2. Technical Stack & Tool Selection

| Component | Technology | Rationale |
| :--- | :--- | :--- |
| **Language & Runtime** | Python 3.10+ | Standard ecosystem for AI/ML and data pipelines. |
| **PDF Ingestion** | `pymupdf` (v1.28+) | High-speed, high-fidelity PDF text and metadata extraction. |
| **DOCX Ingestion** | `python-docx` (v1.2+) | Standard Word document parser; extracts paragraphs and table cells. |
| **Multimodal LLM SDK** | `google-genai` (v2.x) | Official modern Google GenAI SDK; supports direct Pydantic class binding in `response_schema`. |
| **Validation Engine** | `pydantic` (v2.x) | High-performance Rust-backed data validation and serialization; enforces strict business rules. |
| **Environment Config** | `python-dotenv` | Clean segregation of API keys (`GEMINI_API_KEY`) and secrets from source code. |
| **Alternative Engine** | `PaddleOCR` / `Docling` | Evaluated for on-premise layout analysis and air-gapped data compliance. |

---

## 3. End-to-End System Architecture

```
+-------------------------------------------------------------+
|                      Incoming Document                      |
|           (.pdf, .docx, .txt, or raw string)                |
+-------------------------------------------------------------+
                              |
                              v
+-------------------------------------------------------------+
|               Phase 0: File Ingestion Layer                 |
|       - load_document_text(source):                         |
|         * .txt  -> utf-8 reader                             |
|         * .pdf  -> pymupdf text stream                      |
|         * .docx -> python-docx paragraph/table extractor    |
+-------------------------------------------------------------+
                              |
                              v
+-------------------------------------------------------------+
|               Phase 1: Multimodal LLM Tier                  |
|       - Model: Google Gemini 3.5 Flash Lite                 |
|       - Constrained Decoding: ExtractedVehicleDocument      |
|       - Fallback: Multiple candidate models on failure      |
+-------------------------------------------------------------+
                              |
                              v
+-------------------------------------------------------------+
|           Phase 2: Validation & Normalization               |
|       - License Plate: Strip hyphens, assert 7/8 digits     |
|       - Deal Amount: Positive float assertion               |
|       - Cross-Field: Damage Reports must have damage cost   |
+-------------------------------------------------------------+
                              |
            +-----------------+-----------------+
            | Success                           | ValidationError
            v                                   v
+-----------------------------+   +-----------------------------+
| Phase 3: Missing Fields &   |   |   Phase 5: Automated Reject |
| Business Routing            |   |   Status: REJECTED          |
| - Audit: customer_id, date  |   |   Detailed field-level logs |
| - High-risk (> 250k ILS)    |   +-----------------------------+
+-----------------------------+
            |
            +-----------------+-----------------+
            | Clean                             | Missing / High Risk
            v                                   v
+-----------------------------+   +-----------------------------+
|   Status: APPROVED          |   |   Status: NEEDS_REVIEW      |
|   Direct to ERP / Core DB   |   |   Sent to Underwriting Desk |
+-----------------------------+   +-----------------------------+
```

---

## 4. Deep Dive into Implementation Details

### 4.1 Schema Modeling & Cross-Field Logic (`ExtractedVehicleDocument`)
* **Category Tagging:** Classifies documents into `VEHICLE_LICENSE`, `FINANCING_CONTRACT`, `DAMAGE_REPORT`, or `UNKNOWN`.
* **Field Validation (`clean_and_validate_plate`):** Uses regex to strip non-digits and ensures length is 7 or 8 digits.
* **Cross-Field Model Validator (`validate_damage_report_rules`):**
  Using `@model_validator(mode="after")`, enforces domain rules across multiple fields:
  ```python
  if self.category == DocumentCategory.DAMAGE_REPORT and self.estimated_damage is None:
      raise ValueError("Damage Reports require an 'estimated_damage' value.")
  ```

### 4.2 Resilient Live LLM Extraction (`extract_with_gemini`)
* Configures `response_mime_type="application/json"` and `response_schema=ExtractedVehicleDocument` to force the Gemini model's token sampling engine to output only tokens conforming to the JSON schema.
* Loops through a prioritized candidate model list with error tracking:
  ```python
  candidate_models = [preferred_model, "gemini-3.1-flash-lite", "gemini-flash-lite-latest"]
  ```

### 4.3 Automated Decision Engine (`process_extracted_document`)
* Segregates output into three standard statuses:
  * **`APPROVED`**: Document passed all data contract constraints, contains required audit fields, and falls within standard monetary limits.
  * **`NEEDS_REVIEW`**: Document is syntactically valid but flagged for manual human inspection due to missing optional fields (`customer_id`, `contract_date`) or deal value exceeding 250,000 ILS.
  * **`REJECTED`**: Malformed or unparseable OCR output with an itemized list of failed fields.

---

## 5. Execution & Verification Guide

### Prerequisites
```bash
pip install pydantic python-dotenv google-genai
```

### Environment Setup (for live API calls)
Create a `.env` file in the project root:
```env
GEMINI_API_KEY=your_gemini_api_key_here
```

### Run the Pipeline
```bash
python document_intelligence_processing_pipeline.py
```

### Scenarios Tested in Demonstration:
1. **Scenario 1**: Clean, valid vehicle financing contract -> `APPROVED`.
2. **Scenario 2**: Missing optional date & high amount (>250k) -> `NEEDS_REVIEW`.
3. **Scenario 3**: Corrupted OCR (Invalid plate & negative deal) -> `REJECTED`.
4. **Scenario 4**: Cross-field rule violation (Damage report without estimate) -> `REJECTED`.
5. **Scenario 5**: Live Gemini extraction sending unstructured Hebrew text -> Validated and `APPROVED`.
