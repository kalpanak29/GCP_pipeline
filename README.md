# 🚀 GCP Student Data Pipeline

An automated data pipeline built on Google Cloud Platform (GCP) that cleans dirty Excel/CSV files and loads them to BigQuery with duplicate detection and email notifications.

> **Originally built on Azure Data Factory (ADF) and migrated to GCP**

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
| **State Management** | Full audit trail in BigQuery |
| **Free Trial Compatible** | Works within GCP $300 free credits |

---

## 🏗️ Architecture
<img width="800" height="625" alt="image" src="https://github.com/user-attachments/assets/76209d2f-6c41-48e7-b88f-a070c0280364" />





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


