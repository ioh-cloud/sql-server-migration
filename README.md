# Restore SQL Server Transaction Log Files to a Cloud SQL for SQL Server instance

This repository contains the implementation of a Python function that restores transaction log backups uploaded to a cloud bucket to a database of an existing Cloud SQL for SQL Server instance.

## Overview

### Restore Capability

You can import transaction log backups in Cloud SQL for SQL Server since [October 2023](https://cloud.google.com/sql/docs/release-notes#October_17_2023). This functionality helps when migrating to Cloud SQL using backups or setting up Cloud SQL for SQL Server DR instances. 

For details, refer to the [official documentation](https://cloud.google.com/sql/docs/sqlserver/import-export/import-export-bak#import_transaction_log_backups).

---

### Main Workflow

![Transaction log backup restore to Cloud SQL for SQL Server](tlog-restore-cloudsql.png)

The process starts when SQL Server transaction log backup files are uploaded to a cloud bucket. These files may come from a standalone SQL Server instance or another Cloud SQL for SQL Server instance.

The upload event fires an EventArc trigger that calls the Python function. The function gets the path to the log file that was uploaded and constructs the request to restore the uploaded backup file to the Cloud SQL for SQL Server instance.

After the execution of the import request, the function checks periodically the progress of the restore operation. Once the status of the operation changes to "DONE", which means that it has an outcome, the function executes the following:

1. If the operation returns SUCCESS (determined by the absence of the 'error' element in the response JSON), then the function makes a copy of the backup file to the 'processed' storage bucket (if defined) and then finally deletes the file from the source storage bucket, thus signaling that the function processed the file successfully. It then returns an OK, finishing the execution.

2. If the outcome of the operation returns ERROR, then depending on the details of the error response inside, the function implements one of the following decisions:

    - If the import failed with SQL Server error 4326 - too early to apply to the database - then the function assumes that the log file was processed already and deletes it from the bucket. The function returns an OK, finishing the execution.

    - If the import failed with SQL Server error 4305 - too recent to apply to the database - the function assumes that there are some synchronization issues and returns a runtime error. It does not delete the log backup file from the source bucket and it relies on the retry mechanism - the same import is retried later (up to a maximum of 7 days) in a new function execution.
    
    - If the import fails for any other reason or if the function breaks mid-way, the function fails and returns a runtime error, relying on the retry mechanism that schedules a later attempt in a new execution. The file is not deleted from the source bucket.

The function needs certain information to make the proper request to restore the uploaded backup file to the Cloud SQL for SQL Server instance. This information includes:

- The Cloud SQL Instance name
- The database name
- The type of backup (full, differential, or transaction log backup)
- If the backup is restored with recovery or not (leaving the database ready to perform subsequent restores in case of no recovery used)

There are two ways in which the function gets this information: Either from the file name itself or from object metadata. To enable the function to use the file name functionality, set the `USE_FIXED_FILE_NAME_FORMAT` environment variable to "True". In this way, the function expects all the uploaded backup files to have a fixed name pattern from which it infers the needed information. More information below, in the Constraints section. We recommend using the option that is easier for you to implement (either changing backup file names or deciding the logic to persist object metadata).

The function can also restore full and differential backup files. To achieve this functionality, use the two options provided (fixed file name or object metadata to signal to the function that the backups are full, differential, or transaction log backups). In case of fixed file name, make sure that you have the substrings "_full" or "_diff" in the file name to trigger full respectively diff backup restores.

By default, the function restores backups with the norecovery option, leaving the database in a state to expect further sequential restores. Use the "_recovery" substring in the file name or set the Recovery tag to "True" in the object metadata. This is useful when switching to your DR Cloud SQL instance. In such cases, the function must restore a backup file with the recovery option set to true. This triggers the recovery option and leaves the database in an accessible state.

This repository also contains a PowerShell script for regularly uploading new files to cloud storage called `upload-script.ps1`, existing in the scheduled-upload folder. This provides an automated way of uploading the backup files to cloud storage.

The function must have a defined set of environment variables. Details about them are described below, in the constraints section.

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
The Function folder contains a unit tests file called `main_test.py`. The unit tests are using the pytest framework. Install [Pytest](https://pypi.org/project/pytest/) by running:
```bash
pip install pytest
```

To run the unit tests for the cloud function, run:
```bash
pytest
```

### Upload Script Tests
The `scheduled-upload` folder contains a unit tests file called `upload-script.Tests.ps1`. Make sure that you have [Pester](https://pester.dev/docs/quick-start) installed. If not, you can install it by running:
```powershell
Install-Module -Name Pester -Force -SkipPublisherCheck
```

To run the unit tests, either execute the script or run the `Invoke-Pester` command in the folder where the `upload-script.Tests.ps1` file exists.

---

## References
- [Eventarc triggers](https://cloud.google.com/functions/docs/calling/eventarc)
- [Deploy a Cloud Function](https://cloud.google.com/functions/docs/deploy)
- [Import data from a BAK file to Cloud SQL for SQL Server](https://cloud.google.com/sql/docs/sqlserver/import-export/import-export-bak#import_data_from_a_bak_file_to)
- [Recovery and the transaction log](https://learn.microsoft.com/en-us/sql/relational-databases/backup-restore/restore-and-recovery-overview-sql-server?view=sql-server-ver16#TlogAndRecovery)

