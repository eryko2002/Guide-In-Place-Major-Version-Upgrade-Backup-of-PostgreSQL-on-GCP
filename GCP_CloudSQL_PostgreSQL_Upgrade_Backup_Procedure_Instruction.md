# In-Place Major Version Upgrade, Backup and Rollback Procedures for PostgreSQL on GCP

## 1. Plan a major version upgrade

1. Assess current and target versions:
   
To check database instances, run the following command:

```bash
gcloud sql instances list
```
To check the database versions that you can target for an in-place upgrade on your instance, do the following:

```bash
gcloud sql instances describe INSTANCE_NAME --format=json | jq '.upgradableDatabaseVersions'
```
- Replace INSTANCE_NAME with the name of your Cloud SQL instance. Look for the upgradableDatabaseVersions section in the output, which shows the versions you can target.

2. Review Changes and Incompatibilities

## 2. Prepare for a major version upgrade 
### 1. Check and Adjust Character Set Configuration (LC_COLLATE)
#### Verify LC_COLLATE: Connect to the PostgreSQL instance and check LC_COLLATE:
```bash
SELECT datname, datcollate FROM pg_database WHERE datname IN
('template0', 'template1', 'postgres');
```

### 2. Manage PostgreSQL Extensions Before Upgrade
#### Objective: Ensure that all PostgreSQL extensions are compatible with the target version to prevent issues during the upgrade.
#### Steps:
**1**: Identify and Review Existing Extensions
 
**2**: Remove Unsupported Extensions
 
**3**: Upgrade PostGIS Extensions (if applicable)
- PostGIS is supported for Cloud SQL for PostgreSQL across all major versions. Ensure that your PostGIS extension is compatible with the target PostgreSQL version you plan to upgrade to.
- Check the specific version of PostGIS installed on your instance. For the upgrade, ensure the version is supported for the target PostgreSQL version. Below are the PostGIS extension versions for different Cloud SQL PostgreSQL versions:

| Cloud SQL for PostgreSQL Version | PostGIS Extension Version |
| --------------------------------- | ------------------------- |
| PostgreSQL 9.6                    | 3.2.5                     |
| PostgreSQL 10                     | 3.2.5                     |
| PostgreSQL 11                     | 3.2.5                     |
| PostgreSQL 12                     | 3.4.4                     |
| PostgreSQL 13                     | 3.5.2                     |
| PostgreSQL 14                     | 3.5.2                     |
| PostgreSQL 15                     | 3.5.2                     |
| PostgreSQL 16                     | 3.5.2                     |
| PostgreSQL 17                     | 3.5.2                     |

- To specify the version of PostGIS in your CREATE EXTENSION command, use the VERSION clause.
- For more information on PostGIS installation and management, refer to the <https://postgis.net/>


**4**: Remove Deprecated Database Objects
- After cleaning up deprecated database objects, run the following SQL command to confirm there are no warnings before proceeding with the upgrade:
```bash
SELECT PostGIS_full_version();
```
- If there are no warnings in the output, you can proceed with the upgrade.

**5**: Verify Permissions for Extension Management
- Only users with the cloudsqlsuperuser role can manage extensions. Ensure you are using a user with these permissions to manage extensions as needed.

### 3. Note upgrade limitations

- Before you perform an in-place major version upgrade to PostgreSQL 15 and later, upgrade the PostGIS extension for all of your databases to version 3.5.2.
- PostGIS Version – Before upgrading to PostgreSQL 15 or later, you must update the PostGIS extension to version 3.5.2.
- Before you perform an in-place major version upgrade to PostgreSQL 17, upgrade the rdkit extension for all of your databases to version 4.6.1.
- If you install the pg_ivm or pg_squeeze extensions for your instance, then you can't perform a major version upgrade. To fix this, uninstall these extensions and then perform the upgrade.
- Incompatible Flags – The flags ```vacuum_defer_cleanup_age``` and ```force_parallel_mode``` block the upgrade process. These flags must be removed prior to the upgrade. Note that in PostgreSQL 16 and later, the ```vacuum_defer_cleanup_age``` flag is deprecated, and ```force_parallel_mode``` has been renamed to ```debug_parallel_query```.

