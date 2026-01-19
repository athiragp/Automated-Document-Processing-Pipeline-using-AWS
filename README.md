# Automated Document Processing Pipeline using AWS

## 1. Project Overview

This project implements a **fully automated, event-driven document processing pipeline** on AWS. Any document uploaded to a designated S3 input bucket (from local PC automation or manual upload) is automatically:

1. OCR-processed using **Amazon Textract**
2. Cleaned and normalized using **Amazon Bedrock (LLM)**
3. Classified and structured (ID type, name, address, DOB, etc.) using **Amazon Bedrock**
4. Persisted into **Amazon DynamoDB**
5. Archived into a **processed S3 bucket**

The entire system is **hands-free**, scalable, and serverless.

---

## 2. High-Level Architecture

### AWS Services Used

* **Amazon S3 (Input Bucket)** – Document upload trigger
* **Amazon S3 (Processed Bucket)** – Stores processed documents
* **AWS Lambda** – Core processing logic
* **Amazon Textract** – OCR for images and PDFs
* **Amazon Bedrock** – Text cleaning + document classification
* **Amazon DynamoDB** – Structured metadata storage
* **Amazon EventBridge + macOS cron + AWS CLI** – Automated local uploads
* **Amazon CloudWatch Logs** – Observability and debugging

---

## 3. Buckets and Data Stores (from project memory)

| Resource                 | Purpose                                 |
| ------------------------ | --------------------------------------- |
| **docprocessing bucket** | Input bucket for raw documents          |
| **processed bucket**     | Stores successfully processed documents |
| **DynamoDB table**       | Stores extracted metadata per document  |

---

## 4. End-to-End Architectural Flow

### Step 1: Local File Creation (Mac)

* User saves a document (PNG / JPG / PDF) into a local folder:

  ```
  /Users/<username>/<foldername>
  ```

* A **cron job** periodically scans this folder.

---

### Step 2: Automated Upload to S3

* AWS CLI command executed via cron:

  ```bash
  aws s3 sync ~/<foldername> s3://docprocessing-bucket
  ```

* Only **new or modified files** are uploaded.

* No duplicates are re-uploaded.

---

### Step 3: S3 Event Trigger

* The input bucket has an **ObjectCreated** event notification.
* Any new object upload triggers the Lambda function automatically.

---

### Step 4: Lambda Invocation

The Lambda function performs the following:

1. Reads S3 event metadata
2. Downloads the file into `/tmp`
3. Detects file type (`png`, `jpg`, `jpeg`, `pdf`)

---

### Step 5: OCR with Amazon Textract

#### For Images (PNG / JPG / JPEG)

* Uses:

  ```python
  textract.detect_document_text()
  ```

#### For PDFs

* Uses asynchronous processing:

  ```python
  start_document_text_detection()
  get_document_text_detection()
  ```

#### Output

* Extracted raw text
* Average OCR confidence score

---

### Step 6: Text Cleaning with Amazon Bedrock

* Raw OCR text is sent to Bedrock
* Prompt instructs the LLM to:

  * Remove OCR noise
  * Preserve names, IDs, dates, addresses
  * Return **only cleaned text**

This ensures higher accuracy for downstream extraction.

---

### Step 7: Document Classification & Field Extraction (Bedrock)

Bedrock is used again to extract structured fields:

* `customer_name`
* `doc_type` (Aadhaar, PAN, Passport, etc.)
* `id_number`
* `address`
* `dob`

Rules:

* Missing fields → `NA`
* Output format → **strict JSON**

This ensures DynamoDB-safe ingestion.

---

### Step 8: Persisting to DynamoDB

Each document produces one DynamoDB item:

| Attribute       | Description              |
| --------------- | ------------------------ |
| `doc_id`        | UUID (partition key)     |
| `customer_name` | Extracted name           |
| `doc_type`      | Classified document type |
| `id_number`     | ID number                |
| `address`       | Extracted address        |
| `dob`           | Date of birth            |
| `confidence`    | OCR confidence (Decimal) |
| `status`        | PROCESSED                |
| `timestamp`     | Epoch time               |

> Note: Attribute order in DynamoDB console is alphabetical and not configurable.

---

### Step 9: Archiving to Processed Bucket

* Original document is copied to:

  ```
  s3://processed-bucket/processed/<filename>
  ```

* Ensures:

  * No reprocessing
  * Clear separation of raw vs processed data

---

## 5. Automation Design (Why EventBridge Was Not Used)

### Requirement

> *Sweep a local Mac folder at intervals and upload new files automatically*

### Final Choice

* **macOS cron + AWS CLI sync**

### Why this was the right choice

| Option         | Reason                               |
| -------------- | ------------------------------------ |
| EventBridge    | Cannot access local filesystem       |
| DataSync       | Heavy, agent-based, enterprise-grade |
| Lambda         | Cannot read local folders            |
| **Cron + CLI** | Lightweight, reliable, cost-free ✅   |

EventBridge is useful **inside AWS**, not for local machines.

---

## 6. Security Model

* IAM User with programmatic access
* Least-privilege permissions:

  * S3 upload
  * DynamoDB write
  * Textract
  * Bedrock invoke
* No credentials hardcoded in code

---

## 7. Observability & Debugging

* **CloudWatch Logs** capture:

  * Lambda execution logs
  * Errors during OCR / Bedrock / DynamoDB

* S3 upload failures are caught at CLI level

---

## 8. Supported File Types

| Type | Supported | Notes               |
| ---- | --------- | ------------------- |
| PNG  | ✅         | Direct Textract OCR |
| JPG  | ✅         | Direct Textract OCR |
| JPEG | ✅         | Direct Textract OCR |
| PDF  | ✅         | Async Textract job  |
| DOCX | ❌         | Not supported       |

---

## 9. Key Design Principles

* Serverless-first
* Event-driven
* Idempotent uploads
* Separation of concerns
* LLM-based intelligence
* Cost-efficient

---

## 10. Final Outcome

The system now:

* Requires **zero manual intervention**
* Automatically processes documents end-to-end
* Scales with demand
* Produces structured, queryable data

This architecture is production-ready and extensible.

---




