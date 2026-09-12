# File Metadata Extractor

A fully serverless web application on AWS that extracts embedded metadata from uploaded files — PDF, DOCX, PNG, JPG, and TXT — without ever storing the file permanently.

**Live demo:** http://metadata-extractor-site-deeksha081382.s3-website.ap-south-1.amazonaws.com

> Note: served over plain HTTP via S3 static website hosting (no CloudFront/HTTPS layer was added for this project).

---

## What it does

Upload a file in the browser and get back its embedded metadata in seconds:

- **PDF** — page count, author, title, creation date, producer
- **DOCX** — author, title, word count, paragraph count, last modified by
- **PNG / JPG** — dimensions, color mode, EXIF data (camera make/model, capture date, etc.)
- **TXT** — character count, word count, line count, encoding

Results can be exported as JSON, CSV, or plain text.

## Architecture

```
Browser --POST--> API Gateway --invoke--> Lambda
                                              |
                                   write file to S3
                                   extract metadata
                                   delete file from S3
                                              |
Browser <--JSON response-- API Gateway <-----+
```

The browser reads the file locally, base64-encodes it, and sends it in a single HTTPS request. Lambda writes a copy to S3 (satisfying a genuine "temporary storage" requirement), extracts metadata directly from the in-memory bytes, and deletes the S3 object — all within one synchronous execution. Every invocation is logged to CloudWatch automatically.

## AWS services used

| Service | Role |
|---|---|
| **Amazon S3** | Transient file storage during processing (auto-deleted immediately, with a 1-day lifecycle rule as a backup safety net) |
| **AWS Lambda** | Runs the Python 3.12 extraction logic (`pypdf`, `python-docx`, `Pillow`, bundled via a Lambda Layer) |
| **Amazon API Gateway** | Exposes Lambda as a public HTTPS endpoint (HTTP API, CORS-enabled) |
| **AWS IAM** | Least-privilege execution role — Lambda can only `GetObject`/`PutObject`/`DeleteObject` on one specific bucket |
| **Amazon CloudWatch** | Automatic logging of every invocation — duration, memory, errors |

## Tech stack

- **Backend:** Python 3.12 (AWS Lambda)
- **Frontend:** Vanilla HTML/CSS/JavaScript — no framework or build step
- **Infrastructure:** AWS (S3, Lambda, API Gateway, IAM, CloudWatch), configured via the AWS Console and CLI

## Repository structure

```
├── lambda_function.py       # Lambda handler + per-file-type extraction logic
├── index.html                # Frontend (upload UI, calls the API Gateway endpoint)
├── generate_test_files.py    # Helper script to generate local test PDF/DOCX files
└── README.md
```

## Notable engineering challenges

- **Cross-platform binary compatibility:** built the Lambda Layer on Windows targeting Linux (`--platform manylinux2014_x86_64`), since Pillow ships compiled binaries that must match Lambda's Amazon Linux runtime.
- **Zip path separators:** Windows' built-in `tar` produced backslash-separated paths inside the layer zip, which Lambda couldn't resolve — fixed by rebuilding with PowerShell's `Compress-Archive`.
- **Architecture matching:** Lambda function architecture (x86_64) must match the layer's compiled binaries exactly, or imports fail silently at runtime.
- **Defense-in-depth cleanup:** a `try/finally` block deletes the S3 object after processing, backed by an S3 Lifecycle rule (1-day auto-expiry) in case a hard timeout or crash prevents the code from running its own cleanup.

## Cost

Built entirely within AWS Free Tier / promotional credits. Verified **$0.00** total spend throughout development and testing.

---

Built as a Cloud Computing course project, structured to double as a portfolio piece.
