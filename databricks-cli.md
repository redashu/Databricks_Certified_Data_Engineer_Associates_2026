# Databricks CLI Guide

## 1. Installation (macOS)

```bash
brew tap databricks/tap
brew install databricks
```

### Verify Installation

```bash
databricks version
```

---

## 2. Authentication

```bash
databricks auth login \
  --host https://<your-workspace-url>
```

**Profile name:** `delvex-admin`

### List Authentication Profiles

```bash
databricks auth profiles
```

### Verify Logged-in User

```bash
databricks current-user me --profile delvex-admin
```

---

## 3. Create Student Users

### Example: Student 10

```bash
databricks users-v2 create \
  --user-name "student10@yourdomain.com" \
  --display-name "Student 10" \
  --profile delvex-admin
```

---

## 4. List Registered Users

```bash
databricks users-v2 list \
  --profile delvex-admin
```

---

## 5. Command Summary

| Task | CLI Command |
|------|---|
| Check version | `databricks version` |
| Login | `databricks auth login --host URL` |
| List profiles | `databricks auth profiles` |
| Verify identity | `databricks current-user me` |
| Create user | `databricks users-v2 create` |
| List users | `databricks users-v2 list` |


## 6. Creating a Directory

```bash
databricks workspace mkdirs \
  /Users/ashutoshh@delvex.io/Delvex-CLI-Demo \
  --profile delvex-admin
```

---

## 7. Listing Directory Contents

```bash
databricks workspace list \
  /Users/ashutoshh@delvex.io/ \
  --profile delvex-admin
```

**Output Example:**

| ID | Type | Language | Path |
|---|---|---|---|
| 1886362336101505 | DIRECTORY | | /Users/ashutoshh@delvex.io/.db_internal |
| 1886362336101506 | DIRECTORY | | /Users/ashutoshh@delvex.io/.conversations |
| 3064046582005586 | DIRECTORY | | /Users/ashutoshh@delvex.io/Delvex-CLI-Demo |

---

# Databricks CLI Cheatsheet

## What Can We Manage Using Databricks CLI?

| Feature | Capability | Support |
|---------|-----------|---------|
| **Identity & Access** | Users, groups, entitlements, permissions | ✅ Supported |
| **Workspace** | Folders, notebooks, files, Repos | ✅ Supported |
| **Compute** | Clusters, policies, instance pools | ✅ Supported |
| **Workflows** | Jobs, schedules, task execution | ✅ Supported |
| **Lakeflow** | Declarative pipelines and updates | ✅ Supported |
| **Unity Catalog** | Catalogs, schemas, tables, volumes, grants | ✅ Supported |
| **Databricks SQL** | SQL warehouses and queries | ✅ Supported |
| **Security** | Secrets, permissions, storage credentials | ✅ Supported |
| **ML & GenAI** | MLflow experiments, models, serving endpoints | ✅ Supported |
| **Deployment** | Databricks Asset Bundles (DABs), CI/CD | ✅ Supported |