### 4. Check Database Connectivity and Compatibility
- When performing an upgrade from one major version to another, attempt to connect to each database to see if there are any compatibility issues. Ensure that your databases can connect to each other. Check the ```datallowconn``` field for each database to ensure that a connection is allowed. A t value means that it's allowed, and an f value indicates that a connection can't be established.

## 3. Upgrade the Database Major Version In-Place

### 1. Initiate the Upgrade:
- Use the gcloud Command: The upgrade process is initiated by running the gcloud sql instances patch command. Replace the placeholder values for your specific instance and the desired target database version:
```bash
gcloud sql instances patch INSTANCE_NAME --database-version=DATABASE_VERSION
```
- Instance Unavailability: The instance will become unavailable for a period while the upgrade is performed, so plan for downtime.

### 2. Monitor the Upgrade Process:
- Get Upgrade Operation Status: Use the following command to monitor the status of the upgrade operation:
```bash
gcloud sql operations list --instance=INSTANCE_NAME
```
- Track the Upgrade with gcloud: Once you have the operation name, you can track its progress using the describe command:
```bash
gcloud sql operations describe OPERATION_ID
```
The OPERATION_ID corresponds to the value in the NAME column

### 3. Waiting for the operation to complete: If the operation takes longer, it is recommended to use the following command:
```bash
gcloud beta sql operations wait --project=PROJECT_ID OPERATION_ID
```
The OPERATION_ID value is provided immediately after an exception or timeout occurs. Simply copy the recommended command and execute it.

### 4. Note on exceptions:
It is possible that you may see this exception again. Do not worry—within a few minutes, your post-upgrade backup will be visible. Verify its success later.

## 3. Automatic Pre-Upgrade and Post-Upgrade Backups

### 1. View a list of backups
Once you perform a major version upgrade, Cloud SQL automatically creates two on-demand backups as part of the upgrade process:
- Pre-Upgrade Backup: Created before the upgrade starts.
- Post-Upgrade Backup: Created immediately after the upgrade finishes and new writes are allowed.

You can list these backups using the gcloud CLI:
```bash
gcloud sql backups list --instance=INSTANCE_NAME
```

### 2. Filter by Backup Type

The backups related to the upgrade will be labeled with Pre-upgrade or Post-upgrade in their description. For example, if upgrading from PostgreSQL 14 to 15, they will be labeled as:
- Pre-upgrade backup, POSTGRES_14 to POSTGRES_15
- Post-upgrade backup, POSTGRES_14 to POSTGRES_15

You can use this information to quickly identify the backups relevant to the upgrade.

### 3. View Backup Details

To get detailed information about any of the backups (such as start and end times), use the following command with the backup ID:

```bash
gcloud sql backups describe BACKUP_ID --instance=INSTANCE_NAME
```
This will allow you to confirm the exact time the backup was taken, which can be useful for verification or recovery purposes.

## 5. Performing Database Restoration (Rollback) Procedure from Backup to the same instance
Note: This procedure applies only when restoring a backup to the same instance if the backup was taken from the same database version as the current instance. Restoring a backup from a different database version (e.g., from an older to a newer version or vice versa) is not supported.
However, if the database version has not been upgraded (i.e., remains the same as before), you can restore from a backup taken prior to the upgrade to revert to the previous state.

### 1. Describe the instance to see whether it has any replicas:
```bash
gcloud sql instances describe SOURCE_INSTANCE_NAME
```

### 2. List the backups for the instance
```bash
gcloud sql backups list --instance SOURCE_INSTANCE_NAME
```
Find the backup you want to use and record its ```ID``` value.

### 3. Check details about a specific backup for the instance
```bash
gcloud sql backups describe --instance SOURCE_INSTANCE_NAM BACKUP_ID
```

### 4. Restoring the database from a backup
```bash
gcloud sql backups restore BACKUP_ID \
--restore-instance=SOURCE_INSTANCE_NAME
```

### 5. Verification after restoration
```bash
gcloud sql operations list --instance=SOURCE_INSTANCE_NAME
```

## 6. Restoring from a pre-upgrade backup (if the upgrade failed and the database version remains unchanged)
Note: This procedure should be used only if the upgrade process failed and the database version has not been changed. 

