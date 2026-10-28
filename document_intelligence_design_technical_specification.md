# Enterprise Document Intelligence & Extraction Pipeline
**Technical Specification & Architecture Design**
**Target Context: Freesbe Document Processing Assessment**

---

## 0. Original Assignment Requirements

> **The Task Brief:**
> 1. Select two AI tools that can assist with document analysis, and briefly explain the advantages of each, along with your recommendation.
> 2. Write a short code snippet demonstrating how to process information extracted from a document—for example, by identifying missing fields, validating data, or classifying information.
> 3. Include a brief explanation of the logic and instructions on how to run the code.
> 4. *(No need to develop a full system, user interface, or external integrations. Dummy data is encouraged).*
> 5. *(In the follow-up meeting, you will be asked to explain the code and make a small, real-time modification to it).*

---

## 1. Executive Summary & Problem Scope

Automotive retail, leasing, financing, and insurance enterprises process diverse, high-volume documentation, including:
- **Vehicle registration cards (רישיון רכב):** Dense tables, standardized numeric codes, engine/VIN details.
- **Financing agreements & promissory notes (הסכמי מימון):** Semi-structured legal text, payment schedules, multi-party signatures.
- **Damage inspection reports & adjuster estimates (דו"חות שמאי):** Mixed unstructured observations, damaged part descriptions, and itemized pricing.

A resilient AI architecture cannot treat these documents as unconstrained raw text-generation inputs. It requires a decoupled two-tier workflow:
1. **Extraction Tier:** Specialized ingestion combining computer vision layout analysis with semantic language modeling.
2. **Operational Validation Tier:** Deterministic schema enforcement, regional data normalization (e.g., Israeli license plates & Israeli ID checksums), missing-value detection, and automated operational routing (`APPROVED`, `NEEDS_REVIEW`, `REJECTED`).

---

## 2. Evaluation of Candidate AI Tools

### Tool 1: Google Gemini 3.5 Flash Lite (via Google AI Studio / `google-genai`)
* **Type:** Multimodal Large Language Model (Vision-Language Model).
* **Interface:** REST / Python SDK (`google-genai`) with native Structured Outputs via Pydantic / JSON Schema.
* **Pricing & Availability:** Free tier available via Google AI Studio; highly cost-effective and fast execution.
* **Pros:**
  * **Constrained Decoding (Structured Outputs):** Enforces valid JSON conforming to explicit business schemas directly during token generation, preventing malformed JSON syntax.
  * **Multimodal Reasoning:** Directly reads images and PDF pages, reconciling multilingual Hebrew/English text and variable layout geometry without requiring manual bounding-box matching.
  * **Zero Setup:** Immediate cloud API access requiring only an API key (configured via `.env`).
* **Cons:**
  * Free-tier requests may be subject to data logging (enterprise production requires Google Cloud Vertex AI with customer data governance enabled).
  * Cloud network latency and rate-limiting constraints under batch peak workloads.

---

### Tool 2: PaddleOCR / Docling
* **Type:** Deep Learning OCR and Document Layout Parser.
* **Interface:** Python Library / Local CLI (`pip install paddleocr` or `pip install docling`).
* **Pricing & Availability:** 100% Free and Open Source (Apache 2.0 License).
* **Pros:**
  * **Air-Gapped & Local:** Runs on internal infrastructure with zero cloud cost and absolute customer privacy.
  * **Spatial Precision:** Delivers exact pixel bounding boxes, table cell hierarchies, and reading order for complex vehicle inspection sheets.
* **Cons:**
  * **No Semantic Intelligence:** Lacks understanding of contextual meaning (e.g., distinguishing a guarantor from a buyer if visual labels change).
  * Requires an additional parsing or rule layer to transform raw tokens into structured domain models.

---

### Architectural Recommendation: The Hybrid Pipeline
The production recommendation is a **two-phase hybrid architecture**:
1. **Phase 1 (Layout & Local Ingestion):** Run **PaddleOCR / Docling** locally to extract raw text, form tables, and spatial tokens, filtering out illegible or blank pages before incurring API costs.
2. **Phase 2 (Semantic Extraction & Normalization):** Forward parsed text or selected crops to **Gemini Flash Lite** using schema-constrained generation to produce normalized, typed business payloads.

---

## 3. End-to-End System Architecture

```
                    +-----------------------------+
                    |      Incoming Document      |
                    |   (PDF, Mobile Photo, Scan) |
                    +--------------+--------------+
                                   |
                                   v
                    +-----------------------------+
                    |       Extraction Layer      |
                    | - Google Gemini 3.5 Flash   |
                    |   Lite (Structured Schema)  |
                    |            -- OR --         |
                    | - PaddleOCR / Docling       |
                    |   (Layout & Table Parser)   |
                    +--------------+--------------+
                                   |
                              Raw Extracted
                              Payload (JSON)
                                   |
                                   v
                    +-----------------------------+
                    |   Validation & Processing   |
                    |       (Pydantic Engine)     |
                    +--------------+--------------+
                                   |
                 +-----------------+-----------------+
                 |                                   |
            [Valid Schema]                   [Invalid / Corrupt]
                 |                                   |
                 v                                   v
      +--------------------+               +--------------------+
      | Business Invariant |               |  STATUS: REJECTED  |
      |   & Rule Engine    |               | (Schema violation, |
      +----------+---------+               |  bad syntax, etc.) |
                 |                         +--------------------+
        +--------+--------+
        |                 |
 [Clean & Complete] [Missing Data /
                     Threshold Alert]
        |                 |
        v                 v
 +---------------+ +---------------+
 |    STATUS:    | |    STATUS:    |
 |   APPROVED    | | NEEDS_REVIEW  |
 |  (Direct DB/  | | (Quarantine / |
 |   ERP write)  | |  Human Review)|
 +---------------+ +---------------+
```

---

## 4. Code Logic Explanation

The core script `document_intelligence_processing_pipeline.py` implements the data contract and routing engine:

1. **Schema Definition (`ExtractedVehicleDocument`)**:
   - `license_plate`: Cleans spaces/hyphens and enforces strict 7 or 8 Israeli numeric digit validation (`@field_validator`).
   - `customer_id`: Enforces the Israeli National ID (Teudat Zehut) official modulo-10 Luhn checksum algorithm (`@field_validator`).
   - `deal_amount` & `estimated_damage`: Ensures non-negative, strictly positive numerical values.
   - Cross-field consistency: Enforces that damage reports always include an `estimated_damage` figure (`@model_validator(mode="after")`).
2. **Classification Fallback**:
   - If document classification was missing from upstream extraction, Hebrew keyword heuristics infer whether it is a `Vehicle License`, `Financing Contract`, or `Damage Report`.
3. **Missing Value Identification**:
   - Tracks optional non-critical attributes (`customer_id`, `deal_amount`, `contract_date`).
4. **Operational Triage & Routing**:
   - **`APPROVED`**: Completely valid schema, all expected fields present, transaction within threshold.
   - **`NEEDS_REVIEW`**: Valid schema, but missing optional fields or deal value exceeds **250,000 ILS** (triggers human supervisor audit).
   - **`REJECTED`**: Validation failure (corrupted OCR, invalid Israeli plate format, bad ID checksum, negative numbers, or missing mandatory cross-field data).
5. **Live Gemini Integration (`extract_with_gemini`)**:
   - Reads `GEMINI_API_KEY` from `.env`.
   - Sends Hebrew document text to `gemini-3.5-flash-lite` with `response_schema=ExtractedVehicleDocument`.
   - Returns typed structured output directly into the validation engine.

---

## 5. Setup & Execution Instructions

### 5.1 Prerequisites
- Python 3.10+
- Installed packages: `pydantic`, `python-dotenv`, `google-genai`

### 5.2 Running the Pipeline
Run the demonstration script from the root folder:
```bash
python document_intelligence_processing_pipeline.py
```

### 5.3 Execution Output
The script automatically runs through 5 test scenarios:
- **Scenario 1:** Clean financing contract with valid Israeli ID checksum -> `APPROVED`
- **Scenario 2:** High-value financing (>250k ILS) with missing optional date -> `NEEDS_REVIEW`
- **Scenario 3:** Corrupted OCR (invalid plate & negative deal) -> `REJECTED` with specific field errors
- **Scenario 4:** Cross-field violation (Damage Report without estimated damage) -> `REJECTED`
- **Scenario 5 (Live Gemini API):** Real extraction of unstructured Hebrew text using `gemini-3.5-flash-lite` -> Structured output -> `APPROVED`

---

> [!NOTE]
> ### 📘 CANDIDATE INTERVIEW & LEARNING REFERENCE
> **The sections below are personal preparation notes for the interview presentation and live-coding modification test.**
> *(Keep these handy for your video walkthrough and real-time live coding session).*

---

## 6. Loom Presentation Walkthrough Guide (3 Minutes)

* **Minute 0:00 - 0:45 | Architectural Framing:**
  * Introduce the two candidate tools: Google Gemini 3.5 Flash Lite (Multimodal LLM) and PaddleOCR/Docling (Local Deep Learning Engine).
  * Highlight the Hybrid Architecture: local layout extraction for privacy/cost control, paired with semantic constrained generation for clean structured outputs.
* **Minute 0:45 - 1:45 | Code & Domain Model:**
  * Open `document_intelligence_processing_pipeline.py`.
  * Walk through `ExtractedVehicleDocument`: Israeli license plate normalization, Israeli National ID (Teudat Zehut) modulo-10 checksum, and positive amount validator.
  * Show `process_extracted_document`: classification fallback, missing-field detection, and the three-tier routing (`APPROVED`, `NEEDS_REVIEW`, `REJECTED`).
* **Minute 1:45 - 2:30 | Live Terminal Execution:**
  * Run `python document_intelligence_processing_pipeline.py`.
  * Highlight the outputs:
    1. Clean contract passing directly.
    2. High-value transaction flagged for review.
    3. Corrupted OCR rejected with granular field errors.
    4. Cross-field rule enforcement.
    5. Live unstructured Hebrew extraction using `gemini-3.5-flash-lite`.
* **Minute 2:30 - 3:00 | Production Readiness:**
  * Mention scaling to asynchronous processing via Celery / RabbitMQ / SQS for bulk automotive batch uploads.

---

## 7. Live Coding Scenarios (Interview Preparation)

### Scenario A: Israeli National ID (Teudat Zehut) Validation with Checksum
If asked in the live meeting to add or explain ID verification:
```python
@field_validator("customer_id")
@classmethod
def validate_israeli_id(cls, value: Optional[str]) -> Optional[str]:
    if value is None:
        return None
    cleaned = value.replace("-", "").replace(" ", "").strip().zfill(9)
    if not cleaned.isdigit() or len(cleaned) != 9:
        raise ValueError(f"Israeli ID must contain exactly 9 numeric digits: '{value}'")
    total = sum(
        (val - 9 if val > 9 else val)
        for i, digit in enumerate(cleaned)
        for val in [int(digit) * ((i % 2) + 1)]
    )
    if total % 10 != 0:
        raise ValueError(f"Invalid Israeli ID checksum: '{value}'")
    return cleaned
```

### Scenario B: Conditional Cross-Field Business Logic
If asked: *"Ensure Damage Reports always include an adjuster estimate"*:
```python
@model_validator(mode="after")
def verify_cross_field_rules(self):
    if self.category == DocumentCategory.DAMAGE_REPORT and self.estimated_damage is None:
        raise ValueError("Damage Reports require an 'estimated_damage' value.")
    return self
```

### Scenario C: Adding a New Document Type (e.g. Insurance Certificate)
If asked to support car insurance policies:
1. Add `INSURANCE_CERTIFICATE = "Insurance Certificate"` to `DocumentCategory`.
2. Add keyword match in `process_extracted_document`:
   ```python
   elif any(k in snippet for k in ["ביטוח", "פוליסה", "חברת ביטוח"]):
       inferred_category = DocumentCategory.INSURANCE_CERTIFICATE
   ```
3. Add optional field `policy_number: Optional[str] = None` to `ExtractedVehicleDocument`.