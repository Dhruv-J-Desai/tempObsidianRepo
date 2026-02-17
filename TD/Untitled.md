Below is a **complete Databricks notebook (Python) template** that works on **both**:

- **Job compute (job cluster)**
    
- **Standard/all-purpose (interactive) cluster**
    

It will:

- Detect cluster type automatically
    
- Always log to **STDOUT** (so you see logs in Job run output + notebook output)
    
- Also write logs to a **persistent file in DBFS** (so you can fetch/share logs later)
    
- Never requires init scripts / Datadog agent to be present
    

Copy/paste into a Databricks notebook as separate cells.

---

## Cell 1 — Detect compute type + job context

```python
import os

def get_compute_context():
    # Cluster type
    db_is_job_cluster = os.environ.get("DB_IS_JOB_CLUSTER", "").upper()
    is_job_cluster = (db_is_job_cluster == "TRUE")

    # Job run context (can be present even on all-purpose if a job runs on existing cluster)
    job_id = os.environ.get("DATABRICKS_JOB_ID") or os.environ.get("DB_JOB_ID")
    run_id = os.environ.get("DATABRICKS_RUN_ID") or os.environ.get("DB_RUN_ID")

    # Spark tag fallback (works in most Databricks environments)
    cluster_type_tag = None
    try:
        cluster_type_tag = spark.conf.get("spark.databricks.clusterUsageTags.clusterType")
    except Exception:
        cluster_type_tag = None

    if cluster_type_tag:
        # Typical values: "JOB" or "INTERACTIVE"
        if cluster_type_tag.upper() == "JOB":
            is_job_cluster = True

    return {
        "is_job_cluster": is_job_cluster,
        "db_is_job_cluster": db_is_job_cluster,
        "cluster_type_tag": cluster_type_tag,
        "job_id": job_id,
        "run_id": run_id,
        "cluster_id": os.environ.get("DATABRICKS_CLUSTER_ID"),
    }

ctx = get_compute_context()
ctx
```

---

## Cell 2 — TDVIP logger that works on both (STDOUT + DBFS file)

```python
import logging
import sys
import json
from datetime import datetime, timezone

def _safe_makedirs(path: str) -> bool:
    try:
        os.makedirs(path, exist_ok=True)
        return True
    except Exception:
        return False

class JsonFormatter(logging.Formatter):
    def __init__(self, static_fields: dict):
        super().__init__()
        self.static_fields = static_fields or {}

    def format(self, record: logging.LogRecord) -> str:
        payload = {
            "ts": datetime.now(timezone.utc).isoformat(),
            "level": record.levelname,
            "logger": record.name,
            "msg": record.getMessage(),
            # Optional: file/line info (handy while debugging)
            "file": record.pathname,
            "line": record.lineno,
        }
        payload.update(self.static_fields)
        return json.dumps(payload)

def get_tdvip_logger(
    name: str = "TDVIP_APP",
    level: int = logging.INFO,
    log_dir_dbfs: str = "dbfs:/tmp/tdvip_app_logs",
    log_file_prefix: str = "tdvip_app",
    ctx: dict | None = None,
) -> tuple[logging.Logger, str | None]:
    """
    Returns (logger, dbfs_log_file_path or None).
    - Always logs to STDOUT.
    - Also logs to a persistent file under DBFS (best effort).
    """
    ctx = ctx or {}
    logger = logging.getLogger(name)
    logger.setLevel(level)
    logger.propagate = False

    # Clear existing handlers to avoid duplicate logs on reruns
    logger.handlers = []

    static_fields = {
        "cluster_id": ctx.get("cluster_id"),
        "job_id": ctx.get("job_id"),
        "run_id": ctx.get("run_id"),
        "is_job_cluster": ctx.get("is_job_cluster"),
        "cluster_type_tag": ctx.get("cluster_type_tag"),
    }

    formatter = JsonFormatter(static_fields)

    # 1) STDOUT handler (always works; shows in job run output)
    sh = logging.StreamHandler(sys.stdout)
    sh.setLevel(level)
    sh.setFormatter(formatter)
    logger.addHandler(sh)

    # 2) DBFS file handler (persistent). Use /dbfs mount for local file write.
    # Convert dbfs:/... to local /dbfs/... path
    dbfs_file_path = None
    try:
        # Make per-run file name so parallel runs don't collide
        suffix = ctx.get("run_id") or datetime.now().strftime("%Y%m%d_%H%M%S")
        dbfs_file_path = f"{log_dir_dbfs}/{log_file_prefix}_{suffix}.log"

        local_dir = "/dbfs" + log_dir_dbfs.replace("dbfs:", "")
        local_ok = _safe_makedirs(local_dir)

        if local_ok:
            local_file = "/dbfs" + dbfs_file_path.replace("dbfs:", "")
            fh = logging.FileHandler(local_file)
            fh.setLevel(level)
            fh.setFormatter(formatter)
            logger.addHandler(fh)
        else:
            dbfs_file_path = None
            logger.warning("Could not create DBFS log directory; file logging disabled.")
    except Exception as e:
        dbfs_file_path = None
        logger.warning(f"File logging disabled due to error: {e}")

    return logger, dbfs_file_path

log, dbfs_log_path = get_tdvip_logger(ctx=ctx)
log.info("Logger initialized")
{"dbfs_log_path": dbfs_log_path, "ctx": ctx}
```

---

## Cell 3 — Your test logs (proves it works on both)

```python
log.info("TDVIP_TEST: info message")
log.warning("TDVIP_TEST: warning message")
log.error("TDVIP_TEST: error message")
```

---

## Cell 4 — Example Spark work (logs from driver)

```python
from pyspark.sql import functions as F

df = spark.range(0, 5).withColumn("x2", F.col("id") * 2)
log.info("Created sample dataframe")
display(df)
log.info("Displayed dataframe")
```

---

## Cell 5 — Show the DBFS log file contents (if enabled)

```python
if dbfs_log_path:
    print("DBFS log path:", dbfs_log_path)
    # Show last ~200 lines
    local_file = "/dbfs" + dbfs_log_path.replace("dbfs:", "")
    %sh tail -n 200 "$local_file"
else:
    print("No DBFS log file path (file logging disabled).")
```

---

## Cell 6 — Where to look depending on cluster type

```python
if ctx.get("is_job_cluster"):
    print("You are on JOB COMPUTE (job cluster).")
    print("✅ Logs to STDOUT will appear in: Workflows/Jobs → Run → Output / Logs.")
else:
    print("You are on STANDARD/INTERACTIVE compute (all-purpose).")
    print("✅ Logs to STDOUT will appear in notebook cell output.")
print("✅ DBFS log file (if enabled) can be read anytime from dbfs:/tmp/tdvip_app_logs/")
```

---

### Why this works even if Datadog agent/init scripts are broken

Because it doesn’t depend on them. You always get:

- **STDOUT logs** (Databricks captures them)
    
- **DBFS file logs** (persistent)
    

If later Datadog tailing is fixed, you can decide to tail `dbfs:/tmp/tdvip_app_logs` via `/dbfs/tmp/...` on the driver (but that’s an infra task).

If you want, I can adapt this to:

- write human-readable logs (not JSON)
    
- include trace IDs / correlation IDs
    
- include a single helper function you can import across notebooks/jobs (TDVIP “logging module”)