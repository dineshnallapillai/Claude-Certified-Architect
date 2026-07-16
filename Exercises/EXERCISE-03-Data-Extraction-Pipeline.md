# Exercise 3: Build a Structured Data Extraction Pipeline

**Source:** Claude Certified Architect – Foundations Exam Guide v1.0, Section 8  
**Objective:** Practice designing JSON schemas, using tool_use for structured output, implementing validation-retry loops, designing batch processing, and routing human review by confidence.  
**Domains Reinforced:** D4 (Prompt Engineering & Structured Output), D5 (Context Management & Reliability)  
**Estimated Duration:** 3-4 hours  
**Difficulty:** High

---

## Overview

You will build a document data extraction system that processes invoices to extract structured fields reliably. This exercise reinforces:

- JSON schema design (required, optional, nullable, enum fields)
- Tool use with structured output (tool_choice enforcement)
- Validation-retry loops (include error context, retry up to N times)
- Batch processing with Message Batches API
- Human review workflow with confidence-based routing
- Field-level accuracy measurement

---

## Part 1: Design the JSON Schema

### Step 1.1: Define Invoice Extraction Schema

Your goal: Extract structured invoice data with high reliability.

```python
from pydantic import BaseModel
from typing import Optional, List
from enum import Enum

class PaymentStatus(str, Enum):
    PAID = "paid"
    UNPAID = "unpaid"
    PARTIALLY_PAID = "partially_paid"
    UNCLEAR = "unclear"  # When document is ambiguous

class LineItem(BaseModel):
    description: str
    quantity: int
    unit_price: float
    total: float
    confidence_score: float  # 0.0-1.0

class InvoiceData(BaseModel):
    """Extracted invoice data with optional and nullable fields"""
    
    # Required fields (always present in invoices)
    invoice_number: str
    invoice_date: str  # ISO 8601 format
    
    # Optional fields (may not exist in document)
    vendor_name: Optional[str] = None
    vendor_tax_id: Optional[str] = None
    customer_name: Optional[str] = None
    customer_tax_id: Optional[str] = None
    
    # Nullable fields (might exist but be unclear)
    due_date: Optional[str] = None  # Null if not stated
    payment_terms: Optional[str] = None  # e.g., "Net 30"
    
    # Line items
    line_items: List[LineItem]
    
    # Financial totals
    subtotal: float
    tax_amount: Optional[float] = None  # May not be itemized
    total_amount: float
    
    # Payment status
    payment_status: PaymentStatus
    
    # Extraction metadata
    confidence_scores: dict = {
        "overall": float,  # Average across all fields
        "invoice_number": float,
        "total_amount": float,
        "line_items": float
    }
    
    # Error detection
    validation_issues: List[str] = []  # e.g., "Line items don't sum to subtotal"
    
    class Config:
        json_schema_extra = {
            "example": {
                "invoice_number": "INV-2024-001",
                "invoice_date": "2024-01-15",
                "vendor_name": "Acme Corp",
                "total_amount": 1250.00,
                "payment_status": "unpaid"
            }
        }

```

### Step 1.2: Design Schema in JSON Schema Format

Convert to JSON Schema for use with tool_use:

```json
{
  "type": "object",
  "properties": {
    "invoice_number": {
      "type": "string",
      "description": "Unique invoice identifier"
    },
    "invoice_date": {
      "type": "string",
      "format": "date",
      "description": "Invoice issue date (YYYY-MM-DD)"
    },
    "vendor_name": {
      "type": ["string", "null"],
      "description": "Vendor/supplier name if present"
    },
    "line_items": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "description": { "type": "string" },
          "quantity": { "type": "integer" },
          "unit_price": { "type": "number" },
          "total": { "type": "number" },
          "confidence_score": { "type": "number", "minimum": 0, "maximum": 1 }
        },
        "required": ["description", "quantity", "unit_price", "total"]
      }
    },
    "total_amount": {
      "type": "number",
      "description": "Total invoice amount"
    },
    "payment_status": {
      "type": "string",
      "enum": ["paid", "unpaid", "partially_paid", "unclear"],
      "description": "Current payment status"
    },
    "confidence_scores": {
      "type": "object",
      "properties": {
        "overall": { "type": "number", "minimum": 0, "maximum": 1 },
        "invoice_number": { "type": "number" },
        "total_amount": { "type": "number" }
      }
    },
    "validation_issues": {
      "type": "array",
      "items": { "type": "string" },
      "description": "Issues found during extraction (e.g., math errors)"
    }
  },
  "required": [
    "invoice_number",
    "invoice_date",
    "line_items",
    "total_amount",
    "payment_status",
    "confidence_scores"
  ]
}
```

