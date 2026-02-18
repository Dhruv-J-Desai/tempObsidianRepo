Perfect — **Option A (JSON logs)** is the cleanest because Datadog will auto-parse JSON and you’ll get correct `status/level`.

Here’s a **full notebook-ready code** that:

- Works on **Job compute** and **All-purpose**
    
- Chooses path automatically:
    
    - **Job cluster** → `/databricks/driver/logs/...`
        
    - **All-purpose** → `/app_logs/<team>/...` (because `/databricks/driver/logs` can be restricted on some standard clusters)
        
- Uses a **unique filename per run** (timestamp + pid + run/job ids) to avoid the “2nd run permission denied” problem
    
- Writes JSON to **both stdout and file**
    
- Includes `level`, `logger`, `msg`, plus useful context fields
    

```python
import os
import sys
import json
import logging
from datetime import datetime

# ---------- 1) Cluster detection ----------

def detect_cluster_context():
    """
    Detect whether we're on a job cluster vs all-purpose and gather useful IDs.
    """
    # Fast hint (exists in many job compute environments)
    is_job_cluster = os.environ.get("DB_IS_JOB_CLUSTER", "").upper() == "TRUE"

    # Optional: if spark is available, try a more reliable tag
    cluster_type_tag = None
    try:
        cluster_type_tag = spark.conf.get("spark.databricks.clusterUsageTags.clusterType", None)
        if cluster_type_tag and str(cluster_type_tag).upper() == "JOB":
            is_job_cluster = True
    except Exception:
        pass

    return {
        "is_job_cluster": is_job_cluster,
        "cluster_type_tag": cluster_type_tag,
        "cluster_id": os.environ.get("DATABRICKS_CLUSTER_ID"),
        "job_id": os.environ.get("DATABRICKS_JOB_ID"),
        "run_id": os.environ.get("DATABRICKS_RUN_ID"),
        "workspace_url": os.environ.get("DATABRICKS_HOST"),
    }

ctx = detect_cluster_context()

# ---------- 2) Path selection + unique file name ----------

def build_log_path(
    base_name: str,
    context: dict,
    team: str = "tdvip",          # change to "vpda", "rams", etc.
    base_all_purpose: str = "/app_logs",
    base_job: str = "/databricks/driver/logs"
) -> str:
    """
    Creates a unique logfile path per run to avoid permission issues when reusing same file.
    - Job clusters: /databricks/driver/logs/<file>.log
    - All-purpose:  /app_logs/<team>/<file>.log
    """
    # High-resolution timestamp + pid makes collisions extremely unlikely
    timestamp = datetime.utcnow().strftime("%Y%m%d_%H%M%S_%f")  # Unique suffix for each run/process
    pid = os.getpid()

    # Add job/run ids if present (nice for traceability)
    job_id = context.get("job_id") or "nojob"
    run_id = context.get("run_id") or "norun"

    # Ensure base_name doesn't end with .log (we'll add it)
    base_name = base_name[:-4] if base_name.lower().endswith(".log") else base_name

    unique_file = f"{base_name}_{timestamp}_pid{pid}_job{job_id}_run{run_id}.log"

    if context.get("is_job_cluster"):
        full_path = f"{base_job}/{unique_file}"
        # Usually exists, but safe anyway
        os.makedirs(os.path.dirname(full_path), exist_ok=True)
        return full_path
    else:
        # All-purpose: write under /app_logs/<team>/...
        team_dir = f"{base_all_purpose}/{team}"
        os.makedirs(team_dir, exist_ok=True)
        return f"{team_dir}/{unique_file}"

log_path = build_log_path(
    base_name="tdvip_app",  # change per app if you want
    context=ctx,
    team="tdvip"            # change per team
)

# ---------- 3) JSON Formatter (Datadog-friendly) ----------

class DDJsonFormatter(logging.Formatter):
    """
    Emits one JSON object per log line. Datadog parses this well.
    """
    def __init__(self, context: dict):
        super().__init__()
        self.context = context or {}

    def format(self, record: logging.LogRecord) -> str:
        payload = {
            "ts": datetime.utcnow().isoformat(timespec="milliseconds") + "Z",
            "level": record.levelname,            # Datadog will use this once parsed
            "logger": record.name,
            "msg": record.getMessage(),
            "pid": record.process,
            "thread": record.threadName,
            # Extra context helps filtering in Datadog:
            "is_job_cluster": self.context.get("is_job_cluster"),
            "cluster_type_tag": self.context.get("cluster_type_tag"),
            "cluster_id": self.context.get("cluster_id"),
            "job_id": self.context.get("job_id"),
            "run_id": self.context.get("run_id"),
        }

        # If exception info exists, include it
        if record.exc_info:
            payload["exc_info"] = self.formatException(record.exc_info)

        return json.dumps(payload, ensure_ascii=False)

# ---------- 4) Logger setup (stdout + file) ----------

def setup_tdvip_logger(context: dict, file_path: str) -> logging.Logger:
    logger = logging.getLogger("TDVIP_APP")
    logger.setLevel(logging.INFO)
    logger.handlers = []
    logger.propagate = False

    formatter = DDJsonFormatter(context)

    # Stdout handler (Databricks driver output)
    stdout_handler = logging.StreamHandler(sys.stdout)
    stdout_handler.setFormatter(formatter)
    logger.addHandler(stdout_handler)

    # File handler (picked path depends on cluster type)
    try:
        file_handler = logging.FileHandler(file_path)
        file_handler.setFormatter(formatter)
        logger.addHandler(file_handler)

        logger.info(f"File logging enabled at {file_path}")
    except Exception as e:
        logger.warning(f"Could not enable file logging at {file_path}: {e}")

    logger.info(f"Cluster context: {context}")
    return logger

log = setup_tdvip_logger(ctx, log_path)

# ---------- 5) Test logs ----------
log.info("TDVIP logging initialized.")
log.warning("Testing WARNING level")
log.error("Testing ERROR level")
```

### Tiny note (important)

- This code creates a **new file each run**, so you avoid the “first run OK, second run permission denied” issue on all-purpose (caused by ownership/permissions when reusing the same file).
    

If you want, I can also give you a **Teams message to Eilam** summarizing: why levels showed as INFO, and how JSON fixes it.