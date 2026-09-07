# Stale Jobs, Scalable Deletion

A Databricks notebook that bulk-deletes jobs by Job ID using the workspace **Jobs REST API 2.1** (not the Jobs UI). It is built to retire a large stale-job list (on the order of 1000 IDs) with a dry-run, backups, and a thread pool.

Committed as `Jobs_Deletion_API_Calls.ipynb`.



## Safety model

1. `confirm_workspace` widget is required. Empty or mismatched values stop the notebook. Accepts a hostname prefix (`your-workspace`), full hostname (`your-workspace.cloud.databricks.com`), or URL (`https://your-workspace.cloud.databricks.com`).
2. `DRY_RUN` lives in its own cell and defaults to `True`. Nothing is deleted until you set it to `False` and re-run that cell.
3. `create_test_jobs` widget defaults to `no`. The test cell creates 25 disposable jobs only when it is set to `yes`, so a batch of stale jobs is never created by accident.
4. Every ID is validated with `GET /api/2.1/jobs/get` before deletion.
5. Full job definitions are backed up as JSONL to a Unity Catalog Volume before any delete.
6. A thread pool plus a client-side rate limit (9 requests per second) and 429/5xx backoff keep the run under the API limit.

## REST APIs used

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/api/2.1/jobs/get?job_id=` | Existence check and backup payload |
| POST | `/api/2.1/jobs/delete` | Delete one job |
| POST | `/api/2.1/jobs/create` | Test jobs only, opt-in |


Auth uses the notebook session token (`ctx.apiToken()`). The host is read from the attached workspace, not hardcoded.

## Compute and backup volume

Attach a Unity Catalog cluster (or serverless). Backups go to the volume named in the `uc_volume` widget.

The `uc_volume` widget accepts either format:

```text
catalog.schema.volume
/Volumes/catalog/schema/volume
```

Files are written under a `job_deletion_backup/` folder inside the volume. The runner needs `USE CATALOG`, `USE SCHEMA`, `READ VOLUME`, and `WRITE VOLUME`.

## How to run

1. Open the notebook (or import the `.ipynb`) and attach a Unity Catalog cluster.
2. Set `confirm_workspace` to this workspace.
3. Set `uc_volume` to your backup volume, or keep the default test volume.
4. Leave `DRY_RUN = True`.
5. Test workspace only: set `create_test_jobs` to `yes` and run the test cell to create 25 disposable jobs. Otherwise skip it.
6. Customer run: skip the test cell and load real IDs in the customer cell (inline list or a CSV on the volume).
7. Run validate/backup, then run the delete cell. Confirm the printed count.
8. To delete for real, set `DRY_RUN = False`, re-run that cell, then re-run the delete cell. The output prints `DELETE COMPLETE` instead of `DRY-RUN, nothing deleted`.

Common gotcha: editing `DRY_RUN = False` without re-running its cell leaves the old value in the kernel, so deletes stay in dry-run.


