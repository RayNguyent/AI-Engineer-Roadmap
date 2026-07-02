# Fault-Tolerant Structured Extraction Microservice

## Project Overview

A production-style AI extraction service that converts unstructured documents into validated, schema-compliant JSON using:

- FastAPI
- Claude (Anthropic)
- Pydantic
- OCR (Tesseract)
- Multi-format document ingestion
- Validation-driven retry loops
- Observability and metrics

The system accepts uploaded files, extracts text, converts unstructured content into structured JSON, validates the result, and automatically repairs invalid outputs through a self-correction loop.

---

# End-to-End Workflow

```text
User Uploads File
        │
        ▼
FastAPI Upload Endpoint
        │
        ▼
Document Ingestion Layer
        │
        ├── PDF Parser
        ├── OCR Image Parser
        ├── DOCX Parser
        └── TXT Parser
        │
        ▼
Normalized Document Text
        │
        ▼
Claude Extraction Engine
        │
        ▼
JSON Response
        │
        ▼
Pydantic Validation
        │
        ├── Valid
        │      ▼
        │   Return Result
        │
        └── Invalid
               ▼
        Validation Errors
               ▼
        Repair Prompt
               ▼
        Claude Self-Correction
               ▼
        Revalidate
               ▼
        Success / Failure
```

---

# Supported Input Formats

## Documents

- PDF
- TXT
- DOCX

## Images

- PNG
- JPG
- JPEG

## OCR Pipeline

```text
Image
  ↓
Tesseract OCR
  ↓
Plain Text
  ↓
Claude
```

This allows extraction from scanned invoices, screenshots, and image-based documents.

---

# Core Features

## 1. Structured Extraction

Input:

```text
Invoice #INV001

Customer: John Doe

Amount: $500

Due Date: 2026-07-15
```

Output:

```json
{
  "invoice_number": "INV001",
  "customer_name": "John Doe",
  "amount": 500,
  "due_date": "2026-07-15"
}
```

---

## 2. Schema-Guided Validation

All outputs are validated using Pydantic models.

Example:

```python
Invoice.model_validate(data)
```

Validation guarantees:

- Required fields exist
- Correct data types
- Allowed values only
- Business rules are respected

---

## 3. Self-Correction Loop

Instead of immediately failing when Claude returns invalid output:

```json
{
  "amount": "five hundred"
}
```

The service automatically:

```text
Validation Error
      ↓
Capture Error
      ↓
Feed Error Back To Claude
      ↓
Repair JSON
      ↓
Validate Again
```

Example:

```text
amount:
Input should be a valid number
```

Claude receives:

```text
Your previous response failed validation.

Validation Error:

amount must be numeric.

Fix the JSON.
```

This creates a fault-tolerant extraction pipeline.

---

# Business Validation

## Invoice Validation

### Required Fields

```python
invoice_number
customer_name
amount
due_date
```

### Financial Rules

```python
amount >= 0
```

Rejects:

```json
{
  "amount": -100
}
```

---

### Customer Validation

Rejects:

```json
{
  "customer_name": ""
}
```

---

## Email Validation

Uses:

```python
EmailStr
```

Rejects:

```json
{
  "sender": "not-an-email"
}
```

---

## Date Validation

Supports:

```python
start_date
end_date
```

Rule:

```text
end_date > start_date
```

Rejects:

```json
{
  "start_date": "2026-07-10",
  "end_date": "2026-07-05"
}
```

---

## Enum Validation

Support Ticket Status:

```python
OPEN
PENDING
CLOSED
```

Support Ticket Priority:

```python
LOW
MEDIUM
HIGH
```

Rejects:

```json
{
  "priority": "super-high"
}
```

---

# Prompt Engineering

## Structured System Prompt

Defines:

- Output format
- Extraction rules
- Hallucination prevention
- JSON-only responses

---

## Few-Shot Examples

Prompt contains:

```text
Example Input
    ↓
Example Output
```

This improves extraction reliability.

---

## Repair Prompts

Validation errors are injected into repair prompts.

Example:

