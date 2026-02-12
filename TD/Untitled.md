Perfect 👍 — let’s **lock this down cleanly for STANDARD (all-purpose) compute only**, using `/app_logs` since your environment already supports it via the existing init script.

We will define:

- ✅ Where logs are written
    
- ✅ How Python writes them
    
- ✅ How Datadog tails them
    
- ✅ What you search in Datadog
    

No log4j. Only Python logging.

---

# ✅ 1️⃣ Confirm Standard Cluster Init Script Supports `/app_logs`

From your screenshot, the standard cluster init script contains:

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

That means:

👉 Datadog agent is configured to tail:

```
/app_logs
```

So we are safe.

---

# ✅ 2️⃣ Standard Compute Logging Pattern (Python Only)

Use this in notebooks on standard compute:

```python
import logging
import os
from logging.handlers import RotatingFileHandler

LOG_DIR = "/app_logs"
LOG_FILE = f"{LOG_DIR}/tdvip_app.log"

# Ensure directory exists
os.makedirs(LOG_DIR, exist_ok=True)

logger = logging.getLogger("TDVIP_APP")
logger.setLevel(logging.INFO)

# Prevent duplicate handlers
if not logger.handlers:
    handler = RotatingFileHandler(
        LOG_FILE,
        maxBytes=10_000_000,
        backupCount=3
    )

    formatter = logging.Formatter(
        "%(asctime)s | %(levelname)s | %(name)s | %(message)s"
    )

    handler.setFormatter(formatter)
    logger.addHandler(handler)

# Also print to stdout (good practice)
stream_handler = logging.StreamHandler()
stream_handler.setFormatter(formatter)
logger.addHandler(stream_handler)

logger.propagate = False

logger.info("Standard cluster logging initialized")
```

---

# ✅ 3️⃣ What Happens Internally

### Flow:

```
Python logger
        ↓
/app_logs/tdvip_app.log
        ↓
Datadog agent (file tail)
        ↓
Datadog Logs Explorer
```

---

# ✅ 4️⃣ Verify on Standard Cluster

Run:

```python
logger.error("STANDARD_CLUSTER_TEST_123")
```

Then check:

```bash
%sh
ls -al /app_logs
cat /app_logs/tdvip_app.log
```

You should see your log line.

---

# ✅ 5️⃣ Verify Datadog Agent Is Tailing It

Run:

```bash
%sh
cat /etc/datadog-agent/conf.d/spark.d/spark.yaml
```

Confirm you see:

```
path: /app_logs/*.log
```

Then check agent activity:

```bash
cat /var/log/datadog/agent.log | grep app_logs
```

You should see file tailer activity.

---

# ✅ 6️⃣ What To Search in Datadog

Go to:

**Logs → Live Tail**

Search:

```
STANDARD_CLUSTER_TEST_123
```

Or:

```
source:spark
```

Or:

```
service:databricks
```

---

# ✅ 7️⃣ Why This Works on Standard

Because your standard cluster init script explicitly added:

```
/app_logs/*.log
```

The job cluster script did NOT.

That’s the difference.

---

# 🔥 Final Clean Architecture for Standard Compute

|Layer|Value|
|---|---|
|Log location|`/app_logs/*.log`|
|Logging library|Python `logging`|
|Datadog tail config|spark.yaml|
|Cluster type|Standard (all-purpose)|
|Agent enabled|`logs_enabled: true`|

---

If you want, I can now:

- Show how to make this JSON structured for Datadog parsing
    
- Show how to unify this into a reusable TDVIP logging module
    
- Or show how to extend this to job clusters safely
    

Your standard compute setup is now fully clear and correct.