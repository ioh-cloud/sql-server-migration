# Restore SQL Server Transaction Log Files to a Cloud SQL Instance

This repository provides a Python-based solution for restoring transaction log backups stored in a cloud bucket to a database hosted on a Cloud SQL for SQL Server instance.

---

## Overview

### Restore Capability

Starting October 2023, Cloud SQL for SQL Server supports importing transaction log backups. This feature is particularly beneficial for migrations to Cloud SQL using backup files or for setting up disaster recovery (DR) environments for SQL Server instances.

For details, refer to the [official documentation](https://cloud.google.com/sql/docs/sqlserver/import-export/import-export-bak#import_transaction_log_backups).

---

### Workflow Summary

![Workflow Diagram](tlog-restore-cloudsql.png)

1. **Trigger**: Transaction log backup files are uploaded to a cloud bucket, either from a standalone SQL Server instance or another Cloud SQL instance.
2. **Function Execution**: An EventArc trigger initiates a Python function that processes the uploaded files and sends requests to restore them to the Cloud SQL for SQL Server instance.
3. **Restore Progress Monitoring**: The function tracks the status of the restore operation, handling errors and retrying as necessary.
4. **Post-Processing**:
   - On success: The backup file is moved to a "processed" bucket (if defined) and deleted from the source bucket.
   - On specific SQL errors (e.g., 4326 or 4305): Appropriate actions are taken, such as deletion of files already applied or retrying for synchronization issues.
   - On other errors: The function relies on automatic retries.

---

## Features and Functionality

### Backup Types Supported:
- Full backups
- Differential backups
- Transaction log backups

### Restoration Options:
- **Recovery Mode**: Restores with `_recovery` for accessibility.
- **No Recovery Mode** (default): Leaves the database ready for additional restores.

### Backup Information Source:
1. **File Name Format**:
   - Requires a specific file naming convention to derive metadata such as instance name, database name, backup type, and recovery status.
   - Example pattern: `<instance-name>_<database-name>_<backup-type>_<recovery-status>_*.*`

2. **Object Metadata**:
   - Metadata tags provide necessary information for the function (e.g., `CloudSqlInstance`, `DatabaseName`, `BackupType`, `Recovery`).

---

## Prerequisites

### Tools and Software
- **Cloud Shell** or [gcloud CLI](https://cloud.google.com/sdk/gcloud#download_and_install_the)
- **PowerShell** 5.1 or 7.X
- [Google Cloud PowerShell Module](https://cloud.google.com/tools/powershell/docs/quickstart)

### Permissions Required
The function and setup scripts require various IAM roles, including:
- `roles/cloudfunctions.developer`
- `roles/cloudsql.editor`
- `roles/storage.admin`
- `roles/iam.roleAdmin`
- `roles/iam.serviceAccountCreator`

For a detailed list of permissions, see the "Setup and Configuration" section.

---

## Setup and Configuration

### Cloud Function Deployment

1. **Create a GCS Bucket** for storing transaction log backups:
   ```bash
   gcloud storage buckets create gs://<BUCKET_NAME> \
       --location=<BUCKET_LOCATION> \
       --public-access-prevention
   ```

2. **Grant Necessary Permissions**:
   Assign `roles/storage.legacyBucketReader` to the Cloud SQL service account:
   ```bash
   gcloud projects add-iam-policy-binding ${PROJECT_ID} \
       --member="serviceAccount:<service-account-email>" \
       --role="roles/storage.legacyBucketReader"
   ```

3. **Create a Cloud Function Service Account**:
   ```bash
   gcloud iam service-accounts create cloud-function-sql-restore-log \
       --display-name "Service Account for Cloud Function"
   ```

4. **Assign Roles to the Service Account**:
   Create and assign a custom `cloud.sql.importer` role:
   ```bash
   gcloud iam roles create cloud.sql.importer \
       --project ${PROJECT_ID} \
       --permissions "cloudsql.instances.import,storage.objects.get"
   ```

5. **Deploy the Function**:
   ```bash
   gcloud functions deploy <FUNCTION_NAME> \
       --gen2 \
       --region=<REGION> \
       --runtime=python312 \
       --entry-point=fn_restore_log \
       --trigger-bucket=<BUCKET_NAME> \
       --service-account cloud-function-sql-restore-log@${PROJECT_ID}.iam.gserviceaccount.com
   ```

6. **Set Invoker Permissions**:
   ```bash
   gcloud functions add-invoker-policy-binding <FUNCTION_NAME> \
       --member="serviceAccount:cloud-function-sql-restore-log@${PROJECT_ID}.iam.gserviceaccount.com"
   ```

---

### Upload Script Configuration

The `scheduled-upload` folder contains a PowerShell script (`upload-script.ps1`) for automating backup uploads to GCS.

Steps:
1. **Create a Service Account**:
   ```bash
   gcloud iam service-accounts create tx-log-backup-writer \
       --description="Service Account for Backup Uploads"
   ```

2. **Assign Storage Permissions**:
   ```bash
   gcloud storage buckets add-iam-policy-binding gs://<BUCKET_NAME> \
       --member="serviceAccount:tx-log-backup-writer@${PROJECT_ID}.iam.gserviceaccount.com" \
       --role="roles/storage.objectAdmin"
   ```

3. **Generate a Service Account Key**:
   ```bash
   gcloud iam service-accounts keys create KEY_FILE.json \
       --iam-account tx-log-backup-writer@${PROJECT_ID}.iam.gserviceaccount.com
   ```

4. **Configure the PowerShell Script**:
   Update constants in the script with bucket name, key file path, and other details.

---

## Testing

### Cloud Function Tests
Unit tests for the function are located in the `Function` folder (`main_test.py`). Run tests using:
```bash
pytest
```

### Upload Script Tests
Unit tests for the PowerShell script are located in the `scheduled-upload` folder (`upload-script.Tests.ps1`). Run tests with:
```powershell
Invoke-Pester
```

---

## References
- [Cloud Functions Documentation](https://cloud.google.com/functions/docs)
- [Cloud SQL for SQL Server Backups](https://cloud.google.com/sql/docs/sqlserver/import-export/import-export-bak)