```text
Previous Output:

{
  "amount": "five hundred"
}

Validation Errors:

amount must be numeric

Return corrected JSON only.
```

---

# Observability

## Logging

The service logs:

### Document Type

```text
.pdf
.png
.docx
.txt
```

---

### Extraction Attempts

```text
Attempt 1/3
Attempt 2/3
```

---

### Validation Errors

Example:

```text
amount
Input should be a valid number
```

---

### Retry Count

Tracks automatic recovery attempts.

---

### Final Status

```text
Extraction Successful
```

or

```text
Maximum Retries Exceeded
```

---

# Metrics

## Recorded Metrics

### Successful Extractions

```python
successful_extractions
```

---

### Failed Extractions

```python
failed_extractions
```

---

### Validation Failures

```python
validation_failures
```

---

### Total Retries

```python
total_retries
```

---

### Success Rate

```python
successful /
(total successful + failed)
```

---

### Average Retries

```python
total_retries /
successful_extractions
```

---

# FastAPI API Layer

## Extract Endpoint

```http
POST /extract
```

Upload:

```text
invoice.pdf
```

Response:

```json
{
  "invoice_number": "INV001",
  "customer_name": "John Doe",
  "amount": 500,
  "due_date": "2026-07-15"
}
```

---

## Metrics Endpoint

```http
GET /metrics
```

Response:

```json
{
  "successful_extractions": 10,
  "failed_extractions": 1,
  "validation_failures": 3,
  "total_retries": 3,
  "success_rate": 0.91,
  "average_retries": 0.3
}
```

---

# Testing Coverage

## Parser Tests

- PDF Parsing
- DOCX Parsing
- TXT Parsing
- OCR Parsing

---

## Validation Tests

### Missing Fields

```json
{}
```

Expected:

```text
ValidationError
```

---

### Wrong Types

```json
{
  "amount": "five hundred"
}
```

Expected:

```text
ValidationError
```

---

### Invalid Dates

```json
{
  "due_date": "banana"
}
```

Expected:

```text
ValidationError
```

---

### Invalid Enums

```json
{
  "priority": "super-high"
}
```

Expected:

```text
ValidationError
```

---

### Empty Responses

```json
{}
```

Expected:

```text
ValidationError
```

---

### Malformed JSON

```json
{
    "invoice_number":
```

Expected:

```text
JSONDecodeError
```

---

## Retry Loop Tests

Verifies:

```text
Bad JSON
     ↓
Validation Error
     ↓
Repair Prompt
     ↓
Corrected JSON
     ↓
Success
```

---

## API Tests

Verifies:

```text
File Upload
     ↓
FastAPI Route
     ↓
Response
```

---

## End-to-End Tests

Full Pipeline:

```text
PDF
 ↓
Parser
 ↓
Claude
 ↓
Validation
 ↓
Retry Loop
 ↓
Final JSON
```

---

# Criteria Mapping

## ✅ Structured Extraction

Converts unstructured documents into strict JSON.

---

## ✅ Validation Loop Retries

Validation failures are fed directly back into Claude.

---

## ✅ Business Rule Enforcement

Validates:

- Required fields
- Amount constraints
- Email format
- Date relationships
- Allowed enum values

---

## ✅ Fault Tolerance

Automatically repairs invalid outputs through retry loops.

---

## ✅ Multi-Format Ingestion

Supports:

- PDF
- DOCX
- TXT
- PNG
- JPG
- JPEG

---

## ✅ Observability

Provides:

- Logs
- Retry tracking
- Validation diagnostics
- Metrics

---

## ✅ Failure Scenario Testing

Tests:

- Missing fields
- Wrong types
- Invalid dates
- Invalid enums
- Empty responses
- Malformed JSON

---

# Final Result

This project demonstrates a complete:

**Fault-Tolerant Structured Extraction Microservice with Integrated Self-Correction Loops**

capable of:

- Ingesting multiple document formats
- Extracting structured information with Claude
- Validating outputs with Pydantic
- Automatically repairing invalid responses
- Tracking reliability through logs and metrics
- Exposing a production-style FastAPI API
- Verifying resilience through automated tests