### Step 1.3: Design Extraction Tool with tool_choice

```python
def create_extraction_tool():
    return {
        "name": "extract_invoice",
        "description": """Extract structured invoice data from an invoice document.
        
        Input: Invoice text/content
        Output: Structured invoice data with confidence scores
        
        Guidelines:
        - Include confidence_scores for each field (0.0 = not found, 1.0 = certain)
        - Mark fields as null if information is not in the document (don't fabricate)
        - Include validation_issues if math doesn't add up or inconsistencies exist
        - For payment_status, use "unclear" if the document doesn't clearly state it
        """,
        "input_schema": {
            "type": "object",
            "properties": {
                "invoice_content": {
                    "type": "string",
                    "description": "Full text content of invoice document"
                }
            },
            "required": ["invoice_content"]
        }
    }
```

---

## Part 2: Implement Validation-Retry Loop

### Step 2.1: Single-Pass Extraction

```python
def extract_invoice_single_pass(invoice_text: str):
    """Extract invoice data in one pass with tool_use"""
    
    response = claude.messages.create(
        model="claude-opus-4-8",
        max_tokens=1024,
        tools=[create_extraction_tool()],
        tool_choice={"type": "tool", "name": "extract_invoice"},  # Force this tool
        messages=[
            {
                "role": "user",
                "content": f"Extract invoice data:\n\n{invoice_text}"
            }
        ]
    )
    
    # Extract tool result
    tool_result = next(
        (block.input for block in response.content if block.type == "tool_use"),
        None
    )
    
    return tool_result  # InvoiceData
```

### Step 2.2: Validation Function

```python
def validate_extraction(extraction: InvoiceData, original_text: str) -> List[str]:
    """Validate extracted data for consistency and accuracy"""
    
    issues = []
    
    # 1. Math validation: line items should sum to subtotal
    line_items_total = sum(item.total for item in extraction.line_items)
    if abs(line_items_total - extraction.subtotal) > 0.01:  # Allow 1¢ rounding
        issues.append(
            f"Line items sum ({line_items_total:.2f}) doesn't match subtotal ({extraction.subtotal:.2f})"
        )
    
    # 2. Total amount validation
    expected_total = extraction.subtotal + (extraction.tax_amount or 0)
    if abs(expected_total - extraction.total_amount) > 0.01:
        issues.append(
            f"Calculated total ({expected_total:.2f}) doesn't match stated total ({extraction.total_amount:.2f})"
        )
    
    # 3. Presence validation: required fields
    if not extraction.invoice_number:
        issues.append("Invoice number is missing")
    
    if not extraction.line_items:
        issues.append("No line items found")
    
    # 4. Date format validation
    try:
        from datetime import datetime
        datetime.fromisoformat(extraction.invoice_date)
    except ValueError:
        issues.append(f"Invalid date format: {extraction.invoice_date}")
    
    # 5. Confidence validation: flag low-confidence critical fields
    if extraction.confidence_scores.get("total_amount", 0) < 0.7:
        issues.append(
            f"Low confidence on total_amount: {extraction.confidence_scores['total_amount']:.2f}"
        )
    
    return issues
```

### Step 2.3: Validation-Retry Loop

```python
def extract_invoice_with_retry(
    invoice_text: str,
    max_retries: int = 3
) -> tuple[InvoiceData, bool]:
    """Extract with validation-retry loop
    
    Returns: (extraction, success)
    """
    
    for attempt in range(max_retries):
        extraction = extract_invoice_single_pass(invoice_text)
        
        # Validate
        issues = validate_extraction(extraction, invoice_text)
        
        if not issues:
            return extraction, True  # Success
        
        # If validation fails and we have retries left, retry with error context
        if attempt < max_retries - 1:
            print(f"Attempt {attempt + 1} failed. Issues: {issues}")
            
            # RETRY with error feedback
            response = claude.messages.create(
                model="claude-opus-4-8",
                max_tokens=1024,
                tools=[create_extraction_tool()],
                tool_choice={"type": "tool", "name": "extract_invoice"},
                messages=[
                    {
                        "role": "user",
                        "content": f"""
Extract invoice data again. Fix these validation errors:
{chr(10).join(f'- {issue}' for issue in issues)}

Original document:
{invoice_text}

Previous attempt failed. Please carefully re-extract and ensure:
1. Line items sum correctly to subtotal
2. Subtotal + tax = total
3. All required fields present
4. Confidence scores are accurate
                        """
                    }
                ]
            )
            
            extraction = next(
                (block.input for block in response.content if block.type == "tool_use"),
                None
            )
        else:
            # Max retries reached
            extraction.validation_issues = issues
            return extraction, False
    
    return extraction, False
```