> See [Azure Databricks CLI reference](https://docs.databricks.com/en/dev-tools/cli/) for complete command documentation.

---

## 8. Workspace Folders and Notebooks

This is where I'd start for your student environment.

### Create Folders

```bash
databricks workspace mkdirs \
  /Shared/Delvex/Training
```

### Create a Folder for a Specific Student

```bash
databricks workspace mkdirs \
  /Users/student10@yourdomain.com/Training
```

### List Folders

```bash
databricks workspace list /Shared/Delvex
```

### Create a Local Python Notebook

Create a file locally (`hello.py`):

```python
# Databricks notebook source
print("Welcome to Delvex Databricks Training")

# COMMAND ----------

spark.sql("SELECT current_timestamp()").show()
```

### Import Notebook into Databricks

```bash
databricks workspace import \
  /Shared/Delvex/Training/hello \
  --file hello.py \
  --format SOURCE \
  --language PYTHON \
  --profile delvex-admin
```

### Upload an Entire Directory

```bash
databricks workspace import-dir \
  ./notebooks \
  /Shared/Delvex/Training \
  --profile delvex-admin
```

### Export Notebooks

```bash
databricks workspace export \
  /Shared/Delvex/Training/hello \
  --file hello.py \
  --format SOURCE \
  --profile delvex-admin
```

---

## 9. Unity Catalog: Catalogs, Schemas, Tables

This is especially useful for your existing `delvex` catalog and `genai_db` schema.

### Create a Catalog

```bash
databricks catalogs create student_training \
  --comment "Delvex student training catalog" \
  --profile delvex-admin
```

### List Catalogs

```bash
databricks catalogs list \
  --profile delvex-admin
```

### Create a Schema

```bash
databricks schemas create student10_lab \
  --catalog-name student_training \
  --comment "Student 10 training schema" \
  --profile delvex-admin
```

### List Schemas

```bash
databricks schemas list student_training \
  --profile delvex-admin
```

### List Tables

```bash
databricks tables list student_training student10_lab \
  --profile delvex-admin
```

### Create a Volume

```bash
databricks volumes create student10_files \
  --catalog-name student_training \
  --schema-name student10_lab \
  --volume-type MANAGED \
  --profile delvex-admin
```

### Grant Permissions

Grant student10 access to their schema:

```bash
databricks grants update \
  student_training.student10_lab \
  --json '{
    "changes": [
      {
        "principal": "student10@yourdomain.com",
        "add": ["USE_SCHEMA", "CREATE_TABLE", "SELECT"]
      }
    ]
  }' \
  --profile delvex-admin
```

Grant `USE_CATALOG` on the catalog:

```bash
databricks grants update student_training \
  --json '{
    "changes": [
      {
        "principal": "student10@yourdomain.com",
        "add": ["USE_CATALOG"]
      }
    ]
  }' \
  --profile delvex-admin
```

> **Note:** The principal must already exist in Databricks. These grants are examples of schema-level and catalog-level access; table and volume permissions may need additional grants.

---

## 10. Jobs and Workflow Automation

You can create and execute Databricks Jobs from the CLI.

### Create a Job Using JSON

Create `job.json`:

```json
{
  "name": "delvex-student10-job",
  "tasks": [
    {
      "task_key": "python_task",
      "notebook_task": {
        "notebook_path": "/Shared/Delvex/Training/hello"
      },
      "existing_cluster_id": "YOUR_CLUSTER_ID"
    }
  ]
}
```

Create the job:

```bash
databricks jobs create \
  --json @job.json \
  --profile delvex-admin
```

### List Jobs

```bash
databricks jobs list \
  --profile delvex-admin
```

### Execute a Job

```bash
databricks jobs run-now JOB_ID \
  --profile delvex-admin
```

### Check Job Details

```bash
databricks jobs get JOB_ID \
  --profile delvex-admin
```

> You can also update, delete, schedule, and monitor jobs.

---

## 11. Lakeflow Pipelines

You can manage declarative pipelines through CLI.

```bash
databricks pipelines list-pipelines \
  --profile delvex-admin
```

### Create a Pipeline Using JSON

```json
{
  "name": "delvex-student10-pipeline",
  "catalog": "student_training",
  "target": "student10_lab",
  "libraries": [
    {
      "notebook": {
        "path": "/Shared/Delvex/Training/pipeline_notebook"
      }
    }
  ]
}
```

Save as `pipeline.json` and run:

```bash
databricks pipelines create \
  --json @pipeline.json \
  --profile delvex-admin
```

### Other Pipeline Operations

```bash
databricks pipelines list-pipelines
databricks pipelines get PIPELINE_ID
databricks pipelines start-update PIPELINE_ID
databricks pipelines stop PIPELINE_ID
```

> The pipeline JSON is a starting example; configure the appropriate pipeline source, compute, and target for your actual workload.

---

## 12. Compute: Clusters

You can manage clusters using CLI.

```bash
# List clusters
databricks clusters list --profile delvex-admin

# Get cluster details
databricks clusters get CLUSTER_ID --profile delvex-admin

# Start cluster
databricks clusters start CLUSTER_ID --profile delvex-admin

# Stop cluster
databricks clusters delete CLUSTER_ID --profile delvex-admin
```

> **Note:** `clusters delete` terminates and removes the cluster configuration; it isn't merely a stop operation.

### Create a Cluster

```bash
databricks clusters create --json @cluster.json \
  --profile delvex-admin
```

---

## 13. Groups and Permissions

Instead of assigning permissions individually to 50 students, create groups.

### Create a Group

```bash
databricks groups create \
  --display-name "Delvex-Students" \
  --profile delvex-admin
```

### List Groups

```bash
databricks groups list --profile delvex-admin
```

> You can also manage group membership and permissions through the identity and permissions APIs. For your training environment, group-based permissions are much easier to maintain.

---

## 14. Secrets Management

### Create a Secret Scope

```bash
databricks secrets create-scope \
  delvex-training \
  --profile delvex-admin
```

### List Scopes

```bash
databricks secrets list-scopes \
  --profile delvex-admin
```

> This is useful for API keys, database credentials, and application secrets. Avoid storing secrets in notebooks or Git repositories.

---

## 15. Databricks Asset Bundles (DABs)

**Very important for DevOps!**

DABs let you define your Databricks resources as configuration files and deploy them consistently.

### Initialize a Bundle

```bash
databricks bundle init
```

### What You Can Define

- Notebooks
- Jobs
- Pipelines
- Variables
- Targets (dev, test, prod)

### Deploy

```bash
databricks bundle validate

databricks bundle deploy \
  --profile delvex-admin
```

> This is particularly useful when you want to deploy the same training environment repeatedly.
>
> See [Complete Databricks CLI command reference](https://docs.databricks.com/en/dev-tools/cli/).

---

## 16. Recommended Delvex Automation Project

I would build one provisioning solution with the following structure:

### Delvex Databricks Provisioning - Project Structure

```
delvex-databricks-admin/
├── config/                   # Environment and workspace settings
├── students.csv              # Student account details
├── 01_create_users.py        # Register students
├── 02_create_groups.py       # Create groups and memberships
├── 03_workspace_setup.py     # Create student folders and notebooks
├── 04_catalog_setup.py       # Create catalogs, schemas, and volumes
├── 05_permissions.py         # Configure access and grants
├── 06_compute_setup.py       # Configure compute access
├── 07_jobs_setup.py          # Create training jobs
├── 08_verify.py              # Verify provisioning
└── provision.py              # Main orchestration script
```

### One-Command Provisioning Goal

```bash
python provision.py --batch batch01
```

---

## 17. Quick CLI Discovery Commands

You don't need to memorize everything. The CLI provides built-in help.

```bash
databricks -h
databricks workspace -h
databricks jobs -h
databricks pipelines -h
databricks catalogs -h
databricks schemas -h
databricks grants -h
databricks clusters -h
databricks groups -h
```