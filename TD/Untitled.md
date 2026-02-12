Yep — “formalize via `/app_logs`” is exactly the right workaround pattern if **Dedicated vs Standard access mode** (or “shared”) changes what gets captured reliably.

Here’s how to make it **work the same way on both All-purpose and Job compute**, with a clean “TDVIP standard”.

---

## Goal

✅ Every notebook/job writes **application logs** to:

```
/app_logs/<app>/<run_id>.log
```

✅ Datadog Agent tails `/app_logs/**` on **both**:

- All-purpose clusters
    
- Job clusters
    

✅ Works regardless of “Dedicated” vs “Standard” access mode (as long as init scripts are allowed)

---

## 1) Standardize the platform setup (init scripts)

### A. Create ONE “common” init script for app logs

Make a script like:

**`tdvip_app_logs_setup.sh`** (run on driver)

What it should do:

1. Create `/app_logs`
    
2. Make it writable
    
3. (Optional but recommended) create subfolders and set sticky bit
    

Example (bash):

```bash
#!/bin/bash
set -euo pipefail

# Create app log dir
mkdir -p /app_logs
chmod 1777 /app_logs   # world-writable + sticky (safer than 777)

# Optional: per-app dirs
mkdir -p /app_logs/tdvip
chmod 1777 /app_logs/tdvip
```

### B. Update BOTH cluster config init scripts to tail `/app_logs`

You already have:

- `datadog_standard_cluster_driver_node_config.sh`
    
- `datadog_job_cluster_driver_node_config.sh`
    

Make sure **both** write a `spark.yaml` that includes:

```yaml
logs:
  - type: file
    path: /app_logs/*.log
    source: spark
    service: databricks
  - type: file
    path: /app_logs/*/*.log
    source: spark
    service: databricks
  - type: file
    path: /app_logs/*/*/*.log
    source: spark
    service: databricks
```

Then enable logs + restart agent (like you already do):

```bash
sed -i 's/logs_enabled: false/logs_enabled: true/' /etc/datadog-agent/datadog.yaml
systemctl restart datadog-agent
```

**Key point:** the **job-cluster script must include `/app_logs` too**, not only `/databricks/driver/logs`.

---

## 2) Enforce it everywhere (so nobody forgets)

### Option 1 (best): Cluster Policy

Create/modify a **cluster policy** so init scripts are required.

- All-purpose policy includes:
    
    - `adb_datadog_install.sh`
        
    - `tdvip_app_logs_setup.sh`
        
    - `datadog_standard_cluster_driver_node_config.sh`
        
- Job policy includes:
    
    - `adb_datadog_install.sh`
        
    - `tdvip_app_logs_setup.sh`
        
    - `datadog_job_cluster_driver_node_config.sh`
        

This guarantees everyone gets identical logging.

### Option 2: Global init script

If your workspace allows it, make `tdvip_app_logs_setup.sh` a **global init script**, then you only maintain Datadog config per cluster type.

---

## 3) Give developers a “TDVIP logging wrapper” (Python)

### A. Minimal Python logger that writes to /app_logs

Use this in every notebook/job:

```python
import json, logging, os, time
from datetime import datetime

def tdvip_logger(app: str, run_id: str | None = None):
    run_id = run_id or os.environ.get("DB_JOB_RUN_ID") or str(int(time.time()))
    log_dir = f"/app_logs/{app}"
    os.makedirs(log_dir, exist_ok=True)

    path = f"{log_dir}/{run_id}.log"

    logger = logging.getLogger(app)
    logger.setLevel(logging.INFO)
    logger.handlers.clear()

    fh = logging.FileHandler(path)
    fh.setLevel(logging.INFO)

    class JsonFormatter(logging.Formatter):
        def format(self, record):
            payload = {
                "timestamp": datetime.utcnow().isoformat() + "Z",
                "level": record.levelname,
                "logger": record.name,
                "message": record.getMessage(),
                "app": app,
                "run_id": run_id,
                "job_id": os.environ.get("DB_JOB_ID"),
                "task_run_id": os.environ.get("DB_TASK_RUN_ID"),
                "cluster_id": os.environ.get("DB_CLUSTER_ID"),
            }
            return json.dumps(payload)

    fh.setFormatter(JsonFormatter())
    logger.addHandler(fh)
    logger.propagate = False

    logger.info(f"TDVIP logging started -> {path}")
    return logger, path

log, log_path = tdvip_logger(app="tdvip", run_id=None)
log.info("hello from tdvip app_logs")
```

Now Datadog can parse JSON cleanly.

---

## 4) Spark / log4j logs

You noticed log4j worked more reliably. That’s because Spark/Databricks already routes a lot of logs through JVM logging.

To also land those into `/app_logs`, the “formal” approach is:

- add a log4j appender (log4j2 config) pointing to `/app_logs/...`
    

In practice, easiest is:

- keep Spark driver logs going to `/databricks/driver/logs/...`
    
- use `/app_logs` for _your app logs_ (structured JSON)
    
- in Datadog, you’ll have both streams
    

---

## 5) Validation checklist (works on both compute types)

On any cluster (job or all-purpose), run:

```bash
%sh
ls -ald /app_logs
cat /etc/datadog-agent/conf.d/spark.d/spark.yaml | sed -n '1,160p'
grep -R "app_logs" -n /etc/datadog-agent/conf.d/spark.d/
```

And:

```bash
%sh
grep -i "tailer" /var/log/datadog/agent.log | grep app_logs | tail -n 20
```

If you see lines like “Starting a new tailer … /app_logs/…”, you’re done.

---

## The “Dedicated compute issue” part

This pattern avoids relying on:

- notebook cell output capture
    
- special driver log routing behavior
    
- UI output
    

It turns logging into a **file tailing contract**:

> “If you can write to `/app_logs`, Datadog can collect it.”

That’s why it works consistently.

---

If you want, paste the **job-cluster driver config script** snippet around the `logs:` section, and I’ll tell you exactly what to add/change so `/app_logs` is included there too (same as standard).