---

## Part 3: Add Few-Shot Examples for Format Variety

### Step 3.1: Few-Shot Prompting for Varied Document Formats

Create a prompt that demonstrates extraction from different formats:

```python
EXTRACTION_SYSTEM_PROMPT = """You are an invoice data extraction expert. Extract structured data from invoices in various formats.

EXAMPLES OF VARIED FORMATS YOU MAY ENCOUNTER:

Format 1: Traditional tabular layout
---
INVOICE INV-2024-001
Date: January 15, 2024
Bill To: Acme Corp

Items:
Description | Qty | Unit Price | Total
Widget A    |  10 | $5.00      | $50.00
Widget B    |   5 | $8.00      | $40.00

Subtotal: $90.00
Tax (10%): $9.00
Total: $99.00

Status: PAID
---

Expected output:
{
  "invoice_number": "INV-2024-001",
  "invoice_date": "2024-01-15",
  "line_items": [
    {"description": "Widget A", "quantity": 10, "unit_price": 5.00, "total": 50.00},
    {"description": "Widget B", "quantity": 5, "unit_price": 8.00, "total": 40.00}
  ],
  "subtotal": 90.00,
  "tax_amount": 9.00,
  "total_amount": 99.00,
  "payment_status": "paid",
  "confidence_scores": {
    "overall": 0.95,
    "invoice_number": 1.0,
    "total_amount": 1.0
  }
}

Format 2: Prose narrative (invoice embedded in email)
---
Hi, attached is the invoice for work completed. We're billing $1,000 for 40 hours of development at $25/hour. Tax is included in the total. Invoice #E-2024-042, dated today (Jan 20, 2024). Payment is due within 30 days.
---

Expected output:
{
  "invoice_number": "E-2024-042",
  "invoice_date": "2024-01-20",
  "line_items": [
    {"description": "Development services (40 hours)", "quantity": 40, "unit_price": 25.00, "total": 1000.00}
  ],
  "subtotal": 1000.00,
  "tax_amount": null,
  "total_amount": 1000.00,
  "payment_terms": "Net 30",
  "payment_status": "unpaid",
  "confidence_scores": {
    "overall": 0.85,
    "invoice_number": 1.0,
    "total_amount": 0.9
  },
  "validation_issues": ["Tax inclusion not clearly stated; assumed included"]
}

KEY EXTRACTION RULES:
1. Only extract information present in document (null if missing)
2. Use confidence_scores to indicate certainty (0.0-1.0)
3. Flag mathematical inconsistencies in validation_issues
4. Use "unclear" payment_status only when truly ambiguous
5. Preserve original formatting clues (e.g., "Net 30" for payment terms)
"""
```

---

## Part 4: Design Batch Processing Strategy

### Step 4.1: Batch Submission

```python
from anthropic import Anthropic

def prepare_batch_requests(invoices: List[dict]) -> List[dict]:
    """Prepare batch requests for Message Batches API"""
    
    requests = []
    for i, invoice in enumerate(invoices):
        requests.append({
            "custom_id": f"invoice_{invoice['id']}",
            "params": {
                "model": "claude-opus-4-8",
                "max_tokens": 1024,
                "tools": [create_extraction_tool()],
                "tool_choice": {"type": "tool", "name": "extract_invoice"},
                "messages": [
                    {
                        "role": "user",
                        "content": f"Extract invoice data:\n\n{invoice['text']}"
                    }
                ]
            }
        })
    
    return requests

def submit_batch(requests: List[dict]) -> str:
    """Submit batch to Message Batches API"""
    
    client = Anthropic()
    
    batch = client.beta.messages.batches.create(
        requests=requests
    )
    
    print(f"Batch {batch.id} submitted. Status: {batch.processing_status}")
    return batch.id

def poll_batch_status(batch_id: str, poll_interval_seconds: int = 30) -> dict:
    """Poll batch until complete"""
    
    client = Anthropic()
    import time
    
    while True:
        batch = client.beta.messages.batches.retrieve(batch_id)
        
        if batch.processing_status == "processing":
            print(f"Batch {batch_id}: {batch.request_counts.processed}/{batch.request_counts.total} complete")
            time.sleep(poll_interval_seconds)
        else:
            return batch
```

