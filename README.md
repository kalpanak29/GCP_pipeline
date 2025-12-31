# 🚀 GCP Student Data Pipeline

An automated data pipeline built on Google Cloud Platform (GCP) that cleans dirty Excel/CSV files and loads them to BigQuery with duplicate detection and email notifications.

> **Originally built on Azure Data Factory (ADF) and migrated to GCP**

---

## 📋 Table of Contents

- [Overview](#overview)
- [Features](#features)
- [Architecture](#architecture)
- [Components](#components)
- [Workflow](#workflow)
- [Data Cleaning Rules](#data-cleaning-rules)
- [Decision Logic](#decision-logic)
- [Email Notifications](#email-notifications)
- [Prerequisites](#prerequisites)
- [Setup Instructions](#setup-instructions)
- [Usage](#usage)
- [Commands Reference](#commands-reference)
- [Cost Estimation](#cost-estimation)
- [Azure to GCP Mapping](#azure-to-gcp-mapping)
- [Project Structure](#project-structure)
- [Troubleshooting](#troubleshooting)
- [License](#license)

---

## 🎯 Overview

This pipeline automatically:
- Detects new file uploads to Cloud Storage
- Cleans dirty data (removes special characters, fixes phone numbers, standardizes dates)
- Loads cleaned data to BigQuery
- Archives original files
- Tracks processing history to prevent duplicates
- Sends email notifications on success, skip, or failure

---

## ✨ Features

| Feature | Description |
|---------|-------------|
| **Automatic Trigger** | Pipeline runs automatically when file is uploaded to GCS |
| **Duplicate Detection** | Tracks files in log table; skips unchanged files |
| **Data Cleaning** | Removes special characters, fixes phones, standardizes dates |
| **Data Archiving** | Original files archived with timestamp |
| **Email Notifications** | Get notified on SUCCESS, SKIP, or FAILURE |
| **State Management** | Full audit trail in BigQuery |
| **Free Trial Compatible** | Works within GCP $300 free credits |

---

## 🏗️ Architecture
USER
│
│ Upload Excel/CSV
▼
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│ │ │ │ │ │
│ GCS BUCKET │────▶│ CLOUD FUNCTION │────▶│ BIGQUERY │
│ (Raw Incoming) │ │ (Python) │ │ (students) │
│ │ │ │ │ │
└─────────────────┘ └────────┬────────┘ └─────────────────┘
│
┌────────────┼────────────┐
▼ ▼ ▼
┌───────────┐ ┌───────────┐ ┌───────────┐
│ BIGQUERY │ │ GCS │ │ EMAIL │
│ (Log) │ │ (Archive) │ │ SERVICE │
└───────────┘ └───────────┘ └───────────┘

text


---

## 🧩 Components

### GCP Services Used

| Service | Resource Name | Purpose |
|---------|--------------|---------|
| Cloud Storage | `{project}-raw-incoming` | Store incoming files |
| Cloud Storage | `{project}-staging-clean` | Store cleaned files (temp) |
| Cloud Storage | `{project}-archive` | Store archived originals |
| BigQuery | `student_data.students` | Clean data table |
| BigQuery | `student_data.file_load_log` | Processing history |
| Cloud Functions | `student-data-pipeline` | Python processing code |
| Eventarc | GCS Trigger | Auto-trigger on upload |
| Secret Manager | `sendgrid-api-key` | Email API credentials |

---

## 📊 Workflow

The pipeline executes 11 steps:

1. **Upload File** - User uploads `Students.xlsx` to GCS bucket
2. **Trigger** - Eventarc detects file and triggers Cloud Function
3. **Get Metadata** - Read file's last modified timestamp
4. **Lookup Log** - Query BigQuery: was file processed before?
5. **Compare Timestamps** - Determine: First load? Modified? Not modified?
6. **Clean Data** - Remove special characters, fix phones, format dates
7. **Write to Staging** - Save cleaned CSV to staging bucket
8. **Load to BigQuery** - Insert rows into students table
9. **Archive & Cleanup** - Move original to archive, delete staging
10. **Update Log** - Record success/skip in `file_load_log`
11. **Send Email** - Notify user of result via email

---

## 🧹 Data Cleaning Rules

| Field | Input Example | Output Example | Cleaning Applied |
|-------|---------------|----------------|------------------|
| `StudentName` | `John_Doe!!!` | `John Doe` | Remove `# ! @ $ % ^ & * _` |
| `Address` | `123 Main_St##` | `123 Main St` | Remove special chars, replace `_` with space |
| `Phone` | `9.87654321E9` | `9876543210` | Handle scientific notation, extract 10 digits |
| `Phone` | `(555) 123-4567` | `5551234567` | Remove all non-digits |
| `Admission_date` | `March 5, 2024` | `2024-03-05` | Parse various formats to `YYYY-MM-DD` |
| `Admission_date` | `01/20/2024` | `2024-01-20` | Standardize date format |

---

## 🔀 Decision Logic

### Scenario 1: First Time Upload
File: Students_Oct.xlsx
In Log: NO
Decision: ✅ PROCESS
Reason: FIRST_LOAD

text


### Scenario 2: Re-upload Same File (No Changes)
File: Students_Oct.xlsx
In Log: YES
File Timestamp: 10:30:00
Log Timestamp: 10:30:00
Decision: ⏭️ SKIP
Reason: NOT_MODIFIED

text


### Scenario 3: Re-upload Modified File
File: Students_Oct.xlsx
In Log: YES
File Timestamp: 11:45:00 (newer)
Log Timestamp: 10:30:00 (older)
Decision: ✅ PROCESS
Reason: FILE_MODIFIED

text


---

## 📧 Email Notifications

| Status | Subject | When Sent |
|--------|---------|-----------|
| ✅ SUCCESS | `Pipeline SUCCESS: filename` | File processed & loaded to BigQuery |
| ⏭️ SKIPPED | `Pipeline SKIPPED: filename` | File not modified since last run |
| ❌ FAILED | `Pipeline FAILED: filename` | Error during processing |

### Email Content Includes:
- File name and status
- Rows processed (for SUCCESS)
- Archive location
- Timestamp
- Error message (for FAILED)

---

## 📋 Prerequisites

- Google Cloud Platform account (Free Trial works)
- $300 free credits (new accounts)
- SendGrid account (free) OR Gmail with App Password
- Basic command line knowledge

---

## 🛠️ Setup Instructions

### Step 1: Create GCP Project
```bash
gcloud projects create student-pipeline --name="Student Pipeline"
gcloud config set project student-pipeline
Step 2: Enable APIs
Bash

gcloud services enable storage.googleapis.com
gcloud services enable cloudfunctions.googleapis.com
gcloud services enable cloudbuild.googleapis.com
gcloud services enable bigquery.googleapis.com
gcloud services enable eventarc.googleapis.com
gcloud services enable run.googleapis.com
gcloud services enable secretmanager.googleapis.com
Step 3: Create Storage Buckets
Bash

export PROJECT_ID=$(gcloud config get-value project)

gsutil mb -l us-central1 gs://${PROJECT_ID}-raw-incoming
gsutil mb -l us-central1 gs://${PROJECT_ID}-staging-clean
gsutil mb -l us-central1 gs://${PROJECT_ID}-archive
Step 4: Create BigQuery Tables
Bash

bq mk --location=us-central1 --dataset ${PROJECT_ID}:student_data

bq query --use_legacy_sql=false "
CREATE TABLE IF NOT EXISTS student_data.students (
    StudentID INT64,
    StudentName STRING,
    Address STRING,
    Phone STRING,
    Admission_date DATE,
    load_timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP(),
    source_file STRING
)"

bq query --use_legacy_sql=false "
CREATE TABLE IF NOT EXISTS student_data.file_load_log (
    log_id STRING NOT NULL,
    file_name STRING NOT NULL,
    file_path STRING NOT NULL,
    last_modified_timestamp TIMESTAMP NOT NULL,
    file_size_bytes INT64,
    rows_processed INT64,
    load_status STRING,
    staging_file_path STRING,
    archive_file_path STRING,
    error_message STRING,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP(),
    processed_by STRING
)"
Step 5: Store Email Credentials
Bash

# For SendGrid
echo -n "SG.your-api-key" | gcloud secrets create sendgrid-api-key \
    --replication-policy="automatic" --data-file=-

# Grant access
export PROJECT_NUMBER=$(gcloud projects describe $PROJECT_ID --format='value(projectNumber)')
gcloud secrets add-iam-policy-binding sendgrid-api-key \
    --member="serviceAccount:${PROJECT_NUMBER}-compute@developer.gserviceaccount.com" \
    --role="roles/secretmanager.secretAccessor"
Step 6: Deploy Cloud Function
Bash

cd cloud-function

gcloud functions deploy student-data-pipeline \
    --gen2 \
    --runtime=python311 \
    --region=us-central1 \
    --source=. \
    --entry-point=gcs_trigger \
    --trigger-event-filters="type=google.cloud.storage.object.v1.finalized" \
    --trigger-event-filters="bucket=${PROJECT_ID}-raw-incoming" \
    --memory=512MB \
    --timeout=300s \
    --set-env-vars="GCP_PROJECT=${PROJECT_ID},NOTIFICATION_EMAIL=your@email.com,EMAIL_METHOD=sendgrid"
🚀 Usage
Upload a File (Triggers Pipeline)
Bash

gsutil cp Students_Oct.xlsx gs://${PROJECT_ID}-raw-incoming/incoming/
Check Pipeline Logs
Bash

gcloud functions logs read student-data-pipeline --region=us-central1 --limit=50
Query Processed Data
Bash

bq query --use_legacy_sql=false "SELECT * FROM student_data.students"
Check Processing History
Bash

bq query --use_legacy_sql=false "
SELECT file_name, load_status, rows_processed, created_at 
FROM student_data.file_load_log 
ORDER BY created_at DESC LIMIT 10"
📝 Commands Reference
Action  Command
Upload file  gsutil cp file.xlsx gs://${PROJECT_ID}-raw-incoming/incoming/
View logs  gcloud functions logs read student-data-pipeline --region=us-central1 --limit=50
Query students  bq query --use_legacy_sql=false "SELECT * FROM student_data.students"
Check file log  bq query --use_legacy_sql=false "SELECT * FROM student_data.file_load_log ORDER BY created_at DESC"
List archives  gsutil ls gs://${PROJECT_ID}-archive/archived/
Function status  gcloud functions describe student-data-pipeline --region=us-central1
💰 Cost Estimation
Service  Free Tier  Your Usage  Cost
Cloud Storage  5 GB free  ~1 MB  $0.00
Cloud Functions  2M calls free  ~50 calls  $0.00
BigQuery Storage  10 GB free  ~1 KB  $0.00
BigQuery Queries  1 TB free  ~1 MB  $0.00
Secret Manager  6 secrets free  2-3 secrets  $0.00
Total Estimated Cost: $0.00 - $0.50 for testing
Free Trial Credits: $300
Plenty of budget remaining!

🔄 Azure to GCP Mapping
Azure Component  GCP Equivalent
Azure Blob Storage  Google Cloud Storage (GCS)
Event Grid Trigger  Eventarc
Azure Function  Cloud Function
Azure SQL Database  BigQuery
ADF Get Metadata Activity  blob.reload() in Python
ADF Lookup Activity  BigQuery SQL Query
ADF If Condition  Python if/else
ADF Copy Activity  BigQuery Load Job
ADF Delete Activity  GCS blob.delete()
ADF Stored Procedure  BigQuery insert_rows_json()
ADF Web Activity  SendGrid/Gmail API
📁 Project Structure
text

gcp-data-pipeline/
├── cloud-function/
│   ├── main.py              # Cloud Function code
│   └── requirements.txt     # Python dependencies
├── scripts/
│   └── create_test_data.py  # Generate test files
├── docs/
│   └── architecture.md      # Architecture documentation
├── .gitignore
└── README.md
🔧 Troubleshooting
Function Not Triggering
Bash

# Check Eventarc triggers
gcloud eventarc triggers list --location=us-central1

# Redeploy function
gcloud functions deploy student-data-pipeline ...
Permission Denied Error
Bash

# Grant required permissions
export PROJECT_NUMBER=$(gcloud projects describe $PROJECT_ID --format='value(projectNumber)')

gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member="serviceAccount:${PROJECT_NUMBER}-compute@developer.gserviceaccount.com" \
    --role="roles/storage.admin"

gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member="serviceAccount:${PROJECT_NUMBER}-compute@developer.gserviceaccount.com" \
    --role="roles/bigquery.admin"
Email Not Sending
Bash

# Check secret exists
gcloud secrets list

# View function logs for email errors
gcloud functions logs read student-data-pipeline --region=us-central1 | grep -i email
✅ Success Criteria
Pipeline triggers automatically on file upload
Dirty data is cleaned (special characters, phones, dates)
Clean data loads to BigQuery
Original files are archived with timestamp
Duplicate files are detected and skipped
All processing is logged in file_load_log table
Email notifications sent for SUCCESS/SKIP/FAIL
Works within GCP Free Trial ($300 credits)
Mirrors Azure ADF pipeline functionality
📄 License
This project is licensed under the MIT License - see the LICENSE file for details.

👤 Author
Your Name
GitHub: @yourusername

🙏 Acknowledgments
Original pipeline built on Azure Data Factory
Migrated to GCP for learning and comparison purposes
Thanks to Google Cloud documentation and community
text


---

## Additional Files for GitHub

### .gitignore

```gitignore
# Python
__pycache__/
*.py[cod]
*$py.class
*.so
.Python
venv/
ENV/

# IDE
.vscode/
.idea/
*.swp
*.swo

# GCP
*.json
gcp-key.json
service-account.json

# Test files
*.xlsx
*.xls
*.csv

# OS
.DS_Store
Thumbs.db

# Environment
.env
.env.local
LICENSE
text

MIT License

Copyright (c) 2024 Your Name

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
Copy and paste these files directly into your GitHub repository. The README.md will render with proper formatting, tables, and code blocks on GitHub!
