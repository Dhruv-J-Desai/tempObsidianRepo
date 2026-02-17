Yes — below is a **complete, production-ready notebook example** that:

1. Detects whether it’s **job compute** or **standard (interactive)**
    
2. Always logs to **STDOUT**
    
3. If it’s a **job cluster**, additionally writes to  
    `/databricks/driver/logs/tdvip_app.log`
    
4. If it’s **standard**, just uses STDOUT (you can extend if needed)
    
5. Falls back safely if file writing fails
    

You can paste this as a single notebook cell.

---

# ✅ Complete Notebook Logging Code (Works for Both)

```python
import os
import sys
import logging
from logging.handlers import RotatingFileHandler
from datetime import datetime

# ---------------------------------------------------
# 1️⃣ Detect cluster type
# ---------------------------------------------------

def detect_cluster_type():
    is_job_cluster = os.environ.get("DB_IS_JOB_CLUSTER", "").upper() == "TRUE"

    cluster_type_tag = None
    try:
        cluster_type_tag = spark.conf.get(
            "spark.databricks.clusterUsageTags.clusterType"
        )
        if cluster_type_tag and cluster_type_tag.upper() == "JOB":
            is_job_cluster = True
    except Exception:
        pass

    return {
        "is_job_cluster": is_job_cluster,
        "cluster_type_tag": cluster_type_tag,
        "cluster_id": os.environ.get("DATABRICKS_CLUSTER_ID"),
        "job_id": os.environ.get("DATABRICKS_JOB_ID"),
        "run_id": os.environ.get("DATABRICKS_RUN_ID"),
    }


ctx = detect_cluster_type()

# ---------------------------------------------------
# 2️⃣ Setup logger
# ---------------------------------------------------

def setup_tdvip_logger(context):
    logger = logging.getLogger("TDVIP_APP")
    logger.setLevel(logging.INFO)
    logger.handlers = []
    logger.propagate = False

    formatter = logging.Formatter(
        "%(asctime)s | %(levelname)s | %(name)s | %(message)s"
    )

    # Always log to STDOUT
    stdout_handler = logging.StreamHandler(sys.stdout)
    stdout_handler.setFormatter(formatter)
    logger.addHandler(stdout_handler)

    # If job cluster, also log to driver logs path
    if context["is_job_cluster"]:
        try:
            log_path = "/databricks/driver/logs/tdvip_app.log"

            file_handler = RotatingFileHandler(
                log_path,
                maxBytes=10_000_000,
                backupCount=3
            )
            file_handler.setFormatter(formatter)
            logger.addHandler(file_handler)

            logger.info(f"Job cluster detected. File logging enabled at {log_path}")

        except Exception as e:
            logger.warning(
                f"Could not enable file logging in driver logs directory: {e}"
            )
    else:
        logger.info("Standard (interactive) cluster detected.")

    logger.info(f"Cluster context: {context}")
    return logger


log = setup_tdvip_logger(ctx)

# ---------------------------------------------------
# 3️⃣ Example usage
# ---------------------------------------------------

log.info("TDVIP logging initialized.")
log.info("Testing INFO level")
log.warning("Testing WARNING level")
log.error("Testing ERROR level")
```

---

# 🔍 What Happens Now

### If running on **Job Compute**

- Logs appear in:
    
    - Workflows → Run → Output
        
    - Driver logs
        
- Also written to:
    
    ```
    /databricks/driver/logs/tdvip_app.log
    ```
    
- Datadog (if tailing that path) should pick it up.
    

---

### If running on **Standard Compute**

- Logs appear in notebook cell output.
    
- No dependency on `/app_logs`
    
- No init script needed.
    

---

# 🧠 Why This Is Safe

- No init script
    
- No systemctl
    
- No apt-get
    
- No permission-sensitive OS calls
    
- STDOUT always works
    
- File logging is best-effort only
    

---

# 🎯 Important Reality

If Datadog is already tailing:

```
/databricks/driver/logs/*.log
```

Then this will work for job clusters.

If Datadog is tailing something else (like `/app_logs`), you’d adjust that path accordingly.

---

If you’d like, I can also provide a version that:

- Emits structured JSON logs
    
- Includes correlation IDs
    
- Is reusable as a TDVIP logging utility module
    
- Or auto-detects and disables file logging if permissions fail silently