### Step 4.2: Handle Batch Failures

```python
def process_batch_results(batch_id: str, invoices: List[dict]) -> tuple[List, List]:
    """Process batch results, identify failures"""
    
    client = Anthropic()
    
    successful = []
    failed = []
    
    batch = client.beta.messages.batches.retrieve(batch_id)
    
    if batch.processing_status == "failed":
        print(f"Batch {batch_id} failed: {batch.error_message}")
        return [], invoices
    
    # Retrieve results
    results = client.beta.messages.batches.results(batch_id)
    
    processed_ids = set()
    
    for result in results:
        custom_id = result.custom_id
        processed_ids.add(custom_id.replace("invoice_", ""))
        
        if result.result.type == "succeeded":
            extraction = extract_tool_output(result.result.message)
            successful.append({
                "invoice_id": custom_id.replace("invoice_", ""),
                "extraction": extraction
            })
        elif result.result.type == "expired":
            failed.append({
                "invoice_id": custom_id.replace("invoice_", ""),
                "reason": "Request expired (timeout)"
            })
        elif result.result.type == "errored":
            failed.append({
                "invoice_id": custom_id.replace("invoice_", ""),
                "reason": result.result.error.message
            })
    
    # Re-submit failed invoices with modifications (chunking, etc.)
    if failed:
        print(f"Batch {batch_id}: {len(failed)} failures to reprocess")
        # Mark for individual retry or re-batch with chunked content
    
    return successful, failed
```

---

## Part 5: Implement Human Review Routing

### Step 5.1: Confidence-Based Routing

```python
def route_for_review(
    extraction: InvoiceData,
    confidence_threshold: float = 0.80
) -> str:
    """Route extraction to human review based on confidence"""
    
    overall_confidence = extraction.confidence_scores.get("overall", 0.5)
    
    if overall_confidence >= confidence_threshold:
        return "auto_approve"
    elif overall_confidence >= 0.60:
        return "spot_check"  # Random sampling verification
    else:
        return "human_review"  # Full review required
```

### Step 5.2: Field-Level Confidence Calibration

```python
def calibrate_confidence_thresholds(
    validation_set: List[dict],  # {extraction, human_verified}
    model: str = "logistic_regression"
) -> dict:
    """Calibrate confidence thresholds using labeled validation set"""
    
    from sklearn.linear_model import LogisticRegression
    import numpy as np
    
    # Build training data
    X = []  # Confidence scores
    y = []  # Correctness (0=wrong, 1=correct)
    
    for item in validation_set:
        extraction = item["extraction"]
        verified = item["human_verified"]
        
        # Was extraction correct?
        correct = (extraction == verified)
        
        # Features: overall confidence
        confidence = extraction.confidence_scores.get("overall", 0.5)
        X.append([confidence])
        y.append(1 if correct else 0)
    
    # Train calibration model
    clf = LogisticRegression()
    clf.fit(np.array(X), np.array(y))
    
    # Determine thresholds
    thresholds = {}
    for threshold in [0.60, 0.80, 0.95]:
        # What confidence level gives us this accuracy?
        confidence_needed = clf.predict_proba([[threshold]])[0][1]
        thresholds[f"confidence_{int(threshold*100)}_percent"] = confidence_needed
    
    return thresholds

def analyze_accuracy_by_field(validation_set: List[dict]) -> dict:
    """Measure accuracy by document type and field"""
    
    accuracy = {
        "by_field": {},
        "by_document_type": {}
    }
    
    for item in validation_set:
        extraction = item["extraction"]
        verified = item["human_verified"]
        doc_type = item.get("document_type", "unknown")
        
        # Field-level accuracy
        for field in ["invoice_number", "total_amount", "line_items"]:
            if field not in accuracy["by_field"]:
                accuracy["by_field"][field] = {"correct": 0, "total": 0}
            
            accuracy["by_field"][field]["total"] += 1
            if getattr(extraction, field) == getattr(verified, field):
                accuracy["by_field"][field]["correct"] += 1
        
        # Document type accuracy
        if doc_type not in accuracy["by_document_type"]:
            accuracy["by_document_type"][doc_type] = {"correct": 0, "total": 0}
        
        accuracy["by_document_type"][doc_type]["total"] += 1
        if extraction == verified:
            accuracy["by_document_type"][doc_type]["correct"] += 1
    
    # Calculate percentages
    for field_acc in accuracy["by_field"].values():
        field_acc["accuracy"] = field_acc["correct"] / field_acc["total"]
    
    for doc_acc in accuracy["by_document_type"].values():
        doc_acc["accuracy"] = doc_acc["correct"] / doc_acc["total"]
    
    return accuracy
```

---

## Part 6: End-to-End Pipeline

### Step 6.1: Complete Pipeline

```python
def process_invoices_pipeline(
    invoices: List[dict],
    use_batch: bool = False,
    confidence_threshold: float = 0.80
) -> dict:
    """Complete extraction pipeline"""
    
    results = {
        "auto_approved": [],
        "spot_check": [],
        "human_review": [],
        "failed": []
    }
    
    if use_batch:
        # Batch processing for large volumes
        requests = prepare_batch_requests(invoices)
        batch_id = submit_batch(requests)
        batch = poll_batch_status(batch_id)
        successful, failed = process_batch_results(batch_id, invoices)
        
        for item in successful:
            extraction = item["extraction"]
            route = route_for_review(extraction, confidence_threshold)
            results[route].append(extraction)
        
        results["failed"] = failed
    else:
        # Individual processing with retry
        for invoice in invoices:
            try:
                extraction, success = extract_invoice_with_retry(
                    invoice["text"],
                    max_retries=3
                )
                
                if success:
                    route = route_for_review(extraction, confidence_threshold)
                    results[route].append(extraction)
                else:
                    results["human_review"].append(extraction)  # Still send for review
            
            except Exception as e:
                results["failed"].append({
                    "invoice_id": invoice["id"],
                    "error": str(e)
                })
    
    return results
```

---

## Validation Checklist

### Schema Design ✅
- [ ] Required vs optional fields clearly defined
- [ ] Nullable fields for information that might not exist
- [ ] Enum with "other" option for extensible categories
- [ ] Confidence scores for each extraction field

### Tool Use ✅
- [ ] Tool_use enforced (tool_choice: "any" or forced)
- [ ] Strict schema eliminates JSON syntax errors
- [ ] Optional fields prevent hallucination (null instead of fabrication)

### Validation-Retry Loop ✅
- [ ] Single-pass extraction uses tool_use
- [ ] Validation checks math (line items sum) and consistency
- [ ] Retries include specific error feedback from validation
- [ ] Max 3 retries to avoid infinite loops

### Few-Shot Examples ✅
- [ ] Examples demonstrate varied document formats
- [ ] Examples show correct handling of null fields
- [ ] Examples show how to handle ambiguous cases

### Batch Processing ✅
- [ ] Batch requests use custom_id for correlation
- [ ] Polling handles latency (up to 24 hours)
- [ ] Failed requests identified and resubmitted

### Human Review Routing ✅
- [ ] Confidence thresholds calibrated on validation set
- [ ] Accuracy measured by field and document type
- [ ] Low-confidence extractions routed to review
- [ ] Spot-check sampling for medium-confidence

---

## 🎯 Success Criteria

Your pipeline is complete when:

1. **Schema** clearly defines required, optional, nullable, enum fields
2. **Extraction** uses tool_use; validation-retry loop works with error feedback
3. **Few-shot examples** cover varied document formats
4. **Batch processing** handles 100+ documents with failure resubmission
5. **Confidence calibration** achieved >80% accuracy on validation set
6. **Human review routing** based on field-level confidence scores
7. **Accuracy analysis** shows consistent performance by field and document type

---

## 📚 Related Study Materials

- D4-03: Tool Use with JSON Schemas
- D4-04: Validation-Retry Loops
- D4-02: Few-Shot Prompting Techniques
- D4-05: Batch Processing Patterns
- D5-05: Human Review & Synthesis Strategies

---

**Prepared:** July 15, 2026  
**Difficulty:** High  
**Estimated Time:** 3-4 hours  
**Status:** Ready for